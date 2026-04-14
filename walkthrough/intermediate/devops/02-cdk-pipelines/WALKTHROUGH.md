# Walkthrough — 02 CDK Pipelines

## About this service
**CDK Pipelines** is a high-level construct that builds a self-mutating CI/CD pipeline on top of **AWS CodePipeline** + **CodeBuild**. You define stages (dev, prod, etc.) in CDK; the pipeline watches your source repo (GitHub, CodeCommit, S3) and on every push: pulls source → `cdk synth` → updates itself (self-mutation) → deploys each stage. Deploys can span accounts and regions, run tests between stages, and include manual approval gates.

**Why it matters:** You stop maintaining YAML in two places (infra + pipeline). Pipeline changes ship the same way app changes ship: `git push`. Multi-account, multi-region prod rollouts become ~30 lines of TypeScript.

**When to use:** teams deploying CDK apps to >1 environment, anyone who needs audit trails and approval gates, multi-account AWS Organizations.
**When NOT to use:** non-CDK workloads (use plain CodePipeline or GitHub Actions), solo prototypes where `cdk deploy` from your laptop is enough.

## Estimated cost
- **CodePipeline V2 pipeline: $1.00/month** per active pipeline (the pipeline resource itself)
- **CodeBuild: $0.005/min** on `BUILD_GENERAL1_SMALL` — a typical synth+deploy = 3-5 min per run
- **S3 artifact bucket: ~$0.023/GB/month** (usually pennies)
- **CodeStar Connection (GitHub): free**
- Example: 1 pipeline, 20 pushes/month × 5 min × $0.005 = $0.50 build + $1.00 pipeline
- Total for this lesson: **~$1.50/month**

---

## Step 1: Prereqs — bootstrap & GitHub connection
> **Why:** CDK Pipelines needs the modern bootstrap stack (v10+) which includes cross-account trust roles. The GitHub connection (CodeStar Connections) is a one-time OAuth handshake done in the console — it cannot be created programmatically.

```bash
# Modern bootstrap in every account/region you will deploy to
cdk bootstrap aws://111111111111/us-east-1
cdk bootstrap aws://222222222222/us-east-1 --trust 111111111111 \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess
```

In the AWS console → Developer Tools → **Settings → Connections → Create connection → GitHub**. Authorize Anthropic, then copy the ARN:
```
arn:aws:codeconnections:us-east-1:111111111111:connection/abc-123-def
```

## Step 2: App entrypoint with the pipeline stack
> **Why:** The pipeline itself lives in one stack in your "tooling" account. Application stages are instantiated and added to the pipeline — they will be deployed into dev/prod accounts.

`bin/app.ts`:
```typescript
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { PipelineStack } from '../lib/pipeline-stack';

const app = new cdk.App();
new PipelineStack(app, 'PipelineStack', {
  env: { account: '111111111111', region: 'us-east-1' },
});
```

## Step 3: Define the application stage
> **Why:** A `Stage` is a reusable bundle of stacks you want deployed together. You instantiate it multiple times (dev, prod, eu-prod) with different props — same code, different envs.

`lib/app-stage.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';

export class AppStage extends cdk.Stage {
  readonly apiUrl: cdk.CfnOutput;

  constructor(scope: Construct, id: string, props?: cdk.StageProps) {
    super(scope, id, props);

    const stack = new cdk.Stack(this, 'AppStack');
    const fn = new lambda.Function(stack, 'Api', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(
        "exports.handler = async () => ({ statusCode: 200, body: 'ok' });"
      ),
    });
    const url = fn.addFunctionUrl({ authType: lambda.FunctionUrlAuthType.NONE });
    this.apiUrl = new cdk.CfnOutput(stack, 'ApiUrl', { value: url.url });
  }
}
```

## Step 4: Pipeline with dev, prod (manual approval), and multi-region wave
> **Why:** `addStage` is sequential; `addWave` runs stages in parallel. Manual approval before prod is the cheapest safety net known to DevOps. Cross-region waves let you deploy us-east-1 and eu-west-1 simultaneously after prod approval.

`lib/pipeline-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { CodePipeline, CodePipelineSource, ShellStep, ManualApprovalStep } from 'aws-cdk-lib/pipelines';
import { AppStage } from './app-stage';

export class PipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const pipeline = new CodePipeline(this, 'Pipeline', {
      pipelineName: 'AppPipeline',
      synth: new ShellStep('Synth', {
        input: CodePipelineSource.connection('my-org/my-repo', 'main', {
          connectionArn: 'arn:aws:codeconnections:us-east-1:111111111111:connection/abc-123-def',
        }),
        commands: ['npm ci', 'npm run build', 'npx cdk synth'],
      }),
    });

    // Dev
    const dev = new AppStage(this, 'Dev', {
      env: { account: '222222222222', region: 'us-east-1' },
    });
    pipeline.addStage(dev, {
      post: [new ShellStep('SmokeTest', {
        envFromCfnOutputs: { URL: dev.apiUrl },
        commands: ['curl -fsS $URL || exit 1'],
      })],
    });

    // Prod with approval + multi-region wave
    const prodWave = pipeline.addWave('Prod', {
      pre: [new ManualApprovalStep('PromoteToProd')],
    });
    prodWave.addStage(new AppStage(this, 'ProdUsEast', {
      env: { account: '333333333333', region: 'us-east-1' },
    }));
    prodWave.addStage(new AppStage(this, 'ProdEuWest', {
      env: { account: '333333333333', region: 'eu-west-1' },
    }));
  }
}
```

## Step 5: First deploy (bootstrap the pipeline)
> **Why:** You only ever run `cdk deploy` once — to create the pipeline. After that, every change (including to the pipeline itself) flows through `git push`.

```bash
cdk deploy PipelineStack
# PipelineStack: creating CloudFormation changeset...
# Outputs:
#   PipelineStack.PipelineName = AppPipeline
```

Open **CodePipeline console** → watch 4 stages: Source → Build → UpdatePipeline (self-mutation) → Assets → Dev → Prod.

## Step 6: Trigger a self-mutation
> **Why:** The magic moment: modifying the pipeline definition and letting the running pipeline update itself proves end-to-end that you never need to touch CLI again.

```bash
# Edit lib/pipeline-stack.ts — e.g. add another region to the wave
git add . && git commit -m "add ap-southeast-1" && git push
```

In CodePipeline, watch the `UpdatePipeline` step run — the downstream stages list now includes the new region.

## Step 7: Force a rollback via failed smoke test
> **Why:** Rollback-on-failure is only real if you've seen it work. Break the smoke test intentionally so you trust the gate before shipping prod.

Temporarily change the smoke test to `commands: ['exit 1']`, push. Pipeline halts at Dev; Prod wave never runs. Revert and push again.

## Cleanup
```bash
# Destroy app stages first (pipeline has RETAIN on some assets)
cdk destroy PipelineStack
# Also delete the CodeStar connection in console, and empty the artifact bucket
aws s3 rm s3://<pipeline-artifacts-bucket> --recursive
```

## Common Errors
- **`This CDK CLI is not compatible with the CDK library used by your application`** → bump `aws-cdk` CLI and `aws-cdk-lib` to matching versions.
- **`Policy contains a statement with one or more invalid principals`** during cross-account deploy → target account wasn't bootstrapped with `--trust`.
- **Pipeline stuck on `Source` with `ConnectionArn is invalid`** → connection is still in `PENDING` state; complete OAuth in console.
- **`Need to perform AWS CDK bootstrap in account X region Y`** → run `cdk bootstrap` in every target env before first deploy.
- **`Asset publishing failed: access denied`** → bootstrap version in target account is too old; re-bootstrap.
