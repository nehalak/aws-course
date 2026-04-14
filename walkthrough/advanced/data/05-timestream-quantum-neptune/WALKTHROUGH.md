# Walkthrough — 05 Timestream, QLDB, Neptune (Specialized Databases)

## About this service
These are **single-purpose databases** for shapes that RDBMS/NoSQL handle badly:
- **Timestream** — time-series DB with automatic memory (hot) / magnetic (cold) tiering. SQL-ish. Built for IoT / metrics.
- **QLDB** — immutable, cryptographically-verifiable ledger. **Deprecated** — Jul 2025 EOL; migrate to Aurora PostgreSQL with audit extensions.
- **Neptune** — property graph (Gremlin / openCypher) + RDF (SPARQL). For relationship-heavy queries (social, fraud, knowledge graphs).

**Why it matters:** a graph "friends-of-friends" traversal on a 10M-row Postgres table is a nightmare of self-joins; in Neptune it's a single Gremlin line. Downsampling 1B IoT points on Postgres melts the CPU; Timestream does it natively. Use the right shape.
**When to use:** Timestream — sensor data, CloudWatch-like metrics. Neptune — fraud rings, recommendation graphs, knowledge graphs, identity resolution. QLDB — don't; pick Aurora Postgres.
**When NOT to use:** Timestream for OLTP or non-timestamped data. Neptune when relationships are shallow (1 hop) — Postgres is fine. QLDB for anything new.

## Estimated cost
- **Timestream for LiveAnalytics:** writes $0.50/M, memory store $0.036/GB-hr, magnetic $0.03/GB/mo, queries $0.01/GB scanned
- **Timestream for InfluxDB:** **FLAG — instance-based, $0.176/hr for db.influx.medium = ~$128/mo minimum**
- **Neptune Serverless:** $0.1608/NCU-hr, min 1 NCU auto-pause after 10 min idle = ~$0.16/hr active
- **Neptune provisioned:** **FLAG — db.t3.medium = $0.094/hr = ~$69/mo, min 1 instance; prod needs 3-AZ = ~$207/mo**
- **QLDB:** $0.50/M I/O requests, $0.20/GB/mo (ignore — deprecated)
- Total for this lesson (Timestream LiveAnalytics + Neptune Serverless, short use): **~$5–15**

---

## Step 1: Timestream database + table
> **Why:** Memory tier handles fast writes + recent queries; magnetic tier is cheap long-term storage. Retention is per-table; writes older than memory-retention are rejected by default.

```typescript
import * as ts from 'aws-cdk-lib/aws-timestream';

const db = new ts.CfnDatabase(this, 'IotDb', { databaseName: 'iot' });

new ts.CfnTable(this, 'SensorTable', {
  databaseName: db.databaseName!,
  tableName: 'sensors',
  retentionProperties: {
    MemoryStoreRetentionPeriodInHours: '24',    // 24h hot
    MagneticStoreRetentionPeriodInDays: '365',  // 1yr cold
  },
  magneticStoreWriteProperties: {
    EnableMagneticStoreWrites: true,  // allow late-arriving data
  },
}).addDependency(db);
```

## Step 2: Ingest 10k IoT readings
> **Why:** Timestream's write model uses **dimensions** (indexed string tags like deviceId) and **measures** (numeric values). Multi-measure records are cheaper than single-measure.

```python
import boto3, time, random
ts = boto3.client('timestream-write')

records = []
now_ms = int(time.time() * 1000)
for i in range(10_000):
    records.append({
        'Dimensions': [{'Name': 'deviceId', 'Value': f'sensor-{i % 50}'}],
        'MeasureName': 'telemetry',
        'MeasureValues': [
            {'Name': 'temp_c',   'Value': str(20 + random.random()*10), 'Type': 'DOUBLE'},
            {'Name': 'humidity', 'Value': str(50 + random.random()*30), 'Type': 'DOUBLE'},
        ],
        'MeasureValueType': 'MULTI',
        'Time': str(now_ms - i * 1000),
        'TimeUnit': 'MILLISECONDS',
    })
    if len(records) == 100:   # max batch 100
        ts.write_records(DatabaseName='iot', TableName='sensors', Records=records)
        records = []
```

## Step 3: Downsample query
> **Why:** `bin()` buckets time into intervals; averaging inside the bucket gives a downsampled chart. This query on 1M rows in Postgres = slow self-join; in Timestream = native.

```sql
SELECT
  deviceId,
  bin(time, 1m) AS minute,
  ROUND(AVG(measure_value::double), 2) AS avg_temp
FROM "iot"."sensors"
WHERE time > ago(24h)
  AND measure_name = 'telemetry'
GROUP BY deviceId, bin(time, 1m)
ORDER BY minute DESC, deviceId
LIMIT 100;
```

Cost hint: queries are billed per GB scanned. `WHERE time > ago(24h)` hits memory only — cheap. Cross-tier queries (>24h) hit magnetic — slower, more data.

## Step 4: Memory vs. magnetic query comparison
> **Why:** Watch the difference. Use Query Insights to see bytes scanned and execution engine.

```sql
-- Memory only (last hour)  → ~1 MB scanned, ~100 ms
SELECT COUNT(*) FROM "iot"."sensors" WHERE time > ago(1h);

-- Crosses tiers (last 90 days)  → ~100 MB scanned, multi-second
SELECT COUNT(*) FROM "iot"."sensors" WHERE time > ago(90d);
```

## Step 5: Neptune Serverless cluster
> **Why:** Serverless scales NCUs automatically and pauses idle. Saves $$ on dev. Production needs provisioned multi-AZ.

```typescript
import * as neptune from '@aws-cdk/aws-neptune-alpha';

const cluster = new neptune.DatabaseCluster(this, 'Graph', {
  vpc,
  instanceType: neptune.InstanceType.SERVERLESS,
  serverlessScalingConfiguration: { minCapacity: 1, maxCapacity: 8 },
  iamAuthentication: true,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

## Step 6: Load a social graph (Gremlin)
> **Why:** Property graphs model "nodes with properties + typed edges". Gremlin traverses them declaratively.

```python
from gremlin_python.driver import client
c = client.Client(f'wss://{endpoint}:8182/gremlin', 'g')

# vertices
for u in ['alice','bob','carol','dan','erin']:
    c.submit(f"g.addV('person').property('name','{u}')").all().result()

# edges: alice-bob, bob-carol, carol-dan, dan-erin, alice-carol
edges = [('alice','bob'),('bob','carol'),('carol','dan'),('dan','erin'),('alice','carol')]
for a,b in edges:
    c.submit(f"g.V().has('name','{a}').as('a').V().has('name','{b}').addE('friend').from('a')").all().result()
```

## Step 7: Friends-of-friends (not already friends)
> **Why:** The canonical graph query. In SQL: multiple self-joins, nasty deduplication. In Gremlin: 3 lines.

```groovy
g.V().has('name','alice').as('me')
  .out('friend').out('friend')            // 2 hops
  .where(neq('me'))                       // not myself
  .where(not(__.as('me').out('friend').where(eq(__.label('fof')))))  // not direct friend
  .dedup()
  .values('name')
```

Expected: `dan` (alice → bob → carol → dan path; dan is not yet alice's direct friend).

## Step 8: Neptune bulk load from S3
> **Why:** Importing millions of edges row-by-row is slow. Neptune's bulk-load API ingests CSV/N-Triples/RDF from S3 in parallel.

```bash
curl -X POST https://$ENDPOINT:8182/loader \
  -H 'Content-Type: application/json' \
  -d '{
    "source": "s3://graph-ingest-123/edges.csv",
    "format": "csv",
    "iamRoleArn": "arn:aws:iam::123:role/NeptuneLoaderRole",
    "region": "us-east-1",
    "failOnError": "FALSE",
    "parallelism": "HIGH"
  }'
```

## Step 9: Relational vs. graph decision
> **Why:** Not every many-to-many belongs in a graph DB. Rule of thumb:

| Scenario | Use |
|---|---|
| 1-hop lookups ("user's friends") | Postgres / DynamoDB |
| Variable-depth paths, cycle detection, shortest path | Neptune |
| Fraud ring detection (5+ hops) | Neptune |
| Reporting on counts, aggregates | Postgres / Redshift |
| Recommendations ("users like you bought") | Neptune or Vector DB |

## Cleanup
```bash
# Neptune first
cdk destroy

# Timestream — no hourly charge, but delete to avoid storage costs
aws timestream-write delete-table --database-name iot --table-name sensors
aws timestream-write delete-database --database-name iot
```

## Common Errors
- **Timestream: `RejectedRecordsException` with `Timestamp is too far in the past`** → writing beyond memory retention. Enable `MagneticStoreWrites` or shorten timestamps.
- **Timestream queries slow/expensive** → no time filter → full table scan. Always include `WHERE time > ago(...)`.
- **Neptune Gremlin timeout** → query walked the whole graph. Bound with `.limit()` and index on start vertex.
- **Neptune bulk load `Access Denied`** → S3 bucket policy missing Neptune role + VPC endpoint missing for S3.
- **QLDB "deprecated"** — you cannot create new ledgers in many regions as of 2025. Use Aurora Postgres + `pgaudit`.
- **Neptune connect refused** → cluster in private subnet; you need a bastion or SSM session or VPN.
