# Walkthrough — 06 SageMaker & Bedrock (ML + GenAI)

## About this service
**SageMaker** is the full ML lifecycle platform: Studio notebooks, training jobs, hyperparameter optimization (HPO), model registry, pipelines, and endpoints (real-time or serverless). **Bedrock** is managed foundation models behind one API — Claude, Llama, Mistral, Titan — plus higher-level constructs: **Knowledge Bases** (RAG), **Agents** (tool use), **Guardrails** (content safety).

**Why it matters:** SageMaker lets you own a custom model; Bedrock lets you build on someone else's. In 2026 most teams need both: a fine-tuned classifier from SageMaker + a Bedrock Claude front-end for conversational UX.
**When to use:** SageMaker — custom models, proprietary data, need training control. Bedrock — LLM apps, RAG, multi-step agents, when "good enough" LLM beats bespoke.
**When NOT to use:** SageMaker for a $20/mo scikit-learn API — use Lambda. Bedrock for high-QPS cheap inference of tiny models — use SageMaker Serverless or an on-prem ONNX model.

## Estimated cost
- **SageMaker Studio notebook (ml.t3.medium):** $0.05/hr — **FLAG: auto-stop or it runs 24/7**
- **SageMaker training job (ml.m5.xlarge):** $0.23/hr — only runs during job
- **SageMaker real-time endpoint (ml.m5.large):** **FLAG — $0.115/hr × 24 × 30 = ~$83/mo; endpoints bill continuously whether used or not**
- **SageMaker Serverless Inference:** $0.20/M requests + $0.00002/GB-s — cheaper for bursty
- **Bedrock Claude 3.5 Sonnet:** $3/M input tokens, $15/M output tokens
- **Bedrock Claude 3.5 Haiku:** $0.80/M input, $4/M output
- **Bedrock Knowledge Base + OpenSearch Serverless:** **FLAG — OSS minimum 2 OCUs indexing + 2 OCUs search = $0.24/OCU-hr × 4 = ~$700/mo floor.** Use Aurora pgvector instead for low volume.
- **Bedrock Guardrails:** $0.75/1k text units
- Total for this lesson (short endpoint life + ~1 M Bedrock tokens + small vector collection for a few hours): **~$20–40** if disciplined; **several hundred** if endpoints or OSS are left running.

---

## Step 1: SageMaker Studio domain
> **Why:** Studio is the IDE. Domain + user profile → Jupyter with ready-made conda envs. Enable auto-shutdown for idle notebooks.

```typescript
import * as sm from 'aws-cdk-lib/aws-sagemaker';

const domain = new sm.CfnDomain(this, 'Studio', {
  domainName: 'ml-lab',
  authMode: 'IAM',
  defaultUserSettings: { executionRole: studioRole.roleArn },
  subnetIds: vpc.privateSubnets.map(s => s.subnetId),
  vpcId: vpc.vpcId,
});

new sm.CfnUserProfile(this, 'User', {
  domainId: domain.attrDomainId,
  userProfileName: 'data-sci',
  userSettings: { executionRole: studioRole.roleArn },
});
```

## Step 2: Train sklearn and deploy endpoint
> **Why:** Smallest possible end-to-end. Proves Studio → training → endpoint works before tackling bigger models.

```python
# In a Studio notebook
import sagemaker
from sagemaker.sklearn.estimator import SKLearn

sess = sagemaker.Session()
role = sagemaker.get_execution_role()

est = SKLearn(
    entry_point='train.py',
    role=role,
    instance_type='ml.m5.xlarge',
    framework_version='1.2-1',
    py_version='py3',
    hyperparameters={'n_estimators': 100},
)
est.fit({'train': 's3://my-bucket/iris/train/'})

predictor = est.deploy(
    initial_instance_count=1,
    instance_type='ml.m5.large',
    endpoint_name='iris-ep',
)

print(predictor.predict([[5.1, 3.5, 1.4, 0.2]]))
# REMEMBER: predictor.delete_endpoint() or bill accrues 24/7
```

`train.py`:
```python
import argparse, os, joblib, pandas as pd
from sklearn.ensemble import RandomForestClassifier

if __name__ == '__main__':
    p = argparse.ArgumentParser()
    p.add_argument('--n_estimators', type=int, default=100)
    p.add_argument('--train', type=str, default=os.environ['SM_CHANNEL_TRAIN'])
    p.add_argument('--model-dir', type=str, default=os.environ['SM_MODEL_DIR'])
    args = p.parse_args()

    df = pd.read_csv(f'{args.train}/iris.csv')
    X, y = df.iloc[:, :-1], df.iloc[:, -1]
    clf = RandomForestClassifier(n_estimators=args.n_estimators).fit(X, y)
    joblib.dump(clf, f'{args.model_dir}/model.joblib')
```

## Step 3: HPO with 10 trials (XGBoost)
> **Why:** Automated search beats hand-tuning. SageMaker Bayesian/random search runs N jobs in parallel and picks the best.

```python
from sagemaker.xgboost import XGBoost
from sagemaker.tuner import HyperparameterTuner, ContinuousParameter, IntegerParameter

xgb = XGBoost(
    entry_point='xgb_train.py', role=role,
    instance_type='ml.m5.xlarge', framework_version='1.7-1',
)

tuner = HyperparameterTuner(
    estimator=xgb,
    objective_metric_name='validation:auc',
    objective_type='Maximize',
    hyperparameter_ranges={
        'eta':       ContinuousParameter(0.01, 0.3),
        'max_depth': IntegerParameter(3, 10),
    },
    max_jobs=10,
    max_parallel_jobs=3,
)
tuner.fit({'train': 's3://.../train/', 'validation': 's3://.../val/'})
print(tuner.best_estimator().hyperparameters())
```

## Step 4: Pipeline — process → train → eval → register → deploy
> **Why:** Repeatable, auditable ML. Each run logged in SageMaker Lineage; models gated by accuracy threshold.

```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep, TrainingStep
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.conditions import ConditionGreaterThan
from sagemaker.workflow.step_collections import RegisterModel

# ... define processor, estimator, processing_step, training_step, eval_step ...

cond = ConditionGreaterThan(
    left=JsonGet(step_name=eval_step.name, property_file=eval_report, json_path='accuracy'),
    right=0.85,
)

register = RegisterModel(
    name='RegisterModel',
    estimator=estimator,
    model_package_group_name='iris-models',
    approval_status='PendingManualApproval',
)

pipeline = Pipeline(
    name='iris-pipeline',
    steps=[processing_step, training_step, eval_step,
           ConditionStep(name='AccGate', conditions=[cond], if_steps=[register], else_steps=[])],
)
pipeline.upsert(role_arn=role)
pipeline.start()
```

## Step 5: Real-time vs serverless endpoint
> **Why:** Real-time = always-on, steady latency. Serverless = pay-per-request, cold starts. Choose based on QPS and latency SLO.

```python
from sagemaker.serverless import ServerlessInferenceConfig

serverless_cfg = ServerlessInferenceConfig(memory_size_in_mb=4096, max_concurrency=10)
srv_pred = est.deploy(serverless_inference_config=serverless_cfg, endpoint_name='iris-sv')

# Load test both
import time
for ep in ['iris-ep', 'iris-sv']:
    t = time.time(); runtime.invoke_endpoint(EndpointName=ep, Body=...)
    print(ep, 'latency', time.time()-t)
```

Result: real-time ~20 ms warm. Serverless: ~2 s cold, then ~50 ms warm.

## Step 6: Bedrock invoke + streaming
> **Why:** Streaming lowers perceived latency (UX) and lets you stop early if the answer veers off.

```python
import boto3, json
br = boto3.client('bedrock-runtime', region_name='us-east-1')

stream = br.invoke_model_with_response_stream(
    modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
    body=json.dumps({
        'anthropic_version': 'bedrock-2023-05-31',
        'max_tokens': 512,
        'messages': [{'role': 'user', 'content': 'Explain exactly-once in Flink in 3 bullets.'}],
    }),
    contentType='application/json',
)
for event in stream['body']:
    chunk = json.loads(event['chunk']['bytes'])
    if chunk.get('type') == 'content_block_delta':
        print(chunk['delta'].get('text', ''), end='', flush=True)
```

## Step 7: RAG via Bedrock Knowledge Base + OpenSearch Serverless
> **Why:** Offload the hard parts (chunking, embedding, retrieval) to a managed KB. Agents cite sources. **Cost-flag: OpenSearch Serverless has a 2-OCU minimum; for low volume, configure Aurora pgvector instead.**

```typescript
import * as bedrock from 'aws-cdk-lib/aws-bedrock';
import * as oss from 'aws-cdk-lib/aws-opensearchserverless';

const collection = new oss.CfnCollection(this, 'VecCol', {
  name: 'kb-vectors', type: 'VECTORSEARCH',
});

new bedrock.CfnKnowledgeBase(this, 'Kb', {
  name: 'docs-kb',
  roleArn: kbRole.roleArn,
  knowledgeBaseConfiguration: {
    type: 'VECTOR',
    vectorKnowledgeBaseConfiguration: {
      embeddingModelArn: 'arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0',
    },
  },
  storageConfiguration: {
    type: 'OPENSEARCH_SERVERLESS',
    opensearchServerlessConfiguration: {
      collectionArn: collection.attrArn,
      vectorIndexName: 'kb-index',
      fieldMapping: { vectorField: 'vector', textField: 'text', metadataField: 'metadata' },
    },
  },
});
```

Ingest: upload docs to S3, create a data source pointing at the bucket, `StartIngestionJob`. Then:

```python
resp = boto3.client('bedrock-agent-runtime').retrieve_and_generate(
    input={'text': 'What is Flink exactly-once?'},
    retrieveAndGenerateConfiguration={
        'type': 'KNOWLEDGE_BASE',
        'knowledgeBaseConfiguration': {
            'knowledgeBaseId': KB_ID,
            'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0',
        },
    },
)
print(resp['output']['text'])
for c in resp['citations']:
    for ref in c['retrievedReferences']:
        print(' source:', ref['location'])
```

## Step 8: Bedrock Agent with action groups
> **Why:** Agents orchestrate tools. The LLM decides when to call your Lambda "get_weather" or "lookup_order" — you get multi-step reasoning with real-world actions.

1. Create Lambda `order-lookup` that accepts `{ orderId }` → returns order JSON.
2. Write OpenAPI spec describing the tool.
3. Create agent → action group → attach Lambda + OpenAPI.
4. Test: "What's the status of order 42?" → agent calls Lambda → summarizes result.

```bash
aws bedrock-agent create-agent \
  --agent-name order-bot \
  --foundation-model anthropic.claude-3-5-sonnet-20241022-v2:0 \
  --instruction "You are a support agent. Use tools to look up orders."
```

## Step 9: Guardrails
> **Why:** Block prompt injection, PII leakage, off-topic answers before they reach the user. Cheaper than filtering post-hoc.

```bash
aws bedrock create-guardrail \
  --name support-guard \
  --content-policy-config '{"filtersConfig":[
     {"type":"SEXUAL","inputStrength":"HIGH","outputStrength":"HIGH"},
     {"type":"VIOLENCE","inputStrength":"HIGH","outputStrength":"HIGH"},
     {"type":"PROMPT_ATTACK","inputStrength":"HIGH","outputStrength":"NONE"}]}' \
  --topic-policy-config '{"topicsConfig":[
     {"name":"Competitors","type":"DENY","definition":"Do not discuss competitor products","examples":["What about Azure?"]}]}' \
  --word-policy-config '{"managedWordListsConfig":[{"type":"PROFANITY"}]}'
```

Pass `guardrailIdentifier` + `guardrailVersion` to every `invoke_model` call.

## Cleanup
```bash
# ENDPOINTS FIRST — these are the bleeding-cost item
aws sagemaker delete-endpoint --endpoint-name iris-ep
aws sagemaker delete-endpoint --endpoint-name iris-sv
aws sagemaker delete-endpoint-config --endpoint-config-name iris-ep
aws sagemaker delete-endpoint-config --endpoint-config-name iris-sv

# Shut down Studio apps
aws sagemaker list-apps | jq -r '.Apps[]|select(.Status=="InService")|"\(.UserProfileName) \(.AppType) \(.AppName)"' \
  | while read u t n; do aws sagemaker delete-app --domain-id $DOMAIN --user-profile-name $u --app-type $t --app-name $n; done

# Bedrock KB + OSS (expensive floor)
aws bedrock-agent delete-knowledge-base --knowledge-base-id $KB
aws opensearchserverless delete-collection --id $COL

cdk destroy
```

## Common Errors
- **Bedrock: `AccessDeniedException` on model invoke** → model access not enabled for this account/region. Console → Bedrock → Model access → request.
- **Endpoint `InService` but bill huge** → you forgot to delete. Always script `delete-endpoint` in teardown.
- **OpenSearch Serverless costs $700/mo floor** → swap to Aurora pgvector if volume is low.
- **RAG returns "I don't know"** → ingestion job didn't complete, or query embedding model mismatches index. Re-run `StartIngestionJob` and verify index count.
- **Agent calls tool with wrong args** → OpenAPI spec vague. Tighten parameter descriptions + `required` array.
- **Guardrail blocks legitimate input** → strength too high; drop from HIGH to MEDIUM on specific filter.
- **HPO jobs fail with `ResourceLimitExceeded`** → account limit on ml.m5.xlarge. Request service-quota increase.
- **SageMaker Studio notebook kernel dies** → instance too small. Bump ml.t3.medium → ml.m5.large.
