# Walkthrough — 02 DynamoDB Basics

## About this service
**DynamoDB** is AWS's managed NoSQL DB. Key-value + document. Single-digit ms latency at any scale. **Partition key** determines which physical partition stores your item. Serverless (on-demand) or provisioned capacity. No connections — just API calls.

**Why it matters:** Dynamo is the default for high-scale, predictable-access-pattern workloads. Session stores, user profiles, IoT data, game leaderboards, event logs. It never slows down if you model your access patterns right.

**When to use DynamoDB:** known access patterns, key-value/simple queries, burst traffic, global scale (Global Tables).
**When NOT to use DynamoDB:** ad-hoc SQL queries (use RDS/Redshift), complex joins (use RDS), data you don't know yet how you'll query (use RDS first, migrate later), analytics (use S3 + Athena).

## Estimated cost
- **On-demand: $1.25 per million writes + $0.25 per million reads** (us-east-1)
- **Provisioned: $0.00065/WCU/hr + $0.00013/RCU/hr**
- **Storage: $0.25/GB/mo**
- **Streams: $0.02/100k read requests**
- **Free tier: 25GB storage + 25 RCU + 25 WCU** forever
- Total for this lesson: **$0.00** (within free tier)

---

## Step 1: Stack
> **Why:** `PAY_PER_REQUEST` = no capacity planning. `timeToLiveAttribute` = automatic deletion of old items (sessions, temporary data). These are the modern defaults.

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

new dynamodb.Table(this, 'Users', {
  tableName: 'Users',
  partitionKey: { name: 'userId', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  encryption: dynamodb.TableEncryption.AWS_MANAGED,
  timeToLiveAttribute: 'expiresAt',
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

new dynamodb.Table(this, 'Orders', {
  tableName: 'Orders',
  partitionKey: { name: 'userId', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'orderId', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

## Step 2: CRUD
> **Why:** DynamoDB's JSON "DynamoDB type" format (`{"S":"..."}`, `{"N":"..."}`) is verbose but explicit. SDKs usually wrap this — but CLI shows the raw form.

```bash
aws dynamodb put-item --table-name Users \
  --item '{"userId":{"S":"u1"},"name":{"S":"Alice"},"email":{"S":"a@x.com"}}'

aws dynamodb get-item --table-name Users --key '{"userId":{"S":"u1"}}'

aws dynamodb query --table-name Users \
  --key-condition-expression "userId = :u" \
  --expression-attribute-values '{":u":{"S":"u1"}}'
```

## Step 3: Composite key — Orders
> **Why:** Composite keys (PK + SK) model one-to-many. `userId` partition groups all orders for a user; `orderId` sort key distinguishes individual orders. The basis of single-table design.

```bash
for i in 1 2 3; do
  aws dynamodb put-item --table-name Orders \
    --item "{\"userId\":{\"S\":\"u1\"},\"orderId\":{\"S\":\"o$i\"},\"total\":{\"N\":\"$((i*10))\"}}"
done

aws dynamodb query --table-name Orders \
  --key-condition-expression "userId = :u" \
  --expression-attribute-values '{":u":{"S":"u1"}}'
```

## Step 4: Scan vs Query
> **Why:** Scan reads every item in the table. Query reads only one partition. At 1M rows, scan costs $100s. Measuring the difference firsthand teaches you to avoid scan.

Populate 1000 items, then:
```bash
time aws dynamodb scan --table-name Users --select COUNT >/dev/null
# ~3-5s, 1000 items scanned

time aws dynamodb query --table-name Users \
  --key-condition-expression "userId = :u" \
  --expression-attribute-values '{":u":{"S":"u500"}}' >/dev/null
# ~0.1s, 1 item
```

## Step 5: Billing modes
> **Why:** On-demand is 7x more expensive per request than provisioned at sustained load. But provisioned can throttle. Switch to provisioned once traffic is predictable.

```typescript
billingMode: dynamodb.BillingMode.PROVISIONED,
readCapacity: 5,
writeCapacity: 5,
```

RCU math:
- 1 RCU = 1 strong-consistent read of 4KB/sec (or 2 eventually-consistent)
- 100 reads/sec of 8KB (eventual) = 100 × 2 KB/4KB / 2 = 100 RCU

## Step 6: TTL
> **Why:** Automatic per-item expiration. No cron jobs. Critical for sessions, temporary data. Note: actual deletion can lag up to 48h — don't rely on exact timing.

```bash
EXPIRE=$(( $(date +%s) + 300 ))   # 5 min from now
aws dynamodb put-item --table-name Users \
  --item "{\"userId\":{\"S\":\"temp\"},\"expiresAt\":{\"N\":\"$EXPIRE\"}}"
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **`ValidationException: Query condition missed key schema element`** — must provide partition key.
- **Hot partition throttling** — bad key design (all traffic hits one PK).
- **TTL not deleting** — attribute must be `N` (epoch seconds), not ISO string.
