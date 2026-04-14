# Walkthrough — 01 CDK Patterns

## About this service
**AWS CDK** (Cloud Development Kit) lets you define infrastructure in real programming languages. It compiles (`synth`) to CloudFormation. Patterns in CDK fall into levels: **L1** = raw CFN resources (`CfnBucket`), **L2** = typed constructs with sensible defaults (`Bucket`), **L3** = opinionated higher-level patterns (e.g. `ApplicationLoadBalancedFargateService`). On top, **Aspects** walk the construct tree to enforce rules, and **Custom Resources** let CloudFormation manage non-CFN-native things via Lambda.

**Why it matters:** L3 + Aspects + cdk-nag are how teams enforce security and consistency across dozens of stacks without copy-paste. Writing your own L3 once saves hundreds of lines per project.

**When to use:** multi-team orgs that need paved-road constructs, compliance-heavy environments, anywhere you find yourself repeating the same 10 lines of boilerplate.
**When NOT to use:** one-off prototypes, tiny projects with <3 resources — raw L2 is fine. Don't over-abstract early.

## Estimated cost
- **CDK itself: $0/month** (it's a CLI / library)
- **CloudFormation: $0/month** (no charge for stacks; only the resources they create)
- **Custom Resource Lambda: ~$0.00** (a few invocations per deploy, within free tier)
- **S3 bucket created by SecureBucket: ~$0.023/GB/month** (assume <1GB test data = <$0.03)
- **DynamoDB table seeded by custom resource: ~$0.00** (on-demand, <1M requests)
- Total for this lesson: **~$0.05/month**

---

## Step 1: Project setup
> **Why:** A fresh CDK app gives you a clean place to publish an L3 construct and consume it in a sibling stack. We keep everything in one app initially; real orgs publish to a private npm registry.

```bash
mkdir cdk-patterns && cd cdk-patterns
cdk init app --language=typescript
npm install aws-cdk-lib constructs cdk-nag
```

## Step 2: Build an L3 `SecureBucket` construct
> **Why:** Baking encryption, versioning, lifecycle and public-access-block into one construct means engineers can't forget them. This is the core value of L3 — enforce the 80% case by default.

Create `lib/secure-bucket.ts`:
```typescript
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cdk from 'aws-cdk-lib';

export interface SecureBucketProps {
  readonly bucketName?: string;
  readonly expirationDays?: number;
}

export class SecureBucket extends Construct {
  readonly bucket: s3.Bucket;

  constructor(scope: Construct, id: string, props: SecureBucketProps = {}) {
    super(scope, id);

    this.bucket = new s3.Bucket(this, 'Bucket', {
      bucketName: props.bucketName,
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      enforceSSL: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      lifecycleRules: [{
        noncurrentVersionExpiration: cdk.Duration.days(props.expirationDays ?? 30),
      }],
    });
  }
}
```

## Step 3: Consume the L3 + add a cdk-nag Aspect
> **Why:** `AwsSolutionsChecks` runs ~200 AWS Well-Architected rules at synth time. Catching a HIPAA violation in `cdk synth` is 1000x cheaper than catching it in a SOC 2 audit.

`lib/cdk-patterns-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { Aspects, IAspect } from 'aws-cdk-lib';
import { IConstruct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { AwsSolutionsChecks, NagSuppressions } from 'cdk-nag';
import { SecureBucket } from './secure-bucket';

// Aspect that enforces RETAIN on all buckets
class RetainAllBuckets implements IAspect {
  visit(node: IConstruct): void {
    if (node instanceof s3.CfnBucket) {
      node.applyRemovalPolicy(cdk.RemovalPolicy.RETAIN);
    }
  }
}

export class CdkPatternsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new SecureBucket(this, 'Data', { expirationDays: 60 });

    Aspects.of(this).add(new RetainAllBuckets());
    Aspects.of(this).add(new AwsSolutionsChecks({ verbose: true }));

    // Example suppression with justification
    NagSuppressions.addStackSuppressions(this, [
      { id: 'AwsSolutions-S1', reason: 'Access logs not needed for test bucket' },
    ]);
  }
}
```

## Step 4: Custom Resource that seeds DynamoDB on deploy
> **Why:** CloudFormation has no "insert row" primitive. A custom resource wraps a Lambda that runs on create/update/delete — the canonical way to do one-time setup like seed data, Cognito user creation, or DNS records in another system.

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { AwsCustomResource, AwsCustomResourcePolicy, PhysicalResourceId } from 'aws-cdk-lib/custom-resources';

const table = new dynamodb.Table(this, 'Seed', {
  partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

new AwsCustomResource(this, 'SeedData', {
  onCreate: {
    service: 'DynamoDB',
    action: 'putItem',
    parameters: {
      TableName: table.tableName,
      Item: { pk: { S: 'config' }, version: { S: '1.0' } },
    },
    physicalResourceId: PhysicalResourceId.of('SeedData'),
  },
  policy: AwsCustomResourcePolicy.fromSdkCalls({ resources: [table.tableArn] }),
});
```

## Step 5: Synth and verify
> **Why:** `cdk synth` is the single source of truth — the CloudFormation it prints is literally what will deploy. Diffing synth output before a PR merge is the #1 catch-all review technique.

```bash
cdk synth
# Resources:
#   DataBucketE3889A50: Type: AWS::S3::Bucket ...
#   SeedTable...:      Type: AWS::DynamoDB::Table
#   SeedDataCustom...: Type: Custom::AWS
#
# cdk-nag findings:
#   [Error at /CdkPatternsStack/Data/Bucket/Resource] AwsSolutions-S1: ...
```

Fix any nag errors, then deploy:
```bash
cdk deploy
```

## Step 6: Cross-stack reference via SSM
> **Why:** `Fn.importValue` creates a hard CFN dependency — you cannot delete the producer stack while consumers exist. SSM parameters break that coupling: the consumer reads at deploy time, not synth time.

Producer:
```typescript
import * as ssm from 'aws-cdk-lib/aws-ssm';
new ssm.StringParameter(this, 'VpcIdParam', {
  parameterName: '/shared/vpc-id',
  stringValue: vpc.vpcId,
});
```

Consumer:
```typescript
const vpcId = ssm.StringParameter.valueFromLookup(this, '/shared/vpc-id');
```

## Cleanup
```bash
cdk destroy
# SecureBucket has RETAIN — delete manually if desired:
aws s3 rb s3://<bucket-name> --force
```

## Common Errors
- **`AwsSolutions-IAM5` on custom resource** → add a targeted `NagSuppressions.addResourceSuppressions` with justification.
- **Custom resource hangs on delete** → Lambda must respond within 1 hour; ensure `onDelete` is a no-op (or omitted) if you don't need delete behavior.
- **`Cannot use Fn.importValue ... still referenced`** → a consumer stack still imports the export. Refactor to SSM parameters.
- **`cdk-nag` errors block `cdk deploy`** → either fix the underlying issue or add suppression with a `reason` (required).
