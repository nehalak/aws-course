# Capstone 03 — ML Platform

## Goal
End-to-end ML platform: data ingest → feature engineering → training → registry → deployment → monitoring.

## Requirements
- S3 data lake + Glue ETL
- SageMaker Feature Store for features
- SageMaker Pipelines: preprocess → train → evaluate → conditional register
- Model Registry with staging/production aliases
- A/B deployment: 2 endpoints behind ALB with traffic split
- Model Monitor: data drift + quality baselines
- Bedrock-backed RAG assistant answering questions about model metrics
- CI/CD: commit to model repo → retrain → auto-deploy if eval metric ≥ baseline

## Deliverables
- CDK + SageMaker pipeline code
- `ml-lifecycle.md` docs
- Drift demo: inject skewed data; observe alarm fire
- Cost report per training job

## Verification
- Pipeline re-runs on data change.
- Drift alarm triggers on skew.

## Gotchas
- SageMaker Studio domain is expensive idle — stop domain.
- Endpoint billing per hour.

## Cleanup
```bash
cdk destroy
# delete endpoints + Studio
```
