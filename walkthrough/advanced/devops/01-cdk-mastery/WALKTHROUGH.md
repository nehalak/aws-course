# Walkthrough — 01 CDK Mastery

## About this service
**AWS CDK** is infrastructure-as-code in a real programming language. You write TypeScript/Python/Java/Go/.NET; CDK synthesizes CloudFormation. CDK v2 ships a single package (`aws-cdk-lib`) plus L1 (raw CFN), L2 (opinionated), and L3/pattern constructs. Mastery means building your own reusable L3 construct library, enforcing policy via **Aspects** and **cdk-nag**, and shipping the library to multiple languages with **jsii** / **projen**.

**Why it matters:** A shared construct library (`SecureBucket`, `HardenedVpc`, `AuditedLambda`) is how platform teams enforce guardrails without blocking product teams. Aspects + cdk-nag block non-compliant infra at synth time — long before CloudFormation touches an account.
**When to use:** multi-team orgs, regulated environments (HIPAA/PCI/FedRAMP), polyglot consumers (TS + Python CDK apps), platform engineering.
**When NOT to use:** single-stack toy projects (plain CDK app is fine), teams that never reuse patterns, orgs mandated on Terraform.

## Estimated cost
- CDK itself is free — you pay only for synthesized resources.
- **CloudFormation**: free for AWS-native resources; $0.0009 per handler operation for third-party resources.
- **CDK bootstrap stack (CDKToolkit)**: S3 bucket (~$0.10/mo for ~4GB of assets) + ECR repo (~$0.10/mo) + KMS key ($1.00/mo).
- **cdk-nag / Aspects**: free (run locally at synth).
- **jsii package publish**: npm/PyPI/Maven Central are free for public packages; CodeArtifact private repo ~$0.05/GB-mo + $0.05/10k requests.
- Sample `SecureBucket` stack: S3 bucket (empty) = $0.00, KMS CMK = $1.00/mo.
- Total for this lesson: **~$2.20/month** (bootstrap + 1 CMK).

---

## Step 1: Scaffold a jsii construct library with projen
> **Why:** `projen` generates and owns the entire build config (package.json, tsconfig, jest, .github/workflows, .projenrc.ts) so you never hand-edit it. `cdk-construct-library` preset wires jsii for multi-language publishing out of the box.

```bash
mkdir aws-secure-patterns && cd aws-secure-patterns
npx projen new awscdk-construct \
  --name @yourorg/aws-secure-patterns \
  --author "Your Org" --authorAddress platform@yourorg.io \
  --cdkVersion 2.140.0 \
  --repositoryUrl https://github.com/yourorg/aws-secure-patterns
```

Edit `.projenrc.ts` to add Python + Java targets:
```typescript
import { awscdk } from 'projen';

const project = new awscdk.AwsCdkConstructLibrary({
  author: 'Your Org',
  authorAddress: 'platform@yourorg.io',
  cdkVersion: '2.140.0',
  defaultReleaseBranch: 'main',
  name: '@yourorg/aws-secure-patterns',
  repositoryUrl: 'https://github.com/yourorg/aws-secure-patterns',
  jsiiVersion: '~5.4.0',
  publishToPypi: { distName: 'yourorg.aws-secure-patterns', module: 'yourorg_aws_secure_patterns' },
  publishToMaven: {
    javaPackage: 'io.yourorg.awssecurepatterns',
    mavenGroupId: 'io.yourorg',
    mavenArtifactId: 'aws-secure-patterns',
  },
  devDeps: ['cdk-nag@^2.28.0'],
  peerDeps: ['cdk-nag@^2.28.0'],
});
project.synth();
```

```bash
npx projen
```

## Step 2: Author the `SecureBucket` L3 construct
> **Why:** jsii requires exported API surfaces to be "jsii-safe" — classes not interfaces for props with defaults, no union types, no generics leaking out. Keep L3 props flat and enum-driven.

`src/secure-bucket.ts`:
```typescript
import { RemovalPolicy, Duration } from 'aws-cdk-lib';
import {
  Bucket,
  BucketEncryption,
  BlockPublicAccess,
  ObjectOwnership,
  BucketProps,
} from 'aws-cdk-lib/aws-s3';
import { Key } from 'aws-cdk-lib/aws-kms';
import { Construct } from 'constructs';

export interface SecureBucketProps {
  readonly bucketName?: string;
  readonly retainOnDelete?: boolean;
  readonly expireNonCurrentAfterDays?: number;
}

export class SecureBucket extends Construct {
  public readonly bucket: Bucket;
  public readonly key: Key;

  constructor(scope: Construct, id: string, props: SecureBucketProps = {}) {
    super(scope, id);

    this.key = new Key(this, 'Cmk', {
      enableKeyRotation: true,
      removalPolicy: props.retainOnDelete ? RemovalPolicy.RETAIN : RemovalPolicy.DESTROY,
    });

    this.bucket = new Bucket(this, 'Bucket', {
      bucketName: props.bucketName,
      encryption: BucketEncryption.KMS,
      encryptionKey: this.key,
      enforceSSL: true,
      versioned: true,
      blockPublicAccess: BlockPublicAccess.BLOCK_ALL,
      objectOwnership: ObjectOwnership.BUCKET_OWNER_ENFORCED,
      removalPolicy: props.retainOnDelete ? RemovalPolicy.RETAIN : RemovalPolicy.DESTROY,
      autoDeleteObjects: !props.retainOnDelete,
      lifecycleRules: [{
        noncurrentVersionExpiration: Duration.days(props.expireNonCurrentAfterDays ?? 90),
      }],
    });
  }
}
```

Export from `src/index.ts`:
```typescript
export * from './secure-bucket';
export * from './hardened-vpc';
export * from './audited-lambda';
export * from './aspects/require-tag';
```

## Step 3: CostCenter-tag Aspect
> **Why:** Aspects walk the construct tree at synth. Emitting `Annotations.addError` fails `cdk synth` — the PR can't merge if a resource lacks the tag. This is the cheapest policy enforcement point.

`src/aspects/require-tag.ts`:
```typescript
import { IAspect, Annotations, CfnResource, TagManager } from 'aws-cdk-lib';
import { IConstruct } from 'constructs';

export class RequireTag implements IAspect {
  constructor(private readonly tagKey: string) {}

  public visit(node: IConstruct): void {
    if (!(node instanceof CfnResource)) return;
    // Not every CFN resource supports tagging — skip those.
    if (!TagManager.isTaggable(node)) return;
    const tags = node.tags.renderTags() as Array<{ Key: string; Value: string }> | undefined;
    const has = tags?.some((t) => t.Key === this.tagKey && t.Value);
    if (!has) {
      Annotations.of(node).addError(
        `Resource ${node.node.path} missing required tag "${this.tagKey}"`
      );
    }
  }
}
```

## Step 4: Wire cdk-nag (AWS Solutions + HIPAA)
> **Why:** cdk-nag ships curated rule packs. Running both catches ~80% of real-world audit findings before a CFN deploy. Suppressions must be explicit + justified — they show up in the synth output for review.

Example consumer app `bin/app.ts`:
```typescript
import { App, Aspects } from 'aws-cdk-lib';
import { AwsSolutionsChecks, HIPAASecurityChecks, NagSuppressions } from 'cdk-nag';
import { RequireTag } from '@yourorg/aws-secure-patterns';
import { DemoStack } from '../lib/demo-stack';

const app = new App();
const stack = new DemoStack(app, 'DemoStack');

Aspects.of(app).add(new AwsSolutionsChecks({ verbose: true }));
Aspects.of(app).add(new HIPAASecurityChecks());
Aspects.of(app).add(new RequireTag('CostCenter'));

NagSuppressions.addStackSuppressions(stack, [
  { id: 'AwsSolutions-IAM4', reason: 'Managed AWSLambdaBasicExecutionRole is approved for this account.' },
]);
```

Run:
```bash
npx cdk synth
# [Error at /DemoStack/Bucket/Resource] AwsSolutions-S1: Bucket has server access logs disabled.
```

## Step 5: Unit tests with @aws-cdk/assertions
> **Why:** Fine-grained assertions pin specific properties — they survive refactors better than snapshot-only tests. Snapshot tests catch unintended diffs; combine both.

`test/secure-bucket.test.ts`:
```typescript
import { App, Stack } from 'aws-cdk-lib';
import { Template, Match } from 'aws-cdk-lib/assertions';
import { SecureBucket } from '../src';

test('SecureBucket enforces KMS + blocks public + enforces SSL', () => {
  const stack = new Stack(new App(), 'T');
  new SecureBucket(stack, 'B');
  const tpl = Template.fromStack(stack);

  tpl.hasResourceProperties('AWS::S3::Bucket', {
    BucketEncryption: {
      ServerSideEncryptionConfiguration: [
        Match.objectLike({ ServerSideEncryptionByDefault: { SSEAlgorithm: 'aws:kms' } }),
      ],
    },
    PublicAccessBlockConfiguration: {
      BlockPublicAcls: true, BlockPublicPolicy: true,
      IgnorePublicAcls: true, RestrictPublicBuckets: true,
    },
  });

  tpl.hasResourceProperties('AWS::S3::BucketPolicy', {
    PolicyDocument: Match.objectLike({
      Statement: Match.arrayWith([
        Match.objectLike({ Effect: 'Deny', Condition: { Bool: { 'aws:SecureTransport': 'false' } } }),
      ]),
    }),
  });
});

test('Template snapshot', () => {
  const stack = new Stack(new App(), 'T');
  new SecureBucket(stack, 'B');
  expect(Template.fromStack(stack).toJSON()).toMatchSnapshot();
});
```

```bash
npx projen test
# PASS test/secure-bucket.test.ts
```

## Step 6: Integration test with @aws-cdk/integ-tests
> **Why:** Unit assertions only check the synthesized template. `integ-tests-alpha` actually deploys the stack and lets you assert runtime behavior (S3 PutObject fails for non-KMS request, etc).

`test/integ.secure-bucket.ts`:
```typescript
import { App, Stack } from 'aws-cdk-lib';
import { IntegTest, ExpectedResult } from '@aws-cdk/integ-tests-alpha';
import { SecureBucket } from '../src';

const app = new App();
const stack = new Stack(app, 'IntegSecureBucket');
const b = new SecureBucket(stack, 'B');

const integ = new IntegTest(app, 'SecureBucketInteg', { testCases: [stack] });

integ.assertions
  .awsApiCall('S3', 'getBucketEncryption', { Bucket: b.bucket.bucketName })
  .expect(ExpectedResult.objectLike({
    ServerSideEncryptionConfiguration: { Rules: [{ ApplyServerSideEncryptionByDefault: { SSEAlgorithm: 'aws:kms' } }] },
  }));
```

```bash
npx integ-runner --update-on-failed --directory test
```

## Step 7: Publish locally with npm pack and consume
> **Why:** `npm pack` produces the exact tarball that would go to the registry — it's the fastest way to validate packaging without polluting npm. Consuming from a sibling repo catches export/typing mistakes projen can hide.

```bash
npx projen build
npx projen package
npm pack dist/js/*.tgz
# yourorg-aws-secure-patterns-0.1.0.tgz

cd ../demo-app
npm install ../aws-secure-patterns/yourorg-aws-secure-patterns-0.1.0.tgz
```

`lib/demo-stack.ts`:
```typescript
import { Stack, StackProps, Tags } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { SecureBucket } from '@yourorg/aws-secure-patterns';

export class DemoStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);
    const b = new SecureBucket(this, 'Bucket');
    Tags.of(b).add('CostCenter', 'CC-4711');
  }
}
```

## Step 8: CI gate on cdk-nag
> **Why:** If cdk-nag only runs locally, it won't. GitHub Actions runs `cdk synth --strict` on every PR; any `Annotations.addError` causes a non-zero exit and blocks merge.

`.github/workflows/synth.yml`:
```yaml
name: cdk-synth
on: [pull_request]
jobs:
  synth:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx cdk synth --strict --validation
```

## Cleanup
```bash
# Destroy demo consumer app (keeps library source):
cd demo-app && npx cdk destroy --all
# Optionally tear down the CDK bootstrap (careful — shared across projects):
aws cloudformation delete-stack --stack-name CDKToolkit
```

## Common Errors
- **`jsii: error: Type ... is not exported`** → your public API references a non-exported interface. Export it from `src/index.ts` or make it internal.
- **`Aspect ... adding/removing nodes during synth is not permitted`** → move resource creation out of the Aspect; Aspects may only annotate or mutate existing nodes.
- **`cdk-nag: AwsSolutions-IAM5` on `*` resource** → narrow the IAM policy, or add a justified `NagSuppressions` entry with an `appliesTo` selector.
- **`integ-runner` hangs** → your integ test has no `IntegTest` construct; every integ file must instantiate one.
- **`npm pack` tarball missing `.d.ts`** → jsii build didn't run; use `npx projen package` not plain `tsc`.
- **Snapshot mismatch after CDK upgrade** → regenerate with `npx jest -u`, review the diff carefully before committing.
