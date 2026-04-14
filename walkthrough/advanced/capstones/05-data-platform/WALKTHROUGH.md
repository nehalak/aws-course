# Walkthrough — Capstone 05 Data Platform

## About this capstone
You will build a modern end-to-end data platform: Kinesis Firehose streaming app events into a partitioned Parquet data lake, DMS replicating a transactional RDS database as CDC into S3, Glue Data Catalog + Lake Formation governing who can read what, Glue ETL doing raw→curated transformations with data-quality rules, dbt models materialized in Redshift Serverless, QuickSight dashboards for business users, and OpenLineage feeds flowing into Amazon DataZone. This capstone synthesizes streaming ingestion, batch ETL, the lakehouse pattern, data governance, and BI — essentially the entire "data platform" job description in one stack.

**Why it matters:** Every mid-to-large company you have heard of has a team of 10-50 people maintaining this exact stack. Shipping an event to a dashboard is one of the most common end-to-end data engineering tasks; getting the plumbing right (exactly-once-ish, schema evolution, freshness SLAs, lineage) is the difference between a platform trusted by finance and one everyone avoids.

**Prerequisites:**
- `intermediate/kinesis` — Firehose, buffering, partitioning.
- `intermediate/s3` — prefixes, Parquet, Iceberg.
- `intermediate/glue` — catalog, crawlers, ETL, Data Quality.
- `intermediate/dms` — CDC, source filters.
- `intermediate/redshift` — Serverless, federated queries.
- `intermediate/quicksight` — datasets, SPICE.
- `advanced/lake-formation` — LF-tags, row/column filters.
- `advanced/observability` — OpenLineage, CloudWatch.

## Estimated cost
- Kinesis Firehose: $0.029 per GB ingested + data format conversion $0.018 per GB.
- S3 storage: $0.023/GB; Glue catalog: first 1M objects free.
- DMS replication instance (dms.t3.medium): $0.085/hour = **~$62/month**.
- Glue ETL: $0.44/DPU-hour — a 10-minute job on 2 DPU ≈ $0.15/run.
- Glue Data Quality: $0.44/DPU-hour (billed like ETL).
- Redshift Serverless: base 8 RPU @ $0.375/RPU-hour with 60s min — **idle cost is $0** but first query per hour spins up RPUs. Budget ~$40/mo under light use.
- QuickSight: Author $24/user/mo, Reader $0.30 per 30-min session (capped).
- Lake Formation: no extra cost; governs existing resources.
- Amazon DataZone: $0.0068 per domain unit-hour = ~$5/mo for a small domain.
- Total for this capstone: **~$120–200/month idle** driven by DMS + Firehose + Redshift probes + QuickSight authors. **WARN:** DMS bills 24/7 whether it is replicating or idle; stop the replication task AND the instance between sessions, or pay $60/mo for nothing. **Destroy after each session.**

---

## Architecture

```
  App events ----Firehose----> S3 raw (Parquet, year/month/day/hour)
                                   |
  RDS PostgreSQL ---DMS CDC-->     |     Glue Crawler -> Catalog
                                   v
                           Glue ETL (raw -> curated)
                           + Glue Data Quality rules
                                   |
                                   v
                       S3 curated (Iceberg tables)
                              /          \
                             v            v
                       Athena            Redshift Serverless (external schema)
                                              |
                                              v
                                          dbt models
                                              |
                                              v
                                          QuickSight dashboards
                                              |
                         OpenLineage --> DataZone <-- Glue catalog events
                         Lake Formation enforces access across all of it
```

## Step 1: CDK + dbt project layout
> **Why:** Data platforms have infra (CDK), transforms (dbt/SQL), and operational runbooks. Keep them separate directories so the analytics engineer editing dbt models does not have to navigate Terraform/CDK.

```
data-platform/
├── cdk/
│   ├── bin/app.ts
│   └── lib/
│       ├── lake-stack.ts         # S3 buckets, LF admin, KMS
│       ├── ingest-stack.ts       # Firehose, DMS
│       ├── catalog-stack.ts      # Glue catalog DB, crawlers
│       ├── etl-stack.ts          # Glue ETL jobs, DQ rules
│       ├── warehouse-stack.ts    # Redshift Serverless namespace/workgroup
│       ├── bi-stack.ts           # QuickSight datasets
│       └── governance-stack.ts   # LF tags, DataZone domain
├── dbt/
│   ├── models/{staging,marts}/*.sql
│   ├── tests/*.yml
│   └── dbt_project.yml
├── glue/
│   ├── raw_to_curated.py
│   └── dq_rules.json
├── contracts/
│   └── app_events.v1.yml
└── cdk.json
```

## Step 2: The lake bucket + Lake Formation admin
> **Why:** All data lives in S3 with zone-based prefixes — `raw/`, `curated/`, `marts/`. Lake Formation needs a data-lake admin role to register locations; without LF your IAM policies have to enumerate every table — impossible at scale.

```typescript
// lib/lake-stack.ts
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as lf from 'aws-cdk-lib/aws-lakeformation';

const lake = new s3.Bucket(this, 'Lake', {
  versioned: true, lifecycleRules: [{ abortIncompleteMultipartUploadAfter: cdk.Duration.days(1) }],
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
});

new lf.CfnDataLakeSettings(this, 'LFSettings', {
  admins: [{ dataLakePrincipalIdentifier: adminRole.roleArn }],
});

new lf.CfnResource(this, 'RegisterLake', {
  resourceArn: lake.bucketArn, useServiceLinkedRole: true,
});
```

Prefixes:
- `s3://lake/raw/firehose/events/year=.../month=.../...`
- `s3://lake/raw/dms/public.orders/...`
- `s3://lake/curated/events/` (Iceberg table)
- `s3://lake/curated/orders_cdc/` (Iceberg table)

## Step 3: Firehose ingestion with Parquet + partitioning
> **Why:** JSON on S3 is fine for debugging and terrible for analytics — every Athena query pays to decompress and parse. Firehose's built-in JSON → Parquet conversion plus dynamic partitioning writes query-ready files.

```typescript
// lib/ingest-stack.ts
import * as firehose from 'aws-cdk-lib/aws-kinesisfirehose';
import * as glue from 'aws-cdk-lib/aws-glue';

const eventsSchema = new glue.CfnTable(this, 'EventsSchema', {
  catalogId: this.account, databaseName: 'raw',
  tableInput: {
    name: 'events',
    storageDescriptor: {
      columns: [
        { name: 'user_id', type: 'string' }, { name: 'event_type', type: 'string' },
        { name: 'ts', type: 'timestamp' }, { name: 'payload', type: 'string' },
      ],
      location: `s3://${lake.bucketName}/raw/firehose/events/`,
      inputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
      outputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat',
      serdeInfo: { serializationLibrary: 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe' },
    },
  },
});

new firehose.CfnDeliveryStream(this, 'EventsStream', {
  deliveryStreamName: 'app-events',
  extendedS3DestinationConfiguration: {
    bucketArn: lake.bucketArn, roleArn: firehoseRole.roleArn,
    prefix: 'raw/firehose/events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/',
    errorOutputPrefix: 'raw/firehose/errors/',
    bufferingHints: { intervalInSeconds: 60, sizeInMBs: 64 },
    dataFormatConversionConfiguration: {
      enabled: true,
      inputFormatConfiguration: { deserializer: { openXJsonSerDe: {} } },
      outputFormatConfiguration: { serializer: { parquetSerDe: {} } },
      schemaConfiguration: { databaseName: 'raw', tableName: 'events', roleArn: firehoseRole.roleArn },
    },
  },
});
```

Producers post with:
```bash
aws firehose put-record --delivery-stream-name app-events --record Data=$(echo '{"user_id":"u1","event_type":"click","ts":"2026-04-14T10:00:00Z","payload":"{}"}' | base64)
```

## Step 4: DMS CDC from RDS into S3
> **Why:** The order/customer tables live in a transactional database. You do not run analytics there — you replicate. DMS with "migration type = full-load + ongoing-replication" captures the initial snapshot then streams changes into S3 as Parquet.

```typescript
// lib/ingest-stack.ts (continued)
import * as dms from 'aws-cdk-lib/aws-dms';

const srcEp = new dms.CfnEndpoint(this, 'Src', {
  endpointType: 'source', engineName: 'postgres',
  username: 'dms', password: secret.secretValueFromJson('password').unsafeUnwrap(),
  serverName: rds.clusterEndpoint.hostname, port: 5432, databaseName: 'app',
});

const tgtEp = new dms.CfnEndpoint(this, 'Tgt', {
  endpointType: 'target', engineName: 's3',
  s3Settings: {
    serviceAccessRoleArn: dmsS3Role.roleArn,
    bucketName: lake.bucketName, bucketFolder: 'raw/dms/',
    dataFormat: 'parquet', parquetVersion: 'parquet-2-0',
    includeOpForFullLoad: true, cdcInsertsAndUpdates: true,
    datePartitionEnabled: true,
  },
});

new dms.CfnReplicationTask(this, 'Task', {
  replicationInstanceArn: instance.ref,
  sourceEndpointArn: srcEp.ref, targetEndpointArn: tgtEp.ref,
  migrationType: 'full-load-and-cdc',
  tableMappings: JSON.stringify({ rules: [{
    'rule-type': 'selection', 'rule-id': '1', 'rule-name': 'orders',
    'object-locator': { 'schema-name': 'public', 'table-name': 'orders' },
    'rule-action': 'include',
  }]}),
});
```

## Step 5: Glue ETL raw → curated (Iceberg)
> **Why:** Curated data is deduped, enriched, quality-checked, and stored as Iceberg so you get ACID, time-travel, and schema evolution — which you will need because producers break contracts monthly.

```python
# glue/raw_to_curated.py
import sys
from awsglue.context import GlueContext
from pyspark.context import SparkContext
from awsglue.utils import getResolvedOptions

spark = SparkContext().getOrCreate()
gc = GlueContext(spark)

df = gc.create_dynamic_frame.from_catalog(database='raw', table_name='events').toDF()
df = df.dropDuplicates(['user_id', 'event_type', 'ts'])
df = df.withColumn('ingested_at', current_timestamp())

df.writeTo('glue_catalog.curated.events') \
  .using('iceberg') \
  .tableProperty('format-version', '2') \
  .partitionedBy('days(ts)') \
  .createOrReplace()
```

Scheduled via an EventBridge cron every 15 minutes.

## Step 6: Data quality rules
> **Why:** Untested data is untrusted data. Glue Data Quality runs expectations (null rates, uniqueness, referential integrity) and publishes pass/fail to CloudWatch — alarm on fail.

```json
// glue/dq_rules.json (DQDL)
Rules = [
  ColumnExists "user_id",
  IsComplete "user_id",
  ColumnValues "event_type" in [ "click", "view", "purchase", "signup" ],
  RowCount > 100,
  Uniqueness "user_id" > 0.95
]
```

```typescript
// lib/etl-stack.ts
new glue.CfnJob(this, 'DqJob', {
  name: 'events-dq', role: glueRole.roleArn,
  command: { name: 'glueetl', scriptLocation: 's3://.../dq_runner.py', pythonVersion: '3' },
  defaultArguments: {
    '--ruleset': '<inline DQDL>',
    '--datasource': 'curated.events',
    '--publish-metrics': 'true',
  },
});
```

CloudWatch alarm: `DataQualityScore < 0.95` → SNS to data oncall.

## Step 7: Redshift Serverless + dbt
> **Why:** Redshift Serverless with an external schema pointing at the Glue Catalog gives you "query the lake directly" semantics; dbt materializes marts as Redshift-native tables for BI-speed queries. Best of both worlds: no always-on warehouse, fast BI.

```typescript
// lib/warehouse-stack.ts (excerpt)
import * as rs from 'aws-cdk-lib/aws-redshiftserverless';

const ns = new rs.CfnNamespace(this, 'Ns', {
  namespaceName: 'analytics', adminUsername: 'admin',
  adminUserPassword: secret.secretValueFromJson('password').unsafeUnwrap(),
  defaultIamRoleArn: rsRole.roleArn, iamRoles: [rsRole.roleArn],
});

new rs.CfnWorkgroup(this, 'Wg', {
  workgroupName: 'analytics-wg', namespaceName: ns.namespaceName,
  baseCapacity: 8, subnetIds: vpc.privateSubnets.map(s => s.subnetId),
});
```

dbt project:
```yaml
# dbt/dbt_project.yml
name: analytics
models:
  analytics:
    staging: { +materialized: view }
    marts:   { +materialized: table }
```

```sql
-- dbt/models/staging/stg_events.sql
select user_id, event_type, ts, ingested_at from {{ source('curated','events') }}

-- dbt/models/marts/daily_active_users.sql
select date_trunc('day', ts) as day, count(distinct user_id) as dau
from {{ ref('stg_events') }} group by 1
```

## Step 8: QuickSight dashboards
> **Why:** Business users do not write SQL. A QuickSight dashboard with DAU, conversion funnel, and revenue-by-region built on the dbt marts is the visible output that justifies the entire platform.

```typescript
// lib/bi-stack.ts (excerpt)
new qs.CfnDataSource(this, 'Ds', {
  awsAccountId: this.account, dataSourceId: 'rs-analytics', name: 'rs-analytics',
  type: 'REDSHIFT',
  dataSourceParameters: { redshiftParameters: {
    host: workgroup.attrWorkgroupEndpointAddress, database: 'analytics',
    port: 5439,
  }},
  credentials: { credentialPair: { username: 'qs', password: qsPassword } },
});

new qs.CfnDataSet(this, 'DauDs', {
  awsAccountId: this.account, dataSetId: 'dau', name: 'daily-active-users',
  importMode: 'SPICE',
  physicalTableMap: { t: { customSql: {
    dataSourceArn: `arn:aws:quicksight:${this.region}:${this.account}:datasource/rs-analytics`,
    sqlQuery: 'select day, dau from marts.daily_active_users',
    columns: [{ name: 'day', type: 'DATETIME' }, { name: 'dau', type: 'INTEGER' }],
    name: 't',
  }}},
});
```

Five visualizations: DAU trend, new-user funnel, retention cohort heatmap, event-type distribution, geographic map.

## Step 9: Lineage — OpenLineage to DataZone
> **Why:** When the DAU chart drops you have exactly 15 minutes before the CEO asks "which job broke?". Lineage shows which upstream table feeds which model feeds which dashboard.

Install the OpenLineage Spark listener on the Glue job:

```python
# glue job default args
'--conf': 'spark.extraListeners=io.openlineage.spark.agent.OpenLineageSparkListener',
'--conf': 'spark.openlineage.transport.type=http',
'--conf': f'spark.openlineage.transport.url={marquez_url}',
```

DataZone ingests Glue catalog events + OpenLineage feed and displays the DAG.

## Step 10: Data contracts
> **Why:** A data contract is an enforceable schema + semantics agreement between the producer (the app team firing events) and the consumers. Without one, any app deployment can break every dashboard at 2am.

```yaml
# contracts/app_events.v1.yml
name: app_events
version: 1
owner: platform@example.com
freshness_sla_minutes: 15
schema:
  - name: user_id,    type: string, required: true
  - name: event_type, type: enum,   allowed: [click, view, purchase, signup]
  - name: ts,         type: timestamp, required: true
  - name: payload,    type: json
quality:
  - rule: completeness(user_id) > 0.99
  - rule: row_count > 1000 per day
```

A CI job validates that any change is additive (new column / new enum value — OK; remove / rename — block).

## Step 11: Deploy
> **Why:** Buckets + catalog first, then ingestion, then ETL, then warehouse, then BI.

```bash
npx cdk deploy LakeStack CatalogStack
npx cdk deploy IngestStack
# start DMS task
aws dms start-replication-task --replication-task-arn $T --start-replication-task-type start-replication
# seed 10k events via Firehose producer
python scripts/seed_events.py --count 10000
npx cdk deploy EtlStack WarehouseStack
cd dbt && dbt deps && dbt run --profiles-dir .
npx cdk deploy BiStack GovernanceStack
```

## Step 12: Freshness + backfill
> **Why:** The SLA is "data < 15 min old". A freshness check is a cheap Athena query compared to the cost of a stale dashboard. A backfill runbook exists for when Firehose errored for 3 hours.

```sql
-- freshness probe
select datediff('minute', max(ts), current_timestamp) as lag_minutes from curated.events;
-- alarm if > 15
```

Backfill:
```bash
# replay S3 raw → curated for a bad window
aws glue start-job-run --job-name events-etl \
  --arguments '{"--window_start":"2026-04-14T10:00:00Z","--window_end":"2026-04-14T13:00:00Z"}'
```

## Verification / success criteria
- Live event → Redshift query lag < 15 min (measure via a synthetic probe that inserts `ts=now` and queries).
- DMS task state `Running`, CDC lag < 60s at idle.
- Glue Data Quality alarm fires when you inject 500 null `user_id` rows.
- dbt test suite passes (`dbt test`); failing a test blocks the deploy.
- QuickSight dashboard loads in <5s from SPICE.
- Lake Formation: a user with only `event_type=view` row filter cannot `SELECT` rows with `event_type=purchase`.
- DataZone lineage DAG shows Firehose → raw.events → curated.events → stg_events → daily_active_users → dashboard.

## Cleanup
```bash
aws dms stop-replication-task --replication-task-arn $T
aws dms delete-replication-task --replication-task-arn $T
aws dms delete-replication-instance --replication-instance-arn $I  # this is the big billed one

# pause Redshift Serverless (bills are 0 when idle, but delete workgroup anyway)
aws redshift-serverless delete-workgroup --workgroup-name analytics-wg
aws redshift-serverless delete-namespace --namespace-name analytics

# empty S3 lake (versioned — use a lifecycle or explicit delete of all versions)
aws s3api list-object-versions --bucket $LAKE --query 'Versions[].[Key,VersionId]' --output text \
  | xargs -L 1 aws s3api delete-object --bucket $LAKE --key

npx cdk destroy --all
```

## Common Errors
- **Firehose records failing with schema error** → your JSON has a field type Firehose can't map; check the Glue schema and set `OpenXJsonSerDe` case-insensitive.
- **DMS CDC lag growing** → source RDS has no logical replication slot (PostgreSQL `rds.logical_replication=1` parameter group) or the instance is too small; scale up or split tables across tasks.
- **Glue Iceberg writes 500 `table not found`** → Iceberg catalog needs Glue catalog 2.0 + `--datalake-formats iceberg` job arg.
- **Redshift Serverless "namespace not available"** → first query after inactivity takes 30-60s to spin up; cache/warm-up query in the dashboard scheduler.
- **dbt `relation does not exist`** → dbt ran against `dev` schema but QuickSight looks at `marts`; align `target` in `profiles.yml`.
- **QuickSight "Could not reach the data source"** → Redshift workgroup is in private subnets; attach a VPC connection in QuickSight or use public endpoints (not recommended).
- **Lake Formation permission denied despite IAM allow** → LF uses "LF-allow + IAM-allow" (default: LF only on registered locations); double-grant or set `Use only IAM access control` if you are still migrating.
- **DataZone lineage empty** → the OpenLineage endpoint is wrong or the job role cannot reach it; check VPC + DataZone domain data source sync status.
- **$60/mo surprise for "nothing"** → the DMS replication instance was never deleted after the task ended. Always delete the instance, not just the task.
