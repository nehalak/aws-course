# Walkthrough — 03 Streaming Architectures

## About this service
Real-time streaming on AWS pairs **Kinesis Data Streams** (ordered, durable, shard-based log) with **Managed Service for Apache Flink (MSF)** for stateful stream processing. Flink gives you **tumbling/sliding/session windows**, **watermarks**, **exactly-once checkpoints**, and massive parallelism — way beyond what a Lambda consumer can do. Sinks go to DynamoDB, S3, OpenSearch, another Kinesis stream, etc.

**Why it matters:** fraud detection, real-time leaderboards, clickstream analytics, IoT telemetry rollups — any workload where "batch tomorrow" is too late. Exactly-once + checkpointing means you can crash-restart without losing or duplicating data.
**When to use:** sustained high-throughput streams (>1k events/sec), complex stateful processing (multi-event windows, joins), stringent correctness (exactly-once).
**When NOT to use:** low throughput (<100 events/sec) — just Lambda + DynamoDB. Simple per-event transforms — Kinesis Data Firehose is 10x cheaper.

## Estimated cost
- **Kinesis Data Streams (on-demand):** $0.04/hr per stream + $0.04/GB ingested + $0.04/GB retrieved = ~$30/mo base + data
- **Kinesis provisioned:** $0.015/shard-hr = $10.95/shard/mo
- **Managed Service for Flink:** **FLAG — $0.11/KPU-hr × minimum 2 KPU = $0.22/hr = ~$160/mo just running.** Plus $0.10/GB for running-app storage. Dev/stop when idle.
- **DynamoDB sink:** on-demand $1.25/M writes
- **S3 checkpoint storage:** $0.023/GB/mo (tiny)
- Total for this lesson (MSF app running a few hours during dev, small stream): **~$5–15** if you stop the Flink app when done.

---

## Step 1: Kinesis stream + DynamoDB sink
> **Why:** Kinesis is the durable ordered log in front of Flink. DynamoDB is the idempotent sink — key on `(userId, windowEnd)` so retries produce the same row.

```typescript
import * as kinesis from 'aws-cdk-lib/aws-kinesis';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const stream = new kinesis.Stream(this, 'EventsStream', {
  streamName: 'events',
  streamMode: kinesis.StreamMode.ON_DEMAND,
  retentionPeriod: cdk.Duration.hours(24),
});

const agg = new dynamodb.Table(this, 'Agg', {
  tableName: 'WindowAgg',
  partitionKey: { name: 'userId', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'windowEnd', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

## Step 2: Flink app — tumbling 1-min count per user
> **Why:** A tumbling window emits exactly once per interval per key. Simplest windowing primitive; great for "events per user per minute" dashboards.

`TumblingCount.java` (Flink DataStream API, Java 11):
```java
import org.apache.flink.streaming.api.datastream.*;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.kinesis.FlinkKinesisConsumer;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;

public class TumblingCount {
  public static void main(String[] args) throws Exception {
    var env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.enableCheckpointing(60_000); // 60s

    var src = env.addSource(new FlinkKinesisConsumer<>("events", new EventSchema(), kinesisProps()));

    src.assignTimestampsAndWatermarks(
          WatermarkStrategy.<Event>forBoundedOutOfOrderness(java.time.Duration.ofSeconds(10))
            .withTimestampAssigner((e, t) -> e.ts))
       .keyBy(e -> e.userId)
       .window(TumblingEventTimeWindows.of(Time.minutes(1)))
       .aggregate(new CountAgg(), new EmitToDynamo())
       .addSink(new DynamoSink("WindowAgg"));

    env.execute("tumbling-count");
  }
}
```

Deploy the app JAR to S3, then register via CDK:
```typescript
import * as kda from 'aws-cdk-lib/aws-kinesisanalyticsv2';

new kda.CfnApplicationV2(this, 'FlinkApp', {
  applicationName: 'tumbling-count',
  runtimeEnvironment: 'FLINK-1_19',
  serviceExecutionRole: flinkRole.roleArn,
  applicationConfiguration: {
    applicationCodeConfiguration: {
      codeContent: { s3ContentLocation: {
        bucketArn: codeBucket.bucketArn,
        fileKey: 'tumbling-count.jar',
      }},
      codeContentType: 'ZIPFILE',
    },
    flinkApplicationConfiguration: {
      checkpointConfiguration: { configurationType: 'CUSTOM', checkpointingEnabled: true, checkpointInterval: 60_000 },
      parallelismConfiguration: { configurationType: 'CUSTOM', parallelism: 2, parallelismPerKpu: 1, autoScalingEnabled: true },
    },
  },
});
```

## Step 3: Sliding window
> **Why:** Tumbling emits once/minute. Sliding emits every minute for the last 5 minutes — smoother dashboards, catches short bursts that tumbling would split across windows.

```java
.window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
```
Each event participates in 5 windows now → 5x the state. Memory goes up; plan KPU accordingly.

## Step 4: Session window
> **Why:** Session = group events until user is idle N minutes. Perfect for "user session analytics" (page views, game rounds, shopping carts).

```java
.window(EventTimeSessionWindows.withGap(Time.minutes(30)))
```

## Step 5: Exactly-once via checkpointing
> **Why:** Flink's checkpoint mechanism snapshots operator state + source offsets to S3. On restart, the app rewinds Kinesis to the checkpointed position and replays. Combined with an **idempotent DynamoDB sink** (same key = same row), you get end-to-end exactly-once.

```java
env.enableCheckpointing(60_000);
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30_000);
env.getCheckpointConfig().setCheckpointStorage("s3://flink-checkpoints-123/chk/");
```

**Idempotent sink pattern:**
```java
// key: userId + windowEnd (deterministic), so retry = overwrite
PutItemRequest req = PutItemRequest.builder()
  .tableName("WindowAgg")
  .item(Map.of(
    "userId",    AttributeValue.fromS(key),
    "windowEnd", AttributeValue.fromS(windowEnd.toString()),
    "count",     AttributeValue.fromN(String.valueOf(count))
  )).build();
```

**Test:** kill the Flink task manager mid-stream, restart from checkpoint, verify row count in DynamoDB is exactly = events ingested (no dup, no loss).

## Step 6: Late data + allowed lateness
> **Why:** Real-world events arrive late (network, mobile offline). Watermarks declare "I've seen everything up to time T". `allowedLateness` keeps a window open slightly longer to absorb stragglers.

```java
.window(TumblingEventTimeWindows.of(Time.minutes(1)))
.allowedLateness(Time.seconds(30))
.sideOutputLateData(lateTag)
```

Late events beyond 30s go to a side output → a "late-events" DynamoDB table for reconciliation.

## Step 7: Backpressure observation
> **Why:** When sink is slower than source, Flink buffers → memory grows → eventually OOM. Detect via the `backPressureTimeMs` metric or CloudWatch `downtime`.

Force it: add a `Thread.sleep(1000)` to the sink, run for a few minutes, watch `backPressuredTimeMsPerSecond` approach 1000 → app is 100% blocked.

Fix: scale parallelism, batch sink writes, or add a Kinesis buffer stream in between.

## Step 8: Lambda consumer trade-off
> **Why:** Lambda with Kinesis ESM (event source mapping) is simpler, cheaper for low volume, but has no built-in windowing, no exactly-once state, and processor concurrency = shard count × ParallelizationFactor.

```typescript
new lambda.EventSourceMapping(this, 'KinesisEsm', {
  target: consumerFn,
  eventSourceArn: stream.streamArn,
  startingPosition: lambda.StartingPosition.LATEST,
  batchSize: 100,
  parallelizationFactor: 10,        // 10 concurrent invocations per shard
  bisectBatchOnError: true,
  maximumRetryAttempts: 5,
  tumblingWindow: cdk.Duration.seconds(60),  // Lambda can do simple tumbling!
});
```

**Trade-off summary:**
| | Lambda + ESM | Flink (MSF) |
|---|---|---|
| Min cost | ~$1/mo | ~$160/mo |
| Stateful windows | Basic tumbling only | Full (tumbling/sliding/session) |
| Exactly-once | Via idempotent code | Built-in via checkpoints |
| Complex joins / CEP | No | Yes |
| Ops | Near-zero | Checkpoint bucket, KPU sizing |

## Cleanup
```bash
# Stop Flink app FIRST — it's the expensive thing
aws kinesisanalyticsv2 stop-application --application-name tumbling-count --force true

cdk destroy
```

## Common Errors
- **Flink app in `AUTOSCALING`/`UPDATING` forever** → wait; checkpoint restore takes minutes. If stuck >30 min, check CloudWatch logs for IAM errors.
- **`LimitExceededException` on Kinesis** → too many shards-per-second API calls. Use on-demand or request limit increase.
- **Exactly-once-not-so-exactly-once** → sink isn't idempotent. Use deterministic keys or transactional writes.
- **Checkpoint fails: `Access Denied` on S3** → MSF service role missing `s3:PutObject` on checkpoint bucket.
- **Watermarks never advance** → one idle source partition holds everything back. Use `WatermarkStrategy.withIdleness(Duration.ofMinutes(1))`.
- **Lambda ESM throttles** → `ParallelizationFactor` too low, or downstream (DynamoDB/RDS) is the bottleneck.
