# Walkthrough — 04 DynamoDB Internals

## About this service
Under the hood, **DynamoDB** is a distributed hash+range store where each table is sharded by partition-key hash across **physical partitions** (up to 1000 WCU / 3000 RCU / 10 GB each). **Adaptive capacity** rebalances load over minutes; **DAX** is a write-through cache for microsecond reads; **PITR** is continuous backup; **exports to S3** produce queryable Parquet; **Kinesis / DynamoDB Streams** give CDC.

**Why it matters:** when you hit hot-partition throttling or need sub-ms reads or audit replay, you need to understand the internals, not just the API. These are the advanced capabilities that separate "working prototype" from "billion-request-a-day production".

**When to use:** DAX for read-heavy cache-friendly workloads; PITR always in prod; exports for ad-hoc analytics without hitting table RCUs; CDC for fanout to search, data lakes, cache invalidation.
**When NOT to use:** DAX for write-heavy workloads (no benefit — only reads cached); PITR alone as a backup (keep cross-region backups too); exports for near-real-time analytics (they lag).

## Estimated cost
- **DynamoDB on-demand:** $1.25/M writes, $0.25/M reads, $0.25/GB/mo storage
- **DAX:** **FLAG — smallest node `dax.t3.small` = $0.04/hr × 3 nodes (recommended) = $0.12/hr = ~$88/mo**
- **PITR:** extra $0.20/GB/mo (roughly doubles storage cost)
- **Export to S3:** $0.10/GB exported (one-time per export)
- **Kinesis data stream for CDC:** $0.04/hr on-demand
- Total for this lesson (small table + DAX running a few hours + 1 export + PITR): **~$5–10**

---

## Step 1: Baseline table
> **Why:** Single partition key = single physical partition until table grows. We'll stress this to force throttling, then fix it.

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const hot = new dynamodb.Table(this, 'HotTable', {
  tableName: 'HotTable',
  partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'sk', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PROVISIONED,
  readCapacity: 10,
  writeCapacity: 10,
  pointInTimeRecovery: true,              // enables PITR
  kinesisStream: cdcStream,                // CDC stream (if defined)
  dynamoStream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

## Step 2: Force a hot partition
> **Why:** A fixed PK routes 100% of traffic to one partition. At a 3000-write/sec workload, you'll hit the per-partition 1000 WCU cap and throttle even though the table is provisioned for 10000.

```python
# hot_writer.py
import boto3, time, concurrent.futures
ddb = boto3.client('dynamodb')

def write(i):
    ddb.put_item(
        TableName='HotTable',
        Item={'pk': {'S': 'HOT'}, 'sk': {'S': f'{i}'}, 'data': {'S': 'x' * 1024}}
    )

with concurrent.futures.ThreadPoolExecutor(max_workers=50) as ex:
    for i in range(100_000):
        ex.submit(write, i)
```

Watch CloudWatch: `WriteThrottleEvents > 0`, `ConsumedWriteCapacityUnits` plateaus.

## Step 3: Fix with sharded PK
> **Why:** Append a random `0..N` shard suffix to spread writes across partitions. Tune `N` to match your write rate / 1000 WCU per shard.

```python
# sharded_writer.py
import random
def write(i):
    shard = random.randint(0, 9)  # 10 shards → 10 partitions
    ddb.put_item(
        TableName='HotTable',
        Item={'pk': {'S': f'HOT#{shard}'}, 'sk': {'S': f'{i}'}, 'data': {'S': 'x'*1024}}
    )
```

Read path requires scatter-gather across all 10 shards — trade-off acknowledged.

## Step 4: Observe adaptive capacity rebalancing
> **Why:** AWS silently relocates hot items to isolated partitions over ~10 min. Throttle rate decreases even without code change — but it's not instant and not unlimited.

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name WriteThrottleEvents \
  --dimensions Name=TableName,Value=HotTable \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Sum
```

## Step 5: DAX cluster
> **Why:** DAX is write-through: GetItem/Query against DAX returns from memory in ~300µs vs. 5–10ms direct-to-Dynamo. Scales reads cheaply — but **only for items DAX has cached**, and NOT for transactions/writes.

```typescript
import * as dax from 'aws-cdk-lib/aws-dax';

const subnetGroup = new dax.CfnSubnetGroup(this, 'DaxSubnetGroup', {
  subnetGroupName: 'dax-sg',
  subnetIds: vpc.privateSubnets.map(s => s.subnetId),
});

new dax.CfnCluster(this, 'DaxCluster', {
  clusterName: 'hot-cache',
  nodeType: 'dax.t3.small',
  replicationFactor: 3,
  iamRoleArn: daxRole.roleArn,
  subnetGroupName: subnetGroup.subnetGroupName!,
  securityGroupIds: [daxSg.securityGroupId],
});
```

Python client (must use `amazon-dax-client`, not boto3):
```python
from amazondax import AmazonDaxClient
dax = AmazonDaxClient.resource(endpoint_url='dax://hot-cache.xxxx.dax-clusters.us-east-1.amazonaws.com')
table = dax.Table('HotTable')
table.get_item(Key={'pk': 'u1', 'sk': 'profile'})
```

Load-test and compare p99: DAX hot-read ~500µs; Dynamo direct ~5ms.

## Step 6: PITR restore
> **Why:** Continuous backup to any second in the last 35 days. Critical for oops-I-deleted-everything recovery. Restore creates a NEW table — original untouched.

```bash
# Enable (already done in CDK) — verify:
aws dynamodb describe-continuous-backups --table-name HotTable

# Restore to 5 min ago
aws dynamodb restore-table-to-point-in-time \
  --source-table-name HotTable \
  --target-table-name HotTable-restored \
  --restore-date-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S)
```

## Step 7: Export to S3
> **Why:** Exports run against PITR snapshots — no RCU consumption on the live table. Output is DynamoDB JSON or **Ion** partitioned by day → Athena queryable.

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:$ACCOUNT:table/HotTable \
  --s3-bucket ddb-exports-$ACCOUNT \
  --export-format DYNAMODB_JSON \
  --export-time $(date -u +%s)
```

In Athena:
```sql
CREATE EXTERNAL TABLE ddb_export (
  Item struct<pk:struct<S:string>, sk:struct<S:string>, data:struct<S:string>>
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://ddb-exports-123/AWSDynamoDB/<exportId>/data/';
```

## Step 8: Incremental export
> **Why:** Full exports of a 1 TB table take hours. Incremental exports capture just the changes since the last export — daily analytics pipelines stay fresh.

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:$ACCOUNT:table/HotTable \
  --s3-bucket ddb-exports-$ACCOUNT \
  --incremental-export-specification \
    ExportFromTime=$(date -u -d '1 day ago' +%s),ExportToTime=$(date -u +%s),ExportViewType=NEW_AND_OLD_IMAGES
```

## Step 9: CDC → Kinesis → Lambda
> **Why:** Kinesis data stream CDC is more durable than DynamoDB Streams (24hr default), supports enhanced fan-out, and integrates with Flink/Firehose for downstream ETL.

```typescript
const cdcStream = new kinesis.Stream(this, 'Cdc', { streamMode: kinesis.StreamMode.ON_DEMAND });
// already wired via `kinesisStream: cdcStream` on the table

const consumer = new lambda.Function(this, 'CdcFn', { /* ... */ });
consumer.addEventSource(new eventsources.KinesisEventSource(cdcStream, {
  startingPosition: lambda.StartingPosition.LATEST,
  batchSize: 100,
}));
```

## Cleanup
```bash
# DAX first (expensive)
aws dax delete-cluster --cluster-name hot-cache

# Delete exports bucket contents
aws s3 rm s3://ddb-exports-$ACCOUNT --recursive

cdk destroy
```

## Common Errors
- **`ProvisionedThroughputExceededException` on a cold table** → hot partition. Check CloudWatch `Contributor Insights` for offending keys.
- **DAX returns stale data** → write went directly to Dynamo bypassing DAX. Route all writes through DAX client.
- **PITR restore table is empty** → restore time before PITR was enabled. PITR only covers time since it was turned on.
- **Export job `FAILED` with S3 403** → bucket policy missing `dynamodb.amazonaws.com` principal.
- **CDC stream has duplicates** → that's expected. Dynamo Streams / Kinesis CDC are at-least-once; consumers must be idempotent.
- **Incremental export error `InvalidExportTime`** → ExportFromTime must be within the last 35 days and >=15 min after PITR enablement.
