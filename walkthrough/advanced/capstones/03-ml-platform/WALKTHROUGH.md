# Walkthrough — Capstone 03 ML Platform

## About this capstone
You will build an end-to-end ML platform that takes raw event data in S3, engineers features into SageMaker Feature Store, runs a SageMaker Pipeline to preprocess / train / evaluate / conditionally register a model, deploys two endpoints behind an ALB for A/B testing, monitors them for data and quality drift, and wraps the whole thing with a Bedrock-powered RAG assistant that answers "why did recall drop last week?" against your CloudWatch metrics and model-card docs. This is the capstone that forces you to think like an ML platform engineer, not just a model trainer.

**Why it matters:** Every company trying to ship ML has this same stack; the ones that skip pieces (no registry, no drift monitoring, no A/B) end up with models that silently rot in production. Data drift is not hypothetical — customer behavior changed in March 2020, again in late 2022, and again after every major model launch; if your monitoring is absent you find out via a support ticket.

**Prerequisites:**
- `intermediate/s3` — prefixes, lifecycle, Parquet.
- `intermediate/glue` — crawlers, ETL jobs.
- `intermediate/sagemaker` — training jobs, endpoints.
- `intermediate/bedrock` — Knowledge Bases, agents.
- `advanced/cicd` — CodePipeline, CodeBuild.

## Estimated cost
- SageMaker Studio domain (idle): $0 if you stop all apps; **apps cost** $0.05–$1.25/hour each. Forget to stop → $30+/weekend.
- SageMaker training job (ml.m5.large, 30 min): ~$0.07 per run.
- SageMaker endpoint (ml.m5.large): **$0.13/hour = ~$95/month** — **two endpoints = ~$190/month** left running.
- SageMaker Model Monitor: ml.m5.xlarge $0.23/hr when the job runs (hourly schedule = $165/mo — use daily).
- SageMaker Feature Store online: $1.25 / 1M reads, $1.25 / 1M writes; offline = S3 cost.
- Glue ETL: $0.44/DPU-hour.
- ALB: $16/month + LCU.
- Bedrock (Claude): ~$3 / 1M input tokens (varies by model).
- S3 + CloudWatch: ~$10/mo.
- Total for this capstone: **~$220–350/month if endpoints + monitor stay on**. **WARN:** this is the most expensive capstone in the course. Stop endpoints between sessions with `--endpoint-name X --endpoint-config-name Y` deletions, not just stack suspension. **Destroy after each session.**

---

## Architecture

```
   S3 raw events  --->  Glue ETL  --->  S3 curated  --->  Feature Store (offline+online)
                                                             |
                                                             v
                                 +--------- SageMaker Pipeline ---------+
                                 | preprocess -> train -> evaluate      |
                                 | -> conditional RegisterModel         |
                                 +--------------------+-----------------+
                                                      |
                                                      v
                                     Model Registry (staging/prod alias)
                                                      |
                                   +------------------+------------------+
                                   v                                     v
                            Endpoint A (90%)                      Endpoint B (10%)
                                   \                                     /
                                    \-------- ALB (weighted TG) --------/
                                                      |
                                        Model Monitor -> CloudWatch alarms
                                                      |
                                Bedrock KB (metrics + docs) -> RAG chat
```

## Step 1: CDK + SageMaker pipeline layout
> **Why:** ML platforms have two codebases: the CDK infra (endpoints, buckets, IAM) and the pipeline definition (Python). Keep them separate repos or at minimum separate top-level folders; the pipeline evolves on a different cadence.

```
ml-platform/
├── cdk/
│   ├── bin/app.ts
│   └── lib/
│       ├── data-stack.ts        # S3 lake, Glue, Feature Store
│       ├── pipeline-stack.ts    # SageMaker Pipeline role, schedule
│       ├── serving-stack.ts     # 2 endpoints + ALB + target groups
│       ├── monitoring-stack.ts  # Model Monitor, alarms
│       ├── bedrock-stack.ts     # KB + agent
│       └── cicd-stack.ts        # CodePipeline
├── pipelines/
│   ├── definition.py            # SageMaker Pipeline DSL
│   └── steps/{preprocess,train,evaluate}.py
├── models/
│   └── card.md                  # ingested by Bedrock KB
└── dbt/                         # optional SQL features
```

```bash
# CDK side (from cdk/)
npm init -y
npm i -D aws-cdk-lib constructs typescript esbuild @types/aws-lambda
npm i @cdklabs/generative-ai-cdk-constructs   # Step 8 (Bedrock KB/Agent L2s)
npx cdk init app --language typescript

# Pipeline side (from pipelines/)
pip install sagemaker boto3
```

## Step 2: Data lake + Feature Store
> **Why:** Feature Store solves the training/serving skew problem: the same feature definition is used to materialize offline (for training) and online (low-latency inference). If training uses `7d_avg_clicks` computed in pandas and serving recomputes it in a different Lambda, your metrics are lies.

```typescript
// lib/data-stack.ts (excerpt)
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as glue from 'aws-cdk-lib/aws-glue';
import * as sm from 'aws-cdk-lib/aws-sagemaker';
import * as iam from 'aws-cdk-lib/aws-iam';

const fsRole = new iam.Role(this, 'FsRole', {
  assumedBy: new iam.ServicePrincipal('sagemaker.amazonaws.com'),
  managedPolicies: [iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSageMakerFeatureStoreAccess')],
});

const raw     = new s3.Bucket(this, 'Raw',     { versioned: true });
const curated = new s3.Bucket(this, 'Curated', { versioned: true });
const fsOffline = new s3.Bucket(this, 'FsOffline');

new sm.CfnFeatureGroup(this, 'UserFeatures', {
  featureGroupName: 'user-features',
  recordIdentifierFeatureName: 'user_id',
  eventTimeFeatureName: 'event_time',
  featureDefinitions: [
    { featureName: 'user_id',    featureType: 'String'  },
    { featureName: 'event_time', featureType: 'Fractional' },
    { featureName: 'clicks_7d',  featureType: 'Integral' },
    { featureName: 'spend_30d',  featureType: 'Fractional' },
  ],
  onlineStoreConfig: { enableOnlineStore: true },
  offlineStoreConfig: {
    s3StorageConfig: { s3Uri: fsOffline.s3UrlForObject() },
    tableFormat: 'Iceberg',
  },
  roleArn: fsRole.roleArn,
});
```

## Step 3: SageMaker Pipeline — DAG in Python
> **Why:** The pipeline is the spine. `ConditionStep` + `RegisterModel` is the critical pattern: register the model in the registry **only if** the evaluation metric clears the bar. This is how you stop a worse model from reaching prod.

```python
# pipelines/definition.py
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep, TrainingStep
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
from sagemaker.workflow.functions import JsonGet
from sagemaker.workflow.model_step import ModelStep

preprocess = ProcessingStep(name='Preprocess', processor=sk_processor, code='steps/preprocess.py',
                            inputs=[...], outputs=[...])
train      = TrainingStep(name='Train', estimator=xgb_estimator, inputs={'train': preprocess.properties.ProcessingOutputConfig.Outputs['train'].S3Output.S3Uri})
evaluate   = ProcessingStep(name='Evaluate', processor=sk_processor, code='steps/evaluate.py',
                            inputs=[train.properties.ModelArtifacts.S3ModelArtifacts],
                            property_files=[eval_report])

good_enough = ConditionGreaterThanOrEqualTo(
    left=JsonGet(step_name=evaluate.name, property_file=eval_report, json_path='metrics.auc.value'),
    right=0.80,
)

register = ModelStep(name='Register', step_args=model.register(
    content_types=['text/csv'], response_types=['text/csv'],
    inference_instances=['ml.m5.large'], transform_instances=['ml.m5.large'],
    model_package_group_name='recommender', approval_status='PendingManualApproval',
))

cond = ConditionStep(name='CheckMetric', conditions=[good_enough], if_steps=[register], else_steps=[])
Pipeline(name='recommender-pipeline', steps=[preprocess, train, evaluate, cond]).upsert(role_arn=role)
```

## Step 4: Model Registry with staging/prod aliases
> **Why:** A registry is a versioned catalog with approval gates. Your CI only promotes `staging` → `production` after canary metrics look good; rollback is re-pointing the alias, not redeploying.

The `ModelStep` above writes to the `recommender` package group. Approval flips a version to `Approved`, which the serving stack uses to pick the active version.

## Step 5: Serving — two endpoints behind ALB (A/B)
> **Why:** Built-in SageMaker production variants give weighted traffic splits **inside one endpoint**, but that couples the models' lifecycles. Two independent endpoints behind an ALB weighted target group lets you redeploy model B without touching A and gives you per-endpoint CloudWatch metrics.

```typescript
// lib/serving-stack.ts (excerpt)
const endpointA = new sm.CfnEndpoint(this, 'EndpointA', { endpointConfigName: cfgA.attrEndpointConfigName });
const endpointB = new sm.CfnEndpoint(this, 'EndpointB', { endpointConfigName: cfgB.attrEndpointConfigName });

const alb = new elbv2.ApplicationLoadBalancer(this, 'Alb', { vpc, internetFacing: false });
const listener = alb.addListener('Http', { port: 80 });

// fronting Lambdas that call the SM runtime and return JSON; ALB can't target SM directly.
const fnA = new NodejsFunction(this, 'FnA', { environment: { ENDPOINT: endpointA.attrEndpointName }, entry: 'serving/invoke.ts' });
const fnB = new NodejsFunction(this, 'FnB', { environment: { ENDPOINT: endpointB.attrEndpointName }, entry: 'serving/invoke.ts' });

listener.addTargetGroups('Split', {
  targetGroups: [
    new elbv2.ApplicationTargetGroup(this, 'TgA', { targets: [new LambdaTarget(fnA)], weight: 90 }),
    new elbv2.ApplicationTargetGroup(this, 'TgB', { targets: [new LambdaTarget(fnB)], weight: 10 }),
  ],
});
```

## Step 6: Model Monitor — drift + quality baselines
> **Why:** Baselines are computed once from training data; each monitoring run compares the recent inference-capture data to the baseline and raises CloudWatch alarms on drift. Without this, the only signal of a bad model is customer churn.

```python
from sagemaker.model_monitor import DefaultModelMonitor, DataCaptureConfig
from sagemaker.model_monitor.dataset_format import DatasetFormat
from sagemaker.model_monitor import CronExpressionGenerator
capture = DataCaptureConfig(enable_capture=True, sampling_percentage=100, destination_s3_uri=f's3://{capture_bucket}/')

# baseline
monitor = DefaultModelMonitor(role=role, instance_count=1, instance_type='ml.m5.xlarge')
monitor.suggest_baseline(baseline_dataset=f's3://{curated}/train.csv', dataset_format=DatasetFormat.csv(header=True))

# schedule
monitor.create_monitoring_schedule(
    endpoint_input=endpoint_name, output_s3_uri=f's3://{mon_bucket}/reports',
    statistics=monitor.baseline_statistics(), constraints=monitor.suggested_constraints(),
    schedule_cron_expression=CronExpressionGenerator.daily(),  # NOT hourly unless you like bills
    enable_cloudwatch_metrics=True,
)
```

CloudWatch alarm on `feature_baseline_drift_clicks_7d > 0.2` → SNS to the ML oncall.

## Step 7: CI/CD — commit to retrain
> **Why:** ML at velocity requires that `git push` on the model repo triggers a retrain, a full evaluation, and — if the new model beats the champion on a holdout set — auto-promotes to staging. Manual SageMaker Studio clicks do not scale past two data scientists.

```typescript
// lib/cicd-stack.ts (CodePipeline stages)
pipeline.addStage({ stageName: 'Source', actions: [new CodeStarConnectionsSourceAction({ ... })] });
pipeline.addStage({ stageName: 'TrainEval', actions: [new CodeBuildAction({ project: trainProj })] });
pipeline.addStage({ stageName: 'RegisterIfBetter', actions: [new LambdaInvokeAction({ lambda: compareFn })] });
pipeline.addStage({ stageName: 'DeployStaging', actions: [new LambdaInvokeAction({ lambda: deployStagingFn })] });
pipeline.addStage({ stageName: 'ManualProd', actions: [new ManualApprovalAction()] });
pipeline.addStage({ stageName: 'DeployProd', actions: [new LambdaInvokeAction({ lambda: promoteFn })] });
```

## Step 8: Bedrock RAG over model metrics + docs
> **Why:** Stakeholders ask the same five questions over and over ("what's the recall right now? when did it last drop? what does this feature mean?"). A Bedrock Knowledge Base over CloudWatch exports + your `models/card.md` answers them without pinging the ML team.

```typescript
// lib/bedrock-stack.ts (excerpt)
import { bedrock } from '@cdklabs/generative-ai-cdk-constructs';

const kb = new bedrock.KnowledgeBase(this, 'Kb', {
  embeddingsModel: bedrock.BedrockFoundationModel.TITAN_EMBED_TEXT_V2_1024,
  instruction: 'Answer questions about model metrics and documentation.',
});

new bedrock.S3DataSource(this, 'DocsDs', {
  bucket: docsBucket,  // contains models/card.md + daily metric snapshots
  knowledgeBase: kb,
  dataSourceName: 'docs',
});

const agent = new bedrock.Agent(this, 'Agent', {
  foundationModel: bedrock.BedrockFoundationModel.ANTHROPIC_CLAUDE_SONNET_V2,
  instruction: 'You are an ML platform assistant. Cite sources.',
  knowledgeBases: [kb],
});
```

A nightly Lambda exports CloudWatch metrics (`auc`, `latency_p95`, drift scores) to the bucket and triggers KB `StartIngestionJob`.

## Step 9: Deploy
> **Why:** Data first, pipeline second (it needs the buckets), then serving, then monitoring, then Bedrock (which indexes content produced by monitoring).

```bash
npx cdk deploy DataStack
cd pipelines && python definition.py  # upserts the SageMaker pipeline
npx cdk deploy PipelineStack ServingStack MonitoringStack BedrockStack CicdStack
```

## Step 10: Run the pipeline and deploy
> **Why:** First end-to-end run proves every wire is connected.

```bash
aws sagemaker start-pipeline-execution --pipeline-name recommender-pipeline
# wait ~15 min
aws sagemaker list-model-packages --model-package-group-name recommender
# approve latest
aws sagemaker update-model-package --model-package-arn $ARN --model-approval-status Approved
# CI/CD auto-deploys to staging, manual approval pushes to prod endpoint
```

## Step 11: Drift demo
> **Why:** If the alarm does not fire on obvious drift, it will not fire on subtle drift. Inject skewed data and watch the alarm light up.

```bash
# fire 5000 inferences with skewed feature distribution
python scripts/inject-drift.py --endpoint $EP_A --skew 3.0
# wait for next monitor run (or trigger ad-hoc via StartMonitoringSchedule)
aws cloudwatch describe-alarms --alarm-names model-drift-clicks_7d
# expect State=ALARM
```

## Step 12: Ask the Bedrock agent
> **Why:** End-user proof that the loop is closed.

```bash
aws bedrock-agent-runtime invoke-agent --agent-id $A --agent-alias-id $AL \
  --session-id s1 --input-text 'Did recall drop this week and if so what feature is driving it?'
```

## Verification / success criteria
- Pipeline re-runs on every commit; `ConditionStep` blocks registration when AUC < 0.80.
- A/B: `aws sagemaker-runtime invoke-endpoint` calls go 90/10 to A/B (observe CloudWatch `Invocations` per endpoint).
- Drift: injected skew → `model-drift-*` alarm transitions to `ALARM` within 24 hours.
- Bedrock agent: returns a sourced answer citing `models/card.md` and a specific CloudWatch metric file.
- Per-training-job cost visible in Cost Explorer grouped by tag `project=recommender`.

## Cleanup
```bash
# endpoints continue to bill until deleted
aws sagemaker delete-endpoint --endpoint-name $EP_A
aws sagemaker delete-endpoint --endpoint-name $EP_B
aws sagemaker delete-endpoint-config --endpoint-config-name $CFG_A
aws sagemaker delete-endpoint-config --endpoint-config-name $CFG_B

# stop monitor schedule
aws sagemaker stop-monitoring-schedule --monitoring-schedule-name recommender-schedule
aws sagemaker delete-monitoring-schedule --monitoring-schedule-name recommender-schedule

# stop Studio apps
aws sagemaker list-apps | jq -r '.Apps[] | select(.Status=="InService") | .AppName'  # then delete each

npx cdk destroy --all
```

## Common Errors
- **`ResourceLimitExceeded` on training** → your account default for `ml.m5.large` is low; raise via Service Quotas console.
- **Pipeline stuck in `Executing`** → a ProcessingStep container exited 0 but wrote no output; SageMaker waits forever — always set `outputs` paths and write even on empty.
- **`ModelError: unsupported instance type` on endpoint** → image URI does not match region; use `sagemaker.image_uris.retrieve`.
- **Drift alarm never fires** → captured data sampling at 100% is noisy but right; with <1% you will not get enough samples for a monitor run.
- **Bedrock KB returns "I don't know"** → ingestion failed silently; `aws bedrock-agent get-ingestion-job` and check the failure reason (usually S3 permission).
- **Endpoint invocation 403** → ALB Lambda target role cannot `sagemaker:InvokeEndpoint`; the ALB role is NOT the Lambda's role.
- **Feature Store online read returns null** → the record existed offline but was never put online; `PutRecord` against the online store explicitly.
- **$300 surprise bill** → two ml.m5.xlarge endpoints left running for a month. Always `delete-endpoint` not just `cdk destroy` — destroy removes the CFN resource but if the endpoint is out of stack it keeps billing.
