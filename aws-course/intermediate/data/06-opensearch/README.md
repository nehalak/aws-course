# 06 — OpenSearch

## Concept
Managed Elasticsearch/OpenSearch. Search, logs, dashboards (Kibana-fork).

## Exercises
1. **OpenSearch domain**: `t3.small.search`, 1 node, VPC, fine-grained access control with master user.
2. **Ingest**: index 1k sample docs via `_bulk` API from a bastion.
3. **Queries**: match, term, aggregations (terms agg, date histogram).
4. **Dashboards**: create visualization over sample data.
5. **OpenSearch Serverless**: create vector collection. Ingest embeddings. k-NN query.
6. **Fluent Bit → OpenSearch**: ship container logs from ECS/EKS.
7. **Index lifecycle (ISM)**: hot → warm → cold → delete policy.

## Verification
- Dashboards UI renders your data.
- k-NN returns nearest neighbors.

## Gotchas
- Cluster sizing: undersized = red status. Over = money pit.
- Blue/green upgrades take hours on large domains.
- Fine-grained auth + IAM + master user = confusing matrix.

## Cleanup
```bash
cdk destroy
```
