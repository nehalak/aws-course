# 05 — Kinesis & MSK

## Concept
Kinesis Data Streams = managed, shard-based. Firehose = managed delivery to S3/Redshift/OpenSearch. MSK = managed Kafka.

## Exercises
1. **Kinesis Data Stream**: 2 shards. Producer (local script) uses KPL / boto3 to put 10k records. Lambda consumer with enhanced fan-out.
2. **Firehose**: stream → S3 in Parquet format with dynamic partitioning by `eventType`. Query with Athena.
3. **Firehose transformation**: Lambda transforms records in-flight before S3.
4. **Kinesis Data Streams → Analytics (Flink)**: run a windowed aggregation (count per minute).
5. **MSK Serverless**: provision; produce/consume with `kafka-console-producer` from EC2.
6. **MSK Connect**: S3 sink connector dumps topic to S3.

## Verification
- Athena table on Firehose output returns rows.
- Flink aggregates appear in output stream.

## Gotchas
- Shard limits: 1MB/s or 1000 records/sec write.
- MSK provisioned is expensive — prefer Serverless for learning.
- Enhanced fan-out = $$ per consumer.

## Cleanup
```bash
cdk destroy
```
