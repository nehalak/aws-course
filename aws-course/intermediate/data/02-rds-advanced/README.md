# 02 — RDS Advanced & Aurora

## Concept
Aurora = AWS-built MySQL/Postgres-compatible. Separates storage/compute. Read replicas at storage level.

## Exercises
1. **Aurora Postgres cluster**: 1 writer + 2 readers. Reader endpoint load-balances.
2. **Failover**: `failover-db-cluster`. Writer swap ~30 sec.
3. **Aurora Serverless v2**: min 0.5 ACU max 8. Load test; observe auto-scale.
4. **RDS Proxy**: enable, point app at proxy endpoint. Simulate 1000 concurrent Lambdas — proxy pools connections.
5. **Performance Insights**: enable; run slow query; find top SQL by wait time.
6. **Cross-region read replica**: create replica in another region. Promote it.
7. **Blue/Green deployments**: test major version upgrade Blue/Green with cutover.

## Verification
- Reader endpoint returns different AZs via DNS.
- Serverless scales ACU under load.

## Gotchas
- Aurora storage = $0.10/GB but grows to peak never shrinks without restore.
- Serverless v2 min 0.5 ACU = $43/mo idle.
- Proxy requires Secrets Manager creds.

## Cleanup
```bash
cdk destroy
```
