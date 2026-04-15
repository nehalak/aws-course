# Walkthrough — Capstone 01 SaaS Multi-Tenant

## About this capstone
You are building the backend for a B2B SaaS product where each customer (a "tenant") must be isolated from every other tenant — data, encryption keys, and usage metering. This capstone is the synthesis of everything you have learned about Cognito, API Gateway, Lambda, DynamoDB, KMS, IAM, Step Functions, and CloudWatch. The central tension is **pool vs silo**: sharing resources across tenants is cheap but risky, dedicating resources per tenant is secure but expensive. You will implement both strategies in the same stack and tier them by customer plan.

**Why it matters:** Every real SaaS company — Stripe, Slack, Notion, Atlassian — faces these exact problems. If a bug leaks tenant A's data to tenant B you lose the business. Regulated industries (healthcare, finance) require per-tenant encryption keys so one customer's data can be cryptoshredded without touching another's. Usage metering drives the billing system, which must attribute every CPU-second and byte to a tenant ID.

**Prerequisites:**
- `intermediate/cognito` — user pools, custom claims, JWT authorizers.
- `intermediate/dynamodb` — single-table design, condition expressions.
- `intermediate/kms` — customer-managed keys, grants.
- `intermediate/step-functions` — standard workflows.
- `advanced/iam` — condition keys, `dynamodb:LeadingKeys`, session tags.

## Estimated cost
- Cognito User Pool: first 50k MAU free, then $0.0055/MAU.
- API Gateway (REST): $3.50 per million requests.
- Lambda: $0.20 per million + $0.0000166667/GB-s.
- DynamoDB on-demand: $1.25 per million writes, $0.25 per million reads.
- KMS: **$1/key/month** — with 10 silo tenants that's $10/mo just for keys.
- Step Functions Standard: $0.025 per 1,000 state transitions.
- CloudWatch custom metrics: $0.30 per metric per month — budget for ~$5 with per-tenant dimensions.
- S3: $0.023/GB.
- Total for this capstone: **~$35–60/month** idle with a few tenants, climbing fast with silo KMS keys and CloudWatch custom metrics. **WARN:** per-tenant CloudWatch metrics can explode your bill — use EMF and cap the tenant cardinality during testing. **Destroy after each session.**

---

## Architecture

```
          +----------------+
  User -> |   Cognito UP   | custom:tenantId, custom:tier
          +-------+--------+
                  | JWT
                  v
          +----------------+     +--------------------+
          |  API Gateway   | --> | Lambda (authorizer |
          +-------+--------+     |  + handlers)       |
                  |              +----+------+--------+
                  |                   |      |
                  |          reads    |      |  writes
                  |                   v      v
                  |              +------------------+
                  |              | DynamoDB single  |
                  |              | table (PK=T#id)  |
                  |              +------------------+
                  |                   |
                  |              per-tenant KMS (silo tier)
                  |              shared KMS     (pool tier)
                  v
          +----------------+
          | Step Functions | onboarding workflow
          +----------------+
```

Two tiers:
- **Pool (free)** — shared DynamoDB table, shared KMS key, row-level isolation via `dynamodb:LeadingKeys`.
- **Silo (premium)** — same table but encrypted with a tenant-specific KMS key; S3 per-tenant prefix with dedicated access point; dedicated IAM role assumed via session tags.

## Step 1: CDK project layout
> **Why:** A capstone with onboarding, runtime, and observability concerns deserves multi-stack organization so you can deploy pieces independently (you will iterate on onboarding 20 times before touching the data plane).

```
saas-capstone/
├── bin/app.ts
├── lib/
│   ├── identity-stack.ts       # Cognito + tenant attributes
│   ├── data-stack.ts           # DynamoDB, KMS pool key
│   ├── api-stack.ts            # API GW + Lambda handlers
│   ├── onboarding-stack.ts     # Step Functions
│   └── observability-stack.ts  # Dashboards, alarms
├── src/
│   ├── authorizer/index.ts
│   ├── handlers/{create,read,list}.ts
│   ├── onboarding/{create-kms,seed-data}.ts
│   └── shared/tenant-context.ts
└── cdk.json
```

```bash
npm init -y
npm i -D aws-cdk-lib constructs typescript esbuild @types/aws-lambda
npm i @aws-sdk/client-dynamodb @aws-sdk/client-kms @aws-sdk/util-dynamodb
npx cdk init app --language typescript
```

## Step 2: Identity stack — Cognito with tenant claims
> **Why:** The tenant ID must come from a signed JWT, never from the request body. Attack vector #1 in multi-tenant systems is a client passing `tenantId=victim` in a header; an attribute mapped from Cognito is tamper-proof because Cognito signs the token.

```typescript
// lib/identity-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import { Construct } from 'constructs';

export class IdentityStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;
  public readonly client: cognito.UserPoolClient;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.userPool = new cognito.UserPool(this, 'UserPool', {
      userPoolName: 'saas-capstone',
      selfSignUpEnabled: false, // tenants onboarded via admin flow
      standardAttributes: { email: { required: true, mutable: false } },
      customAttributes: {
        tenantId: new cognito.StringAttribute({ mutable: false }),
        tier:     new cognito.StringAttribute({ mutable: true }),
      },
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    this.client = this.userPool.addClient('app', {
      authFlows: { userPassword: true, adminUserPassword: true },
      readAttributes: new cognito.ClientAttributes()
        .withStandardAttributes({ email: true })
        .withCustomAttributes('tenantId', 'tier'),
    });

    new cdk.CfnOutput(this, 'UserPoolId',   { value: this.userPool.userPoolId });
    new cdk.CfnOutput(this, 'UserPoolClient', { value: this.client.userPoolClientId });
  }
}
```

## Step 3: Data stack — DynamoDB + pool KMS key
> **Why:** A single DynamoDB table with `tenantId` as the partition-key prefix is the canonical multi-tenant data layout: cheap, hot partition risk is low if tenant IDs are ULIDs, and the `dynamodb:LeadingKeys` IAM condition lets you prove isolation in an audit.

```typescript
// lib/data-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ddb from 'aws-cdk-lib/aws-dynamodb';
import * as kms from 'aws-cdk-lib/aws-kms';
import { Construct } from 'constructs';

export class DataStack extends cdk.Stack {
  public readonly table: ddb.Table;
  public readonly poolKey: kms.Key;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.poolKey = new kms.Key(this, 'PoolKey', {
      alias: 'saas/pool',
      enableKeyRotation: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    this.table = new ddb.Table(this, 'Table', {
      tableName: 'saas-capstone',
      partitionKey: { name: 'pk', type: ddb.AttributeType.STRING },
      sortKey:      { name: 'sk', type: ddb.AttributeType.STRING },
      billingMode: ddb.BillingMode.PAY_PER_REQUEST,
      encryption: ddb.TableEncryption.CUSTOMER_MANAGED,
      encryptionKey: this.poolKey,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });
  }
}
```

Key format: `pk = T#<tenantId>`, `sk = <entity>#<id>` — e.g. `pk=T#01HY..., sk=PROJECT#01HZ...`.

## Step 4: Authorizer and tenant context
> **Why:** Every handler needs tenant context. Extract it once in a shared helper that **always** reads from `event.requestContext.authorizer.claims` (signed) — never from body/query/header.

```typescript
// src/shared/tenant-context.ts
export interface TenantCtx { tenantId: string; tier: 'pool' | 'silo'; sub: string; }

export function extract(event: any): TenantCtx {
  const c = event.requestContext?.authorizer?.jwt?.claims
         ?? event.requestContext?.authorizer?.claims;
  if (!c?.['custom:tenantId']) throw new Error('missing tenant claim');
  return {
    tenantId: c['custom:tenantId'],
    tier: (c['custom:tier'] as any) ?? 'pool',
    sub: c.sub,
  };
}
```

## Step 5: API stack with scoped IAM
> **Why:** The Lambda execution role uses a **`dynamodb:LeadingKeys`** condition tied to `aws:PrincipalTag/tenantId`. Session tags are set by the authorizer via `sts:TagSession`, so even if your handler has a bug it cannot read another tenant's rows.

```typescript
// lib/api-stack.ts (excerpt)
import * as apigwv2 from 'aws-cdk-lib/aws-apigatewayv2';
import { HttpJwtAuthorizer } from 'aws-cdk-lib/aws-apigatewayv2-authorizers';
import { HttpLambdaIntegration } from 'aws-cdk-lib/aws-apigatewayv2-integrations';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import * as iam from 'aws-cdk-lib/aws-iam';

// inside constructor:
const handlerRole = new iam.Role(this, 'HandlerRole', {
  assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
});

handlerRole.addToPolicy(new iam.PolicyStatement({
  actions: ['dynamodb:GetItem', 'dynamodb:PutItem', 'dynamodb:Query'],
  resources: [table.tableArn],
  conditions: {
    'ForAllValues:StringLike': {
      'dynamodb:LeadingKeys': ['T#${aws:PrincipalTag/tenantId}*'],
    },
  },
}));

const api = new apigwv2.HttpApi(this, 'Api');
const auth = new HttpJwtAuthorizer('Cognito', userPool.userPoolProviderUrl, {
  jwtAudience: [clientId],
});

const createFn = new NodejsFunction(this, 'CreateFn', {
  entry: 'src/handlers/create.ts',
  role: handlerRole,
  environment: { TABLE: table.tableName },
});

api.addRoutes({
  path: '/projects', methods: [apigwv2.HttpMethod.POST],
  integration: new HttpLambdaIntegration('Int', createFn),
  authorizer: auth,
});
```

## Step 6: Onboarding Step Function
> **Why:** Onboarding a silo tenant touches 5+ services (create KMS key, create grants, seed table rows, create Cognito group, send welcome email). If step 4 fails you need compensation for steps 1-3. Step Functions with `Retry` + `Catch` gives you that for free.

```typescript
// lib/onboarding-stack.ts (excerpt)
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';

const createKms = new tasks.LambdaInvoke(this, 'CreateKmsKey', {
  lambdaFunction: createKmsFn,
  resultPath: '$.kms',
});

const seedData = new tasks.LambdaInvoke(this, 'SeedData', {
  lambdaFunction: seedFn, resultPath: '$.seed',
});

const createCognitoGroup = new tasks.CallAwsService(this, 'CreateGroup', {
  service: 'cognitoidentityprovider',
  action: 'createGroup',
  parameters: {
    UserPoolId: userPool.userPoolId,
    GroupName: sfn.JsonPath.format('tenant-{}', sfn.JsonPath.stringAt('$.tenantId')),
  },
  iamResources: [userPool.userPoolArn],
});

const failed = new sfn.Fail(this, 'Failed', { cause: 'Onboarding failed' });

const chain = createKms
  .addCatch(failed, { resultPath: '$.error' })
  .next(seedData.addCatch(failed, { resultPath: '$.error' }))
  .next(createCognitoGroup);

new sfn.StateMachine(this, 'Onboarding', { definition: chain });
```

## Step 7: Per-tenant KMS for silo tier
> **Why:** Silo customers pay for the privilege of having their data encrypted under a key that only they can touch. This enables **cryptoshred** — deleting the key makes the data unrecoverable, which is sometimes a regulatory requirement (GDPR right to erasure, schrems-II, etc.).

```typescript
// src/onboarding/create-kms.ts
import { KMSClient, CreateKeyCommand, CreateAliasCommand, CreateGrantCommand } from '@aws-sdk/client-kms';
const kms = new KMSClient({});

export const handler = async (e: { tenantId: string }) => {
  const key = await kms.send(new CreateKeyCommand({
    Description: `tenant ${e.tenantId}`,
    Tags: [{ TagKey: 'tenantId', TagValue: e.tenantId }],
  }));
  await kms.send(new CreateAliasCommand({
    AliasName: `alias/tenant-${e.tenantId}`,
    TargetKeyId: key.KeyMetadata!.KeyId,
  }));
  await kms.send(new CreateGrantCommand({
    KeyId: key.KeyMetadata!.KeyId,
    GranteePrincipal: process.env.HANDLER_ROLE_ARN!,
    Operations: ['Encrypt', 'Decrypt', 'GenerateDataKey'],
  }));
  return { keyId: key.KeyMetadata!.KeyId };
};
```

## Step 8: Usage metering via EMF
> **Why:** Embedded Metric Format is the right pattern for per-tenant metrics: you write structured JSON to stdout, CloudWatch extracts the metrics asynchronously, you pay for the log lines not per PutMetricData call. Cardinality still costs so cap tenant count on the dimension.

```typescript
// inside handler after successful request
console.log(JSON.stringify({
  _aws: {
    Timestamp: Date.now(),
    CloudWatchMetrics: [{
      Namespace: 'Saas/Usage',
      Dimensions: [['TenantId', 'Tier']],
      Metrics: [{ Name: 'Requests', Unit: 'Count' }, { Name: 'BytesWritten', Unit: 'Bytes' }],
    }],
  },
  TenantId: ctx.tenantId,
  Tier: ctx.tier,
  Requests: 1,
  BytesWritten: payloadBytes,
}));
```

## Step 9: Deploy
> **Why:** Deploy identity and data first — the API stack references their exports.

```bash
npx cdk deploy IdentityStack DataStack
npx cdk deploy ApiStack OnboardingStack ObservabilityStack
```

Expected output: API URL, user-pool ID, onboarding state-machine ARN.

## Step 10: Isolation test
> **Why:** The whole capstone exists to prove isolation. If this test does not return 403 you have shipped a data breach.

```bash
# create 2 tenants via the onboarding state machine
aws stepfunctions start-execution --state-machine-arn $SM \
  --input '{"tenantId":"tenantA","tier":"silo"}'
aws stepfunctions start-execution --state-machine-arn $SM \
  --input '{"tenantId":"tenantB","tier":"pool"}'

# get tenant-A JWT
TOKEN_A=$(aws cognito-idp admin-initiate-auth ... | jq -r .AuthenticationResult.IdToken)

# tenant A creates a project
curl -H "Authorization: Bearer $TOKEN_A" -d '{"name":"A-secret"}' $API/projects

# tenant B (with their own token) tries to read it by guessing the id
curl -H "Authorization: Bearer $TOKEN_B" $API/projects/<A-project-id>
# expect: 403 "AccessDenied by dynamodb:LeadingKeys"
```

## Step 11: Load test and cost attribution
> **Why:** A multi-tenant bill that says "DynamoDB = $412" is useless; finance needs to know that tenant A owes $180 and tenant B owes $232. CloudWatch + Cost Explorer tags answer this.

Create `load.yml` alongside your project (set `TOKEN_A/B/C` in your shell to valid tenant JWTs first):

```yaml
# load.yml
config:
  target: "{{ $processEnvironment.API }}"   # export API=https://...execute-api...
  phases:
    - duration: 600   # 10 minutes
      arrivalRate: 100
  variables:
    token:
      - "{{ $processEnvironment.TOKEN_A }}"
      - "{{ $processEnvironment.TOKEN_B }}"
      - "{{ $processEnvironment.TOKEN_C }}"
  defaults:
    headers:
      authorization: "Bearer {{ token }}"
scenarios:
  - name: create-and-read
    flow:
      - post:
          url: "/projects"
          json: { name: "load-{{ $randomString() }}" }
      - get:
          url: "/projects"
```

```bash
npm i -g artillery  # load-test CLI used below
# artillery with 3 tenants sending mixed traffic
artillery run load.yml  # 100 rps for 10 min, 3 tenant tokens

# check per-tenant metrics
aws cloudwatch get-metric-statistics --namespace Saas/Usage \
  --metric-name Requests --dimensions Name=TenantId,Value=tenantA \
  --start-time $(date -u -d '1 hour ago' +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) --period 60 --statistics Sum
```

## Verification / success criteria
- Tenant A JWT used against tenant B path → HTTP 403, CloudTrail shows `AccessDenied` with reason `dynamodb:LeadingKeys`.
- CloudWatch dashboard shows per-tenant request counts with no zero values for active tenants.
- Onboarding state machine: happy path completes < 30s; injected failure rolls back KMS key.
- Silo tenant's data is decryptable only when its KMS key is enabled — disable the key and reads return `KMSInvalidStateException`.
- Cost Explorer grouped by tag `tenantId` shows non-zero split.

## Cleanup
```bash
# schedule KMS keys for deletion (7-day window, cannot skip)
for k in $(aws kms list-aliases --query "Aliases[?starts_with(AliasName,'alias/tenant-')].TargetKeyId" --output text); do
  aws kms schedule-key-deletion --key-id $k --pending-window-in-days 7
done

npx cdk destroy --all
```

Remember: CloudWatch Logs retain per-tenant data — delete those log groups explicitly if tenant data must not survive.

## Common Errors
- **`AccessDeniedException: LeadingKeys`** on legitimate requests → your authorizer is not setting session tags or Cognito claim name mismatch (`custom:tenantId` vs `tenantId`).
- **`The security token included in the request is invalid`** after rotation → JWT audience mismatch between authorizer config and user-pool client ID.
- **`KMSInvalidStateException: Key is pending deletion`** → silo tenant KMS alias pointed at a scheduled-deleted key; either cancel deletion or recreate and update the alias.
- **CloudWatch bill spike** → per-tenant dimension cardinality exploded; switch to sampling or bucket tenants into tiers for metrics.
- **`ResourceNotFoundException` on DynamoDB** in onboarding → Step Function ran before DataStack export propagated; add `addDependency(dataStack)`.
- **Cognito `InvalidParameterException: Attributes did not conform`** → you tried to make `custom:tenantId` mutable; it must be immutable to be trustworthy.
- **Step Function stuck** `States.TaskFailed` with no detail → missing `resultPath` causes the Lambda error to overwrite input; always set `resultPath: '$.error'` in `Catch`.
