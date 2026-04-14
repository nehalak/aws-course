# Walkthrough — 01 DynamoDB Advanced

## About this service
**DynamoDB** is AWS's serverless NoSQL key-value + document database. At scale, expert teams collapse dozens of relational tables into a *single* DynamoDB table using PK/SK overloading, GSIs, streams, TTL, and transactions. This is the "single-table design" pattern popularized by Rick Houlihan.

**Why it matters:** a well-designed DynamoDB table answers every access pattern in single-digit ms at any throughput. Teams that treat it like SQL get burned by hot partitions and scan costs. Teams that embrace key overloading unlock massive scale at flat cost.

**When to use:** known access patterns, high write/read throughput, serverless stacks, event sourcing, session stores.
**When NOT to use:** ad-hoc analytics, relational joins, when access patterns are unknown, large blobs (use S3 with DynamoDB pointer).

## Estimated cost
- **On-demand table: $1.25 per million writes, $0.25 per million reads** (us-east-1)
- **Storage: $0.25/GB/month** after 25 GB free
- **Streams: $0.02 per 100K read requests**
- **GSI: same pricing as base table**, separate capacity
- **PITR backup: $0.20/GB/month**
- Total for this lesson: **~$1/month** at learning volumes. Destroy after!

---

## Step 1: Single-table stack
> **Why:** One table is cheaper, faster, and serves multiple entities via PK/SK overloading. `GSI1PK/GSI1SK` is a generic secondary index name — populate it only on items that need that query. Streams feed Lambda; TTL auto-expires items.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ddb from 'aws-cdk-lib/aws-dynamodb';

export class DynamoAdvancedStack extends cdk.Stack {
  public readonly table: ddb.Table;

  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.table = new ddb.Table(this, 'AppTable', {
      tableName: 'app-single',
      partitionKey: { name: 'PK', type: ddb.AttributeType.STRING },
      sortKey: { name: 'SK', type: ddb.AttributeType.STRING },
      billingMode: ddb.BillingMode.PAY_PER_REQUEST,
      stream: ddb.StreamViewType.NEW_AND_OLD_IMAGES,
      timeToLiveAttribute: 'expiresAt',
      pointInTimeRecovery: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    this.table.addGlobalSecondaryIndex({
      indexName: 'GSI1',
      partitionKey: { name: 'GSI1PK', type: ddb.AttributeType.STRING },
      sortKey: { name: 'GSI1SK', type: ddb.AttributeType.STRING },
      projectionType: ddb.ProjectionType.ALL,
    });

    new cdk.CfnOutput(this, 'TableName', { value: this.table.tableName });
    new cdk.CfnOutput(this, 'StreamArn', { value: this.table.tableStreamArn! });
  }
}
```

## Step 2: Insert overloaded items
> **Why:** The same table stores Users, Orders, and Products — differentiated only by the PK/SK prefix convention. This is the single-table pattern: one query can fetch a user AND their orders in one `Query` call on `PK=USER#u1`.

```bash
aws dynamodb put-item --table-name app-single --item '{
  "PK":{"S":"USER#u1"},"SK":{"S":"USER#u1"},
  "name":{"S":"alice"},"email":{"S":"a@x.com"}
}'

aws dynamodb put-item --table-name app-single --item '{
  "PK":{"S":"USER#u1"},"SK":{"S":"ORDER#2026-01-15#o1"},
  "total":{"N":"42"},"status":{"S":"SHIPPED"},
  "GSI1PK":{"S":"STATUS#SHIPPED"},"GSI1SK":{"S":"2026-01-15#o1"}
}'
```

Query "all items for user u1" (profile + orders in one call):
```bash
aws dynamodb query --table-name app-single \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"USER#u1"}}'
```

Query "all SHIPPED orders" via GSI1:
```bash
aws dynamodb query --table-name app-single --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"STATUS#SHIPPED"}}'
```

Expected: both commands return the items you just wrote.

## Step 3: Streams + Lambda
> **Why:** Streams capture every write to the table. A Lambda trigger can replicate changes to S3 (event log / analytics), notify other services, or maintain materialized views. This is the foundation of event-driven patterns on DynamoDB.

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as sources from 'aws-cdk-lib/aws-lambda-event-sources';

const fn = new lambda.Function(this, 'StreamFn', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (e) => {
      for (const r of e.Records) {
        console.log(r.eventName, JSON.stringify(r.dynamodb));
      }
    };
  `),
});

fn.addEventSource(new sources.DynamoEventSource(this.table, {
  startingPosition: lambda.StartingPosition.LATEST,
  batchSize: 10,
  retryAttempts: 2,
}));
```

Trigger an update; CloudWatch Logs should show `MODIFY` with old + new images.

## Step 4: TTL auto-expiry
> **Why:** TTL silently deletes items whose `expiresAt` epoch is in the past. Perfect for sessions, carts, temporary tokens. Deletion can lag up to 48h but costs nothing — no WCU consumed.

```bash
EXPIRE=$(($(date +%s) + 60))
aws dynamodb put-item --table-name app-single --item "{
  \"PK\":{\"S\":\"SESSION#s1\"},\"SK\":{\"S\":\"SESSION#s1\"},
  \"jwt\":{\"S\":\"abc...\"},\"expiresAt\":{\"N\":\"$EXPIRE\"}
}"
```

Wait a few minutes then `get-item` — eventually returns empty.

## Step 5: Transactions
> **Why:** `TransactWriteItems` makes up to 100 writes atomic across items AND tables. If any `ConditionExpression` fails, ALL writes roll back. This is how you implement money transfers, inventory reservation, atomic counters with invariants.

```bash
aws dynamodb transact-write-items --transact-items '[
  {"Update":{"TableName":"app-single",
    "Key":{"PK":{"S":"USER#u1"},"SK":{"S":"USER#u1"}},
    "UpdateExpression":"ADD balance :neg",
    "ConditionExpression":"balance >= :amt",
    "ExpressionAttributeValues":{":neg":{"N":"-10"},":amt":{"N":"10"}}}},
  {"Update":{"TableName":"app-single",
    "Key":{"PK":{"S":"USER#u2"},"SK":{"S":"USER#u2"}},
    "UpdateExpression":"ADD balance :pos",
    "ExpressionAttributeValues":{":pos":{"N":"10"}}}},
  {"Put":{"TableName":"app-single",
    "Item":{"PK":{"S":"AUDIT#t1"},"SK":{"S":"AUDIT#t1"},
            "from":{"S":"u1"},"to":{"S":"u2"},"amt":{"N":"10"}}}}
]'
```

If `u1.balance < 10`, the entire transaction fails with `TransactionCanceledException` and `u2` stays unchanged.

## Step 6: Optimistic locking
> **Why:** Without locks, two concurrent writers can clobber each other. `ConditionExpression: "version = :v"` with `ADD version :one` is the canonical pattern. Each writer bumps version; stale writer gets `ConditionalCheckFailedException`.

```bash
aws dynamodb update-item --table-name app-single \
  --key '{"PK":{"S":"USER#u1"},"SK":{"S":"USER#u1"}}' \
  --update-expression "SET #n = :n ADD version :one" \
  --condition-expression "version = :v" \
  --expression-attribute-names '{"#n":"name"}' \
  --expression-attribute-values '{":n":{"S":"Alicia"},":v":{"N":"0"},":one":{"N":"1"}}'
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **`ValidationException: The provided key element does not match the schema`** — sending only PK, forgot SK. Composite key tables need both.
- **`ProvisionedThroughputExceededException`** on a specific key — hot partition (>1000 WCU or >3000 RCU on one partition). Add random suffix to PK or redesign.
- **GSI returns stale data** — GSIs are eventually consistent; production code must tolerate lag.
- **TTL item still there after 1 min** — normal, TTL scanner runs every ~48h. Only delete is guaranteed *eventually*.
- **`TransactionCanceledException: ConditionalCheckFailed`** — condition expression returned false on at least one item; check `CancellationReasons`.
