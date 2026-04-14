# 03 — Streaming Architectures

## Concept
Kinesis + Managed Service for Flink (MSF) + Lambda consumers. Exactly-once via checkpointing.

## Exercises
1. **Tumbling window aggregation**: Flink app counting events per minute per userId; sink to DynamoDB.
2. **Sliding window**: 5-min window, sliding every 1 min. Compare.
3. **Session window**: group events by inactivity gap (30 min).
4. **Exactly-once pipeline**: Kinesis → Flink (checkpoints every 60s to S3) → idempotent DynamoDB sink.
5. **Late-arriving data**: watermarks + allowed lateness.
6. **Backpressure**: inject slow sink; observe Flink metric `backPressureTimeMs`.
7. **Compare to Lambda consumer** with `ParallelizationFactor` + checkpoints — write `trade-offs.md`.

## Verification
- Kill Flink during processing; restart; no duplicates or gaps in DynamoDB.

## Gotchas
- Flink state size grows — configure RocksDB + S3 checkpoints.
- MSF pricing per KPU-hour.

## Cleanup
```bash
cdk destroy
```
