# Walkthrough — 05 Kinesis & MSK

## About this service
AWS has three streaming products: **Kinesis Data Streams** (shard-based, Kafka-like, simpler), **Kinesis Firehose** (fully-managed buffering delivery to S3/Redshift/OpenSearch), and **MSK** (managed Apache Kafka, for teams that need Kafka ecosystem compatibility). Streams ingest high-velocity events (clickstreams, IoT, log tails) for downstream processing.

**Why it matters:** streaming is the backbone of real-time analytics, event-driven architectures, and data lakes. Kinesis gives you Kafka semantics without running ZooKeeper; MSK gives you actual Kafka with all its tooling.

**When to use Kinesis Data Streams:** AWS-native stacks, simple fan-out, <1 GB/s per stream.
**When to use Firehose:** "I just want events in S3/Redshift in Parquet" — zero code.
**When to use MSK:** Kafka ecosystem (Connect, Streams, Schema Registry, KSQL), >1 GB/s, existing Kafka apps.
**When NOT to use:** low-volume, simple pub/sub (SNS/SQS is cheaper); batch ETL (use Glue).

## Estimated cost
- **Kinesis Data Streams on-demand: $0.04/hour per stream** + $0.04 per million records written
- **Provisioned: $0.015/shard-hour** (~$11/shard/month), $0.014 per million PUT payload units
- **Enhanced fan-out: $0.015 per consumer-shard-hour** + $0.013/GB retrieved
- **Firehose: $0.029/GB ingested** (first 500 TB)
- **MSK Serverless: $0.75/hour per cluster** + $0.10/GB-in + $0.05/GB-out (~$540/month idle!)
- **MSK provisioned `kafka.m7g.large`: ~$140/month** per broker
- Total for this lesson: **~$10-20** if destroyed within a day. Delete MSK promptly!

---

## Step 1: Kinesis Data Stream
> **Why:** Provisioned shards (predictable cost) vs on-demand (auto-scales, higher cost). 1 shard = 1 MB/s or 1000 records/sec ingress, 2 MB/s egress. On-demand auto-scales but costs ~2.5x at steady state.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as kinesis from 'aws-cdk-lib/aws-kinesis';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as sources from 'aws-cdk-lib/aws-lambda-event-sources';

export class StreamingStack extends cdk.Stack {
  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const stream = new kinesis.Stream(this, 'Events', {
      streamName: 'events',
      shardCount: 2,
      retentionPeriod: cdk.Duration.hours(24),
      encryption: kinesis.StreamEncryption.MANAGED,
    });

    const consumer = new lambda.Function(this, 'Consumer', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`
        exports.handler = async (e) => {
          for (const r of e.Records) {
            const data = Buffer.from(r.kinesis.data, 'base64').toString();
            console.log('key=%s seq=%s data=%s', r.kinesis.partitionKey, r.kinesis.sequenceNumber, data);
          }
        };
      `),
    });

    consumer.addEventSource(new sources.KinesisEventSource(stream, {
      startingPosition: lambda.StartingPosition.LATEST,
      batchSize: 100,
      maxBatchingWindow: cdk.Duration.seconds(5),
    }));

    new cdk.CfnOutput(this, 'StreamName', { value: stream.streamName });
  }
}
```

## Step 2: Produce records
> **Why:** Partition key determines shard placement. Same key always goes to the same shard (ordering guarantee per key). Production uses KPL for batching/retries; for a demo, boto3 is fine.

```python
import boto3, json, random
k = boto3.client('kinesis')
for i in range(10000):
    k.put_record(
        StreamName='events',
        Data=json.dumps({'user': f'u{i%100}', 'event': random.choice(['click','view','buy'])}),
        PartitionKey=f'u{i%100}',
    )
```

CloudWatch → Kinesis → `IncomingRecords` climbs; Lambda consumer logs show messages flowing.

## Step 3: Enhanced fan-out
> **Why:** Standard consumers share a 2 MB/s-per-shard read budget. 5 consumers = 400 KB/s each. Enhanced fan-out gives EACH consumer its own 2 MB/s pipe and push-based delivery (<70 ms). Costs extra per consumer.

```typescript
const efoConsumer = new kinesis.CfnStreamConsumer(this, 'Efo', {
  streamArn: stream.streamArn,
  consumerName: 'fast-analytics',
});

consumer.addEventSource(new sources.KinesisEventSource(stream, {
  startingPosition: lambda.StartingPosition.LATEST,
  // Enhanced fan-out in CDK: use KinesisConsumerEventSource with the consumer ARN
}));
```

## Step 4: Firehose to S3 in Parquet with dynamic partitioning
> **Why:** Zero-code ETL. Firehose buffers events, converts to Parquet (columnar, compressed, query-friendly), partitions by `eventType` so Athena can prune. Output is a ready-to-query data lake.

```typescript
import * as firehose from 'aws-cdk-lib/aws-kinesisfirehose';
import * as glue from 'aws-cdk-lib/aws-glue';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';

const bucket = new s3.Bucket(this, 'Lake', { removalPolicy: cdk.RemovalPolicy.DESTROY, autoDeleteObjects: true });

const db = new glue.CfnDatabase(this, 'Db', {
  catalogId: this.account,
  databaseInput: { name: 'events_db' },
});

const table = new glue.CfnTable(this, 'Tbl', {
  catalogId: this.account,
  databaseName: 'events_db',
  tableInput: {
    name: 'events',
    tableType: 'EXTERNAL_TABLE',
    parameters: { classification: 'parquet' },
    storageDescriptor: {
      columns: [
        { name: 'user', type: 'string' },
        { name: 'event', type: 'string' },
      ],
      location: `s3://${bucket.bucketName}/events/`,
      inputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
      outputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat',
      serdeInfo: { serializationLibrary: 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe' },
    },
  },
});
table.addDependency(db);

const fhRole = new iam.Role(this, 'FhRole', { assumedBy: new iam.ServicePrincipal('firehose.amazonaws.com') });
bucket.grantReadWrite(fhRole);
stream.grantRead(fhRole);

new firehose.CfnDeliveryStream(this, 'ToS3', {
  deliveryStreamType: 'KinesisStreamAsSource',
  kinesisStreamSourceConfiguration: { kinesisStreamArn: stream.streamArn, roleArn: fhRole.roleArn },
  extendedS3DestinationConfiguration: {
    bucketArn: bucket.bucketArn,
    roleArn: fhRole.roleArn,
    bufferingHints: { intervalInSeconds: 60, sizeInMBs: 64 },
    prefix: 'events/type=!{partitionKeyFromQuery:event}/',
    errorOutputPrefix: 'errors/',
    dynamicPartitioningConfiguration: { enabled: true },
    processingConfiguration: {
      enabled: true,
      processors: [{
        type: 'MetadataExtraction',
        parameters: [
          { parameterName: 'MetadataExtractionQuery', parameterValue: '{event:.event}' },
          { parameterName: 'JsonParsingEngine', parameterValue: 'JQ-1.6' },
        ],
      }],
    },
    dataFormatConversionConfiguration: {
      enabled: true,
      inputFormatConfiguration: { deserializer: { openXJsonSerDe: {} } },
      outputFormatConfiguration: { serializer: { parquetSerDe: {} } },
      schemaConfiguration: {
        catalogId: this.account,
        databaseName: 'events_db',
        tableName: 'events',
        roleArn: fhRole.roleArn,
      },
    },
  },
});
```

Wait ~60 sec, then:
```bash
aws s3 ls s3://$LAKE/events/ --recursive
# events/type=click/2026/.../
# events/type=view/2026/.../
```

Query with Athena:
```sql
SELECT event, count(*) FROM events_db.events GROUP BY event;
```

## Step 5: MSK Serverless
> **Why:** Full Kafka API without managing brokers. Serverless auto-scales capacity. Authenticates via IAM — no SASL/SCRAM setup. **WARNING: ~$540/month floor** — destroy promptly.

```typescript
import * as msk from 'aws-cdk-lib/aws-msk';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

const kafkaSg = new ec2.SecurityGroup(this, 'KafkaSg', { vpc: props.vpc });

new msk.CfnServerlessCluster(this, 'MskS', {
  clusterName: 'demo',
  clientAuthentication: { sasl: { iam: { enabled: true } } },
  vpcConfigs: [{
    subnetIds: props.vpc.privateSubnets.map(s => s.subnetId),
    securityGroups: [kafkaSg.securityGroupId],
  }],
});
```

From bastion with IAM auth:
```bash
bin/kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --command-config client.properties --create --topic demo --partitions 3
bin/kafka-console-producer.sh --bootstrap-server $BOOTSTRAP \
  --producer.config client.properties --topic demo
> hello
```

## Step 6: MSK Connect — S3 sink
> **Why:** Managed Kafka Connect. Deploy a connector JAR; MSK Connect runs workers that sink each topic to S3. Zero custom consumer code.

```bash
# Upload the Confluent S3 Sink connector zip to S3
aws kafkaconnect create-custom-plugin ...
aws kafkaconnect create-connector \
  --connector-name s3-sink \
  --kafka-cluster '{...MskS arn...}' \
  --plugin '{...}' \
  --connector-configuration '{
    "connector.class":"io.confluent.connect.s3.S3SinkConnector",
    "topics":"demo",
    "s3.bucket.name":"'$LAKE'",
    "flush.size":"1000"
  }'
```

## Cleanup
```bash
# Delete MSK IMMEDIATELY — biggest cost
aws kafka delete-cluster --cluster-arn $MSK_ARN
cdk destroy
```

## Common Errors
- **`ProvisionedThroughputExceededException` on PutRecord** — shard over 1 MB/s or 1000 rec/s. Reshard or switch on-demand.
- **Firehose buffers but nothing in S3** — IAM role missing s3:PutObject, OR data format conversion failing (check error prefix).
- **Athena returns 0 rows** — Glue table schema mismatches Parquet; Firehose partition keys not registered as partitions (run `MSCK REPAIR TABLE events`).
- **MSK consumer can't connect** — IAM auth requires `aws-msk-iam-auth.jar` on classpath and correct `sasl.jaas.config`.
- **Enhanced fan-out: "LimitExceededException"** — max 20 consumers per stream.
- **MSK Serverless idle charge surprise** — $0.75/hr = $540/mo. No pause. Delete when not in use.
