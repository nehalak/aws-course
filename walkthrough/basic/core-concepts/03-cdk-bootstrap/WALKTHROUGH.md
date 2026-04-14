# Walkthrough — 03 CDK Bootstrap

## About this service
**AWS CDK (Cloud Development Kit)** lets you define AWS infrastructure in TypeScript/Python/etc., then it compiles to CloudFormation and deploys. `cdk bootstrap` is a one-time per-account/region setup that creates a staging S3 bucket, ECR repo, and 5 IAM roles CDK uses to upload assets and deploy.

**Why use CDK over raw CFN/Terraform:** real programming language (loops, conditionals, imports), type safety, AWS-maintained high-level constructs (`ApplicationLoadBalancedFargateService` replaces 200 lines of YAML).

**When to use CDK:** AWS-only shops, teams comfortable with TypeScript/Python.
**When NOT to use CDK:** multi-cloud (use Terraform), teams that prefer declarative-only (use CloudFormation/Terraform directly), tiny single-service projects (raw CLI is fine).

## Estimated cost
- CDK itself: **free** (it's just a CLI that generates CFN)
- Bootstrap stack resources: **~$0.05/month** (tiny S3 bucket + ECR repo, essentially free)
- Bucket storage scales with asset size (Lambda zips ~few MB, Docker images ~100MB) — still pennies
- Total: **<$1/month** unless you have large Docker assets

---

## Step 1: Install toolchain
> **Why:** CDK is a Node.js CLI. `cdk` synthesizes/deploys, `tsc` compiles TypeScript. You'll use these dozens of times across the course.

```bash
npm install -g aws-cdk typescript
cdk --version       # expect 2.150+
node --version      # expect v18+
```

## Step 2: Bootstrap
> **Why:** CDK needs a place to upload Lambda zips, Docker images, and templates too big for inline CFN. `cdk bootstrap` creates that shared infrastructure. You do this **once per account/region combination**.

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
cdk bootstrap aws://$ACCOUNT/us-east-1
cdk bootstrap aws://$ACCOUNT/eu-west-1
```

Expected output:
```
 ⏳  Bootstrapping environment aws://123456789012/us-east-1...
 ✅  Environment aws://123456789012/us-east-1 bootstrapped.
```

## Step 3: Inspect CDKToolkit stack
> **Why:** Understanding what bootstrap creates demystifies CDK. When something fails ("can't upload asset"), you'll know which role or bucket is involved.

**CloudFormation → Stacks → CDKToolkit → Resources**. Document:

| Logical ID             | Type                  | Purpose                                  |
|------------------------|-----------------------|------------------------------------------|
| StagingBucket          | AWS::S3::Bucket       | Upload assets (Lambda zips, Docker)      |
| ContainerAssetsRepo    | AWS::ECR::Repository  | Docker image assets                      |
| FilePublishingRole     | AWS::IAM::Role        | Upload assets (assumed by CLI)           |
| ImagePublishingRole    | AWS::IAM::Role        | Push ECR images                          |
| LookupRole             | AWS::IAM::Role        | Run `Vpc.fromLookup` etc.                |
| DeploymentActionRole   | AWS::IAM::Role        | Deploy (create change sets)              |
| CloudFormationExecutionRole | AWS::IAM::Role   | CFN uses this to create resources        |

## Step 4: First app
> **Why:** A minimal app (one S3 bucket) proves the whole pipeline works: CDK → CFN → AWS. From here everything else is just adding more constructs.

```bash
mkdir hello-cdk && cd hello-cdk
cdk init app --language=typescript
```

Edit `lib/hello-cdk-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class HelloCdkStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new s3.Bucket(this, 'HelloBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });
  }
}
```

Edit `bin/hello-cdk.ts`:

```typescript
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { HelloCdkStack } from '../lib/hello-cdk-stack';

const app = new cdk.App();
new HelloCdkStack(app, 'HelloCdkStack-USE1', {
  env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: 'us-east-1' },
});
new HelloCdkStack(app, 'HelloCdkStack-EUW1', {
  env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: 'eu-west-1' },
});
```

## Step 5: Deploy
> **Why:** `synth` shows you the generated CFN (learning aid). `diff` previews changes (saves you from surprises). `deploy` actually creates resources.

```bash
cdk synth                 # prints CFN YAML
cdk diff HelloCdkStack-USE1
cdk deploy --all
```

Expected output:
```
✨  Synthesis time: 3.2s
HelloCdkStack-USE1: deploying...
 ✅  HelloCdkStack-USE1
```

## Step 6: Verify
```bash
aws s3 ls --region us-east-1 | grep hellocdk
aws s3 ls --region eu-west-1 | grep hellocdk
```

## Step 7: Cleanup
> **Why:** Destroy everything you don't need. Every empty bucket, every orphaned volume costs something. Build the habit now.

```bash
cdk destroy --all
```

## Common Errors
- **"Need to perform AWS calls for account X, but no credentials..."** → `aws configure` wasn't done.
- **"This stack uses assets, so the toolkit stack must be deployed"** → Run `cdk bootstrap` for that region.
- **"context providers required but no context available"** → run `cdk synth` first, or commit `cdk.context.json`.
