# 01 — DynamoDB Advanced

## Concept
Single-table design, GSI/LSI, streams, TTL, transactions, conditional writes.

## Exercises
1. **Single-table design**: one table for Users + Orders + Products. Use PK/SK overloading: `PK=USER#u1 SK=USER#u1`, `PK=USER#u1 SK=ORDER#o1`. Implement 5 access patterns.
2. **GSI**: add GSI with `GSI1PK/GSI1SK` for cross-entity queries ("all orders by status").
3. **Streams + Lambda**: enable stream; Lambda processes `MODIFY/INSERT/REMOVE`. Replicate to S3 as event log.
4. **TTL**: set `expiresAt` on session items. Watch items vanish.
5. **Transactions**: `TransactWriteItems` atomic update across 3 items (debit user, credit user, insert audit row). Fail one condition → all roll back.
6. **Conditional writes**: optimistic locking using `attribute_not_exists(PK)` or `version = :v`.
7. **DynamoDB Streams → Kinesis**: fan out to analytics.

## Verification
- 5 access patterns executable with only PK/SK + GSI.
- Transaction fails atomically on conditional violation.

## Gotchas
- Hot partitions: max 1000 WCU / 3000 RCU per partition.
- GSI propagation is eventually consistent.
- TTL delete can lag 48h.

## Cleanup
```bash
cdk destroy
```
