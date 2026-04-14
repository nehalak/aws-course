# 04 — DynamoDB Internals

## Concept
Adaptive capacity, hot partitions, DAX, export to S3, PITR, global tables internals.

## Exercises
1. **Force a hot partition**: insert 1M rows with same PK. Observe throttling. Fix via composite key (PK + shard suffix).
2. **Adaptive capacity**: watch CloudWatch metric `ThrottledRequests` drop over minutes as AWS rebalances.
3. **DAX cluster**: in front of table; measure p99 cache vs direct DynamoDB.
4. **PITR**: enable; delete table's items; restore to timestamp 5 min ago.
5. **Export to S3**: full-table export → S3 → Athena query.
6. **Incremental export**: enable streams-based incremental to S3.
7. **CDC to Kinesis**: enable Kinesis stream on table; consume with Lambda.

## Verification
- DAX p99 < 1ms for cached keys.
- PITR restores to exact second.

## Gotchas
- DAX only caches `GetItem`/`Query`/`Scan` — not transactions or writes.
- Hot partition fix often needs data remodel.

## Cleanup
```bash
cdk destroy
```
