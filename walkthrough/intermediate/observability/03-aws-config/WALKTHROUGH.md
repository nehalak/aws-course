# Walkthrough — 03 AWS Config

## About this service
**AWS Config** continuously records the configuration of your AWS resources and evaluates them against **rules**. Every change (an S3 bucket policy edit, a security group rule, an IAM role update) becomes a **configuration item** in a history timeline. **Managed rules** are pre-written checks (`s3-bucket-public-read-prohibited`, `encrypted-volumes`). **Custom rules** run a Lambda to evaluate. **Conformance packs** bundle rules + remediation for frameworks like PCI-DSS, HIPAA, NIST. **Auto-remediation** invokes an SSM Automation document to fix non-compliant resources automatically.

**Why it matters:** "who turned off encryption on that bucket last Tuesday?" is an unanswerable question without Config. It's the audit trail and the continuous-compliance engine that CloudTrail alone can't provide.

**When to use:** regulated industries (SOC2, HIPAA, PCI, FedRAMP), any team that needs a resource history timeline, orgs enforcing tag/security baselines across many accounts.
**When NOT to use:** personal sandbox accounts (cost vs. value is terrible), ephemeral CI/CD accounts with <10 resources, regions you don't operate in. Also consider just using SCPs + preventive guardrails first.

## Estimated cost
- **Configuration items recorded: $0.003 each** — a churny EKS/Lambda account can push thousands/day
- **Rule evaluations: $0.001 each** (first 100k/month free per account)
- **Conformance pack evaluations: $0.0012 each** (first 1M/month free)
- **S3 storage for delivery channel: ~$0.023/GB/month** (negligible)
- **SSM remediation: free** (you pay for whatever the doc does)
- Typical small account: ~$5-15/month. Large multi-region org: hundreds.
- Total for this lesson (single region, ~10 resources): **~$3-5/month**

---

## Step 1: Enable the Config recorder and delivery channel
> **Why:** Config won't do anything until you create a **Configuration Recorder** (what it watches) and a **Delivery Channel** (where it writes snapshots). These are per-region singletons. CDK's L1 CFN constructs are the correct tool — there's no L2 yet.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as config from 'aws-cdk-lib/aws-config';

export class ConfigStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const bucket = new s3.Bucket(this, 'ConfigBucket', {
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: cdk.RemovalPolicy.RETAIN, // auditors hate losing these
    });
    bucket.addToResourcePolicy(new iam.PolicyStatement({
      principals: [new iam.ServicePrincipal('config.amazonaws.com')],
      actions: ['s3:PutObject', 's3:GetBucketAcl'],
      resources: [bucket.bucketArn, `${bucket.bucketArn}/*`],
    }));

    const role = new iam.Role(this, 'ConfigRole', {
      assumedBy: new iam.ServicePrincipal('config.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWS_ConfigRole'),
      ],
    });

    const recorder = new config.CfnConfigurationRecorder(this, 'Recorder', {
      name: 'default',
      roleArn: role.roleArn,
      recordingGroup: { allSupported: true, includeGlobalResourceTypes: true },
    });

    const channel = new config.CfnDeliveryChannel(this, 'Channel', {
      s3BucketName: bucket.bucketName,
    });
    channel.addDependency(recorder);
  }
}
```

Expected after `cdk deploy`:
```
$ aws configservice describe-configuration-recorder-status
{
  "ConfigurationRecordersStatus": [{
    "name": "default",
    "recording": true,
    "lastStatus": "SUCCESS"
  }]
}
```

## Step 2: Attach managed rules
> **Why:** Managed rules cover 90% of common compliance checks with zero code. Each rule evaluates once on config-change or on a schedule. Start with the obvious ones (public buckets, unencrypted volumes, password policy).

```typescript
new config.ManagedRule(this, 'NoPublicBuckets', {
  identifier: config.ManagedRuleIdentifiers.S3_BUCKET_PUBLIC_READ_PROHIBITED,
});

new config.ManagedRule(this, 'EncryptedVolumes', {
  identifier: config.ManagedRuleIdentifiers.EBS_ENCRYPTED_VOLUMES,
});

new config.ManagedRule(this, 'IamPasswordPolicy', {
  identifier: config.ManagedRuleIdentifiers.IAM_PASSWORD_POLICY,
  inputParameters: {
    RequireUppercaseCharacters: 'true',
    RequireLowercaseCharacters: 'true',
    RequireNumbers: 'true',
    MinimumPasswordLength: '14',
  },
});
```

Expected (Config console > Rules): within ~5 min each rule evaluates the account and reports `COMPLIANT` / `NON_COMPLIANT` / `NOT_APPLICABLE` per resource.

## Step 3: Custom rule — require `Environment` tag on EC2
> **Why:** Managed rules don't cover org-specific policies. A custom rule is a Lambda that receives the configuration item, returns a compliance verdict. This pattern scales to anything you can express as code.

`lambda/tag-check.ts`:
```typescript
import { ConfigServiceClient, PutEvaluationsCommand } from '@aws-sdk/client-config-service';

const cfg = new ConfigServiceClient({});

export const handler = async (event: any) => {
  const invoking = JSON.parse(event.invokingEvent);
  const item = invoking.configurationItem;
  const tags = item.tags ?? {};

  const compliance = tags['Environment'] ? 'COMPLIANT' : 'NON_COMPLIANT';

  await cfg.send(new PutEvaluationsCommand({
    Evaluations: [{
      ComplianceResourceType: item.resourceType,
      ComplianceResourceId: item.resourceId,
      ComplianceType: compliance,
      OrderingTimestamp: new Date(item.configurationItemCaptureTime),
    }],
    ResultToken: event.resultToken,
  }));
};
```

Wire it up:
```typescript
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';

const ruleFn = new NodejsFunction(this, 'TagCheckFn', {
  entry: 'lambda/tag-check.ts',
  logRetention: 7,
});
ruleFn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['config:PutEvaluations'],
  resources: ['*'],
}));
ruleFn.addPermission('ConfigInvoke', {
  principal: new iam.ServicePrincipal('config.amazonaws.com'),
});

new config.CustomRule(this, 'RequireEnvTag', {
  lambdaFunction: ruleFn,
  configurationChanges: true,
  ruleScope: config.RuleScope.fromResource(config.ResourceType.EC2_INSTANCE),
});
```

## Step 4: Conformance pack (PCI-DSS sample)
> **Why:** Conformance packs are pre-assembled YAML of dozens of rules + remediation actions for a specific framework. They're how auditors get the compliance report they want in one click.

```typescript
new config.CfnConformancePack(this, 'PciPack', {
  conformancePackName: 'Operational-Best-Practices-PCI-DSS',
  templateS3Uri: 's3://aws-configservice-conformance-packs-us-east-1/Operational-Best-Practices-for-PCI-DSS.yaml',
});
```

Expected (Config console > Conformance packs > Operational-Best-Practices-PCI-DSS): a compliance score like `68% compliant (34/50 rules)` with a drill-down per rule.

## Step 5: Auto-remediation on public S3 buckets
> **Why:** Detection without remediation is just noise. `AWS-DisableS3BucketPublicReadWrite` is an SSM Automation doc AWS provides; Config can trigger it automatically when a bucket flips non-compliant.

```typescript
const remediationRole = new iam.Role(this, 'RemediationRole', {
  assumedBy: new iam.ServicePrincipal('ssm.amazonaws.com'),
  inlinePolicies: {
    fix: new iam.PolicyDocument({
      statements: [new iam.PolicyStatement({
        actions: ['s3:PutBucketPublicAccessBlock', 's3:GetBucketPublicAccessBlock'],
        resources: ['*'],
      })],
    }),
  },
});

new config.CfnRemediationConfiguration(this, 'FixPublicBuckets', {
  configRuleName: 'NoPublicBuckets',   // must match managed-rule id above
  targetId: 'AWS-DisableS3BucketPublicReadWrite',
  targetType: 'SSM_DOCUMENT',
  automatic: true,
  maximumAutomaticAttempts: 3,
  retryAttemptSeconds: 60,
  parameters: {
    S3BucketName: { ResourceValue: { Value: 'RESOURCE_ID' } },
    AutomationAssumeRole: { StaticValue: { Values: [remediationRole.roleArn] } },
  },
});
```

Test it:
```bash
aws s3api create-bucket --bucket test-public-$RANDOM
aws s3api delete-public-access-block --bucket test-public-<suffix>
# wait ~5 min
aws s3api get-public-access-block --bucket test-public-<suffix>
# Expected: BlockPublicAcls=true, BlockPublicPolicy=true (Config remediated)
```

## Step 6: View resource timeline
> **Why:** The timeline view is the single most audit-useful feature of Config. It answers "what did this resource look like on 2026-01-15T03:22Z?" which no other AWS service can.

In the Config console: Resources > pick any recorded resource > "Resource Timeline". You see every config change as a diff, linked to the CloudTrail event that caused it.

```bash
aws configservice get-resource-config-history \
  --resource-type AWS::S3::Bucket \
  --resource-id <bucket-name> \
  --limit 5
```

## Cleanup
```bash
# Stop recording FIRST — otherwise destroy leaves orphan config items billing you.
aws configservice stop-configuration-recorder --configuration-recorder-name default

cdk destroy
# ConfigBucket has RETAIN policy — delete manually if you're sure:
#   aws s3 rb s3://<configbucket> --force
```

## Common Errors
- **`InsufficientDeliveryPolicyException` on deploy** — bucket policy missing the `config.amazonaws.com` principal. Check the `addToResourcePolicy` block.
- **Rule stuck `NO_RESULTS_REPORTED`** — recorder is paused, or rule was created before the recorder was active. Stop/start the recorder.
- **Custom rule Lambda returns but compliance never updates** — missing `config:PutEvaluations` permission, OR `resultToken` not echoed back.
- **Conformance pack deploy fails with `LimitExceededException`** — default limit is 60 rules per region. Request a quota increase.
- **Auto-remediation fires in a loop** — the SSM doc's action itself triggers another config-change event. Use `maximumAutomaticAttempts` to cap, and scope the rule narrowly.
- **Bill spike after enabling** — `allSupported: true` + `includeGlobalResourceTypes: true` in every region records IAM globally N times. Enable global resources in only one region.
- **`cdk destroy` hangs on ConfigurationRecorder** — CloudFormation can't delete a recorder that's still recording. Run `stop-configuration-recorder` first.
