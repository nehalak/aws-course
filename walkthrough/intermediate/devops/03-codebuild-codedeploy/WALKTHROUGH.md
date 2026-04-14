# Walkthrough — 03 CodeBuild & CodeDeploy

## About this service
**AWS CodeBuild** is a managed build service: you give it a `buildspec.yml` and a source, it spins up a container, runs your commands, and publishes artifacts. Pay per build-minute. **AWS CodeDeploy** is a managed deployment service: it orchestrates rolling / blue-green / canary rollouts to EC2, on-prem, Lambda, or ECS using `appspec.yml`. They're often paired — CodeBuild produces an artifact, CodeDeploy ships it.

**Why it matters:** CodeBuild replaces Jenkins for most AWS-heavy teams. CodeDeploy's canary + auto-rollback for Lambda/ECS is the easiest safe-deploy mechanism in AWS — one alarm triggers a full rollback with zero human intervention.

**When to use:** CodeBuild when you want build infra that scales to zero, integrates with IAM, and runs in your VPC. CodeDeploy for Lambda canary, ECS blue/green, or EC2 rolling updates where you need hooks + rollback on alarm.
**When NOT to use:** CodeBuild — if your team already has GitHub Actions or CircleCI humming. CodeDeploy — for simple container pushes (just update the task definition) or anything not on EC2/Lambda/ECS.

## Estimated cost
- **CodeBuild `general1.small` (3 vCPU, 3GB Linux): $0.005/min** — first 100 min/month free
- **CodeBuild `general1.medium`: $0.01/min**
- **CodeDeploy for EC2 / on-prem: $0.02 per on-prem instance update** (EC2 deploys are free)
- **CodeDeploy for Lambda / ECS: free**
- Example: 50 builds × 4 min × $0.005 = $1.00 + CodeDeploy free = $1.00
- Total for this lesson: **~$1-2/month**

---

## Step 1: Project and basic buildspec
> **Why:** `buildspec.yml` is the contract between CodeBuild and your build. Versioning it in the repo means builds are reproducible and reviewable. `phases` run in order, each in a fresh shell.

```bash
mkdir codebuild-deploy && cd codebuild-deploy
cdk init app --language=typescript
npm install aws-cdk-lib constructs
```

`buildspec.yml` at repo root:
```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to ECR
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URI
  build:
    commands:
      - echo Build started on `date`
      - docker build -t $IMAGE_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $IMAGE_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION $ECR_URI/$IMAGE_REPO:latest
  post_build:
    commands:
      - docker push $ECR_URI/$IMAGE_REPO:latest
      - printf '[{"name":"app","imageUri":"%s"}]' $ECR_URI/$IMAGE_REPO:latest > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef.json
```

## Step 2: CodeBuild project in CDK
> **Why:** `BuildSpec.fromSourceFilename` points at the file in the repo (vs inline) so your CI config lives with the code. `privileged: true` is required for Docker-in-Docker (building images).

`lib/build-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as codebuild from 'aws-cdk-lib/aws-codebuild';
import * as ecr from 'aws-cdk-lib/aws-ecr';

export class BuildStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const repo = new ecr.Repository(this, 'AppRepo', {
      repositoryName: 'myapp',
      imageScanOnPush: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      emptyOnDelete: true,
    });

    const project = new codebuild.Project(this, 'BuildProject', {
      projectName: 'myapp-build',
      source: codebuild.Source.gitHub({
        owner: 'my-org',
        repo: 'my-repo',
        webhook: true,
        webhookFilters: [
          codebuild.FilterGroup.inEventOf(codebuild.EventAction.PUSH).andBranchIs('main'),
        ],
      }),
      environment: {
        buildImage: codebuild.LinuxBuildImage.STANDARD_7_0,
        privileged: true, // required for docker build
        computeType: codebuild.ComputeType.SMALL,
      },
      environmentVariables: {
        AWS_REGION: { value: cdk.Stack.of(this).region },
        ECR_URI:    { value: `${cdk.Stack.of(this).account}.dkr.ecr.${cdk.Stack.of(this).region}.amazonaws.com` },
        IMAGE_REPO: { value: repo.repositoryName },
      },
      buildSpec: codebuild.BuildSpec.fromSourceFilename('buildspec.yml'),
    });

    repo.grantPullPush(project);
  }
}
```

Deploy:
```bash
cdk deploy
# BuildStack: ✅  myapp-build project created
```

## Step 3: Trigger a build and watch logs
> **Why:** First-build failures are almost always `buildspec.yml` syntax or missing IAM — you want to see them now, not during a release.

```bash
aws codebuild start-build --project-name myapp-build
# { "build": { "id": "myapp-build:abc-123", "buildStatus": "IN_PROGRESS" } }

aws codebuild batch-get-builds --ids myapp-build:abc-123 \
  --query 'builds[0].logs.deepLink'
```

Expected tail of logs:
```
[Container] ... Running command docker push ...
latest: digest: sha256:9f2b... size: 1788
[Container] ... Phase complete: POST_BUILD State: SUCCEEDED
```

## Step 4: Lambda canary deploy with CodeDeploy
> **Why:** Canary = shift a small % of traffic to the new version, wait, then flip the rest. If CloudWatch alarms fire during the bake window, CodeDeploy auto-rolls back to the previous Lambda version. This is the simplest safe-deploy in AWS.

`lib/lambda-canary-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';

export class LambdaCanaryStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const fn = new lambda.Function(this, 'Api', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(
        "exports.handler = async () => ({ statusCode: 200, body: 'v1' });"
      ),
    });

    const version = fn.currentVersion;
    const alias = new lambda.Alias(this, 'LiveAlias', {
      aliasName: 'live',
      version,
    });

    const errorsAlarm = new cloudwatch.Alarm(this, 'ErrorsAlarm', {
      metric: alias.metricErrors({ period: cdk.Duration.minutes(1) }),
      threshold: 1,
      evaluationPeriods: 1,
    });

    new codedeploy.LambdaDeploymentGroup(this, 'DeployGroup', {
      alias,
      deploymentConfig: codedeploy.LambdaDeploymentConfig.CANARY_10PERCENT_5MINUTES,
      alarms: [errorsAlarm],
      autoRollback: { failedDeployment: true, deploymentInAlarm: true },
    });
  }
}
```

On next `cdk deploy` with a code change, CodeDeploy shifts 10% of traffic for 5 minutes; if `ErrorsAlarm` breaches, it rolls back automatically.

## Step 5: Verify rollback by forcing errors
> **Why:** Rollback is worthless if you haven't tested it. Deploy a version that throws and confirm CodeDeploy aborts.

Change the handler:
```typescript
code: lambda.Code.fromInline("exports.handler = async () => { throw new Error('boom'); };"),
```

```bash
cdk deploy
# CodeDeploy console → Deployments → status: Failed → "Rollback initiated: deployment alarm"
aws lambda invoke --function-name <name>:live --payload '{}' out.json
# Returns v1 output — rollback succeeded
```

## Step 6: ECS Blue/Green (concept)
> **Why:** Blue/Green for ECS requires two target groups and two listeners on an ALB. CodeDeploy spins up the green task set, shifts the production listener when healthy, and keeps blue around for `terminationWaitTime` in case you need to bail.

```typescript
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';

new codedeploy.EcsDeploymentGroup(this, 'EcsBg', {
  service,  // ecs.FargateService with deploymentController ECS/CODE_DEPLOY
  blueGreenDeploymentConfig: {
    blueTargetGroup: tg1,
    greenTargetGroup: tg2,
    listener: prodListener,
    testListener,
    terminationWaitTime: cdk.Duration.minutes(5),
  },
  deploymentConfig: codedeploy.EcsDeploymentConfig.CANARY_10PERCENT_5MINUTES,
});
```

## Cleanup
```bash
cdk destroy
# Also delete CodeDeploy applications + deployment groups if not owned by CDK
aws deploy list-applications
```

## Common Errors
- **`YAML_FILE_ERROR: YAML file does not exist`** → `buildspec.yml` not at repo root, or path mismatch in `BuildSpec.fromSourceFilename`.
- **`Cannot perform an interactive login from a non TTY device`** (ECR) → using `docker login -u -p` instead of `--password-stdin`; also means `privileged: true` is missing.
- **`AccessDenied when calling PutImage`** → build role missing `ecr:PutImage`; use `repo.grantPullPush(project)`.
- **CodeDeploy canary stuck "In progress" forever** → no traffic hitting the alias so CloudWatch alarms never evaluate — send synthetic traffic during the bake.
- **Blue/Green ECS `deployment controller mismatch`** → service was created with `ECS` controller; must set `deploymentController: { type: CODE_DEPLOY }` at service creation (cannot change later).
