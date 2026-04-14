# Walkthrough — 02 Secrets Rotation

## About this service
**Secrets Manager rotation** automates credential rotation via a Lambda "rotator." AWS ships built-in rotator templates for RDS (Postgres, MySQL, Oracle, MSSQL), Redshift, and DocumentDB — you flip a switch and Secrets Manager runs the 4-step state machine (**createSecret → setSecret → testSecret → finishSecret**) on your schedule. For third-party APIs (Datadog, Stripe, etc.) you write your own rotator Lambda implementing the same 4 steps. Rotated secrets are versioned with staging labels: **AWSCURRENT** (what apps read), **AWSPENDING** (in-flight new value), **AWSPREVIOUS** (last version, briefly valid so in-flight connections don't fail).

**Why it matters:** static DB passwords in Parameter Store / env vars age like milk. Compliance frameworks (SOC 2, PCI, HIPAA) require rotation. Secrets Manager does it without downtime — apps pull `AWSCURRENT` each connection and never care.

**When to use:** any production database credential, any third-party API key that supports rotation, any service account password.
**When NOT to use:** truly static secrets (a license key that never changes). Also not worth it for hobby projects — Parameter Store free tier is fine.

## Estimated cost
- Secret storage: **$0.40/secret/month**
- API calls: **$0.05 per 10,000 API calls**
- Rotation Lambda invocations: **~$0.00** (well under free tier at 1 rotation/30 days)
- Typical 5-secret setup: **~$2/month**
- Total for this lesson: **~$2/month**

---

## Step 1: RDS with Secrets Manager-generated password
> **Why:** The most common rotation pattern. RDS + Secrets Manager is a first-class integration — CDK sets up the secret, attaches it to the DB, wires the rotator Lambda, and puts it in the DB's VPC. Zero custom code.

`lib/rotation-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as sm from 'aws-cdk-lib/aws-secretsmanager';
import { Construct } from 'constructs';

export class RotationStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

    const db = new rds.DatabaseInstance(this, 'Pg', {
      engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_15 }),
      vpc,
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.MICRO),
      credentials: rds.Credentials.fromGeneratedSecret('dbadmin'),
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      deletionProtection: false,
    });

    // 30-day automatic rotation using built-in template
    db.secret!.addRotationSchedule('Rotation', {
      hostedRotation: sm.HostedRotation.postgreSqlSingleUser({
        vpc,
        vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      }),
      automaticallyAfter: cdk.Duration.days(30),
    });

    new cdk.CfnOutput(this, 'SecretArn', { value: db.secret!.secretArn });
  }
}
```

Deploy:
```bash
cdk deploy
```

## Step 2: Force rotation & inspect versions
> **Why:** Trigger rotation manually to watch the state machine run. You'll see `AWSCURRENT` move to the new version while `AWSPREVIOUS` holds briefly — that's the zero-downtime magic.

```bash
SECRET_ARN=$(aws cloudformation describe-stacks --stack-name RotationStack \
  --query "Stacks[0].Outputs[?OutputKey=='SecretArn'].OutputValue" --output text)

aws secretsmanager rotate-secret --secret-id $SECRET_ARN

aws secretsmanager describe-secret --secret-id $SECRET_ARN \
  --query "VersionIdsToStages"
```

Expected output:
```json
{
  "a1b2c3...": ["AWSPREVIOUS"],
  "d4e5f6...": ["AWSCURRENT"]
}
```

Fetch the new password:
```bash
aws secretsmanager get-secret-value --secret-id $SECRET_ARN \
  --query SecretString --output text | jq .
```

## Step 3: Custom rotator for a third-party API key
> **Why:** AWS only ships rotators for databases. For Datadog / Stripe / GitHub tokens you write your own — but the 4-step contract is identical so the harness handles scheduling, versioning, retries.

`lambda/rotator.py`:
```python
import boto3
import os
import urllib3

sm = boto3.client('secretsmanager')
http = urllib3.PoolManager()

def handler(event, context):
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    if step == 'createSecret':
        # Generate a new API key (call third-party API)
        current = sm.get_secret_value(SecretId=arn, VersionStage='AWSCURRENT')
        try:
            sm.get_secret_value(SecretId=arn, VersionId=token, VersionStage='AWSPENDING')
            return  # already created
        except sm.exceptions.ResourceNotFoundException:
            pass
        new_key = create_new_third_party_key(current['SecretString'])
        sm.put_secret_value(
            SecretId=arn, ClientRequestToken=token,
            SecretString=new_key, VersionStages=['AWSPENDING'])

    elif step == 'setSecret':
        pass  # third-party already activated in createSecret

    elif step == 'testSecret':
        pending = sm.get_secret_value(SecretId=arn, VersionId=token, VersionStage='AWSPENDING')
        r = http.request('GET', 'https://api.example.com/me',
                         headers={'Authorization': f'Bearer {pending["SecretString"]}'})
        assert r.status == 200

    elif step == 'finishSecret':
        meta = sm.describe_secret(SecretId=arn)
        current_version = [v for v, stages in meta['VersionIdsToStages'].items()
                           if 'AWSCURRENT' in stages][0]
        sm.update_secret_version_stage(
            SecretId=arn, VersionStage='AWSCURRENT',
            MoveToVersionId=token, RemoveFromVersionId=current_version)

def create_new_third_party_key(old):
    # Call third-party rotation endpoint; return new key
    return 'new-key-value'
```

CDK:
```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';

const rotator = new lambda.Function(this, 'ApiKeyRotator', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'rotator.handler',
  code: lambda.Code.fromAsset('lambda'),
  timeout: cdk.Duration.seconds(30),
});

const apiSecret = new sm.Secret(this, 'ThirdPartyApiKey', {
  secretName: 'third-party/api-key',
});
apiSecret.grantRead(rotator);
apiSecret.grantWrite(rotator);
rotator.addPermission('AllowSm', {
  principal: new iam.ServicePrincipal('secretsmanager.amazonaws.com'),
});

new sm.RotationSchedule(this, 'ApiRotation', {
  secret: apiSecret,
  rotationLambda: rotator,
  automaticallyAfter: cdk.Duration.days(90),
});
```

## Step 4: Staging labels and version retrieval
> **Why:** Apps always read `AWSCURRENT` (default). Canaries/health checks can read `AWSPENDING` during rotation. `AWSPREVIOUS` lets long-lived connections finish gracefully. Understanding this = debugging rotation failures.

```bash
# Explicit stage retrieval
aws secretsmanager get-secret-value --secret-id $SECRET_ARN --version-stage AWSCURRENT
aws secretsmanager get-secret-value --secret-id $SECRET_ARN --version-stage AWSPREVIOUS
```

## Step 5: Share secret cross-account via resource policy
> **Why:** Multi-account setups often have a "shared-services" account holding the secret. Other accounts read via resource policy — no replication, one source of truth.

```bash
aws secretsmanager put-resource-policy --secret-id $SECRET_ARN \
  --resource-policy '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"AWS":"arn:aws:iam::222222222222:root"},
      "Action":"secretsmanager:GetSecretValue",
      "Resource":"*"
    }]
  }'
```

From account 222222222222 (must also be granted via identity policy):
```bash
aws secretsmanager get-secret-value --secret-id arn:aws:secretsmanager:us-east-1:111111111111:secret:...
```

## Cleanup
```bash
cdk destroy
# Secrets deleted with 7-day recovery window by default. Force-delete:
aws secretsmanager delete-secret --secret-id $SECRET_ARN --force-delete-without-recovery
```

## Common Errors
- **Rotation Lambda times out** → Lambda is in wrong subnet; needs path to RDS + VPC endpoint for Secrets Manager (or NAT).
- **`InvalidRequestException: Rotation not enabled`** → secret has no `RotationLambdaARN` attached.
- **App fails for 30 seconds after rotation** → app caches the secret; re-fetch on auth failure or use the caching SDK which refreshes on error.
- **`testSecret` fails** → rotator didn't activate the pending creds with the provider yet. Check your `setSecret` logic.
- **Old password "still works" hours later** → `AWSPREVIOUS` intentionally stays valid until the next rotation. Not a bug.
