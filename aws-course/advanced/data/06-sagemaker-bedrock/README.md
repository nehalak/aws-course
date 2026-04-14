# 06 — SageMaker & Bedrock (ML + GenAI)

## Concept
SageMaker = full ML lifecycle (train, tune, deploy). Bedrock = managed foundation models (Claude, Llama, etc).

## Exercises
1. **SageMaker Studio**: open a notebook; train sklearn on built-in dataset; deploy endpoint.
2. **Training job**: XGBoost on S3 CSV; HPO (hyperparameter optimization) with 10 trials.
3. **Model Registry + Pipelines**: pipeline with `processing → train → eval → register → deploy` stages.
4. **Real-time endpoint vs serverless**: deploy both; load test; compare cold start.
5. **Bedrock basics**: invoke `anthropic.claude-sonnet` via SDK. Stream response.
6. **RAG pipeline**: ingest docs into OpenSearch Serverless vector collection. Bedrock Knowledge Base ties it together. Query → retrieves → synthesizes answer.
7. **Bedrock Agents**: create agent with action groups (Lambda-backed tools). Test multi-step reasoning.
8. **Guardrails**: attach content filters + denied topics to Bedrock invocations.

## Verification
- Endpoint serves prediction via `sagemaker-runtime invoke-endpoint`.
- RAG agent cites retrieved docs.

## Gotchas
- SageMaker endpoints bill per hour — delete when done.
- Bedrock model access requires opt-in per region.
- Vector collection minimum size = cost floor.

## Cleanup
```bash
cdk destroy
# delete SageMaker endpoints manually
```
