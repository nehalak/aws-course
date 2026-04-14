# Walkthrough — 02 Analytics Stack

## About this service
The AWS analytics stack is a menu of engines, each tuned to a workload: **Athena** (serverless SQL on S3), **Redshift Serverless / RA3** (columnar warehouse for complex joins and BI), **EMR Serverless** (Spark/Hadoop for big-data compute), **Redshift Spectrum** (query S3 directly from Redshift), and **QuickSight** (BI dashboards). They share the Glue Catalog so one dataset is queryable from all of them.

**Why it matters:** choosing the wrong engine wastes money. Athena is free when idle but expensive on big scans; Redshift is cheap per-query at scale but has a fixed floor; EMR Spark wins for heavy transforms but has operational overhead.
**When to use:** you have a data lake and need SQL/BI/Spark on top. Scale beyond single-Postgres (~TB+).
**When NOT to use:** small datasets or OLTP — stick with Postgres. Ad-hoc single-query-per-day — Athena is cheaper than a provisioned warehouse.

## Estimated cost
- **Athena:** $5/TB scanned
- **Redshift Serverless:** **FLAG — $0.375/RPU-hour, 8 RPU minimum → $3/hr when active**; auto-pause after 5 min idle by default. A small workload: $20–$100/mo.
- **Redshift RA3:** **FLAG — ra3.xlplus = $1.086/hr/node × 2 nodes × 730 hr = ~$1,585/mo** if left running. Pause when not in use.
- **EMR Serverless:** $0.052968/vCPU-hr + $0.0057785/GB-hr memory — a 10-min Spark job on 4 vCPU/16GB = ~$0.05
- **QuickSight:** **FLAG — Enterprise $18/author/mo, $0.30/session/reader (or $5/reader/mo capped)**
- Total for this lesson (short-lived Redshift Serverless + a few Athena queries + one EMR job): **~$10–20**

---

## Step 1: Athena federated query (S3 + RDS)
> **Why:** Federation lets one Athena SQL statement join a warehouse fact table (S3 Parquet) with an operational dimension table (RDS Postgres) — no ETL required. Useful for "current customers × historical events" questions.

Deploy the RDS connector via CDK + Serverless Application Repository:
```typescript
import * as sar from 'aws-cdk-lib/aws-sam';

new sar.CfnApplication(this, 'AthenaJdbcConnector', {
  location: {
    applicationId: 'arn:aws:serverlessrepo:us-east-1:292517598671:applications/AthenaJdbcConnector',
    semanticVersion: '2024.41.1',
  },
  parameters: {
    LambdaFunctionName: 'athena-rds-connector',
    SecretNamePrefix: 'rds/',
    DefaultConnectionString: `postgres://jdbc:postgresql://${dbHost}:5432/app?user=$\{rds/app:username}&password=$\{rds/app:password}`,
    SpillBucket: spillBucket.bucketName,
  },
});
```

Then in Athena:
```sql
-- Register data source: 'rds' → lambda:athena-rds-connector
SELECT s.event, s.amount, r.customer_tier
FROM awsdatacatalog.datalake.events_parquet s
JOIN "rds"."public"."customers" r ON s.user_id = r.user_id
WHERE s.dt = '2026-04-01';
```

## Step 2: Redshift Serverless workgroup
> **Why:** Serverless = no cluster management, scales RPU automatically, pauses when idle. Best starting point before committing to RA3.

```typescript
import * as redshiftserverless from 'aws-cdk-lib/aws-redshiftserverless';

const ns = new redshiftserverless.CfnNamespace(this, 'Ns', {
  namespaceName: 'analytics-ns',
  adminUsername: 'admin',
  adminUserPassword: cdk.SecretValue.secretsManager('rs-admin').toString(),
  dbName: 'analytics',
  iamRoles: [redshiftRole.roleArn],
  defaultIamRoleArn: redshiftRole.roleArn,
});

new redshiftserverless.CfnWorkgroup(this, 'Wg', {
  workgroupName: 'analytics-wg',
  namespaceName: ns.namespaceName!,
  baseCapacity: 8, // 8 RPU minimum
  publiclyAccessible: false,
  subnetIds: vpc.selectSubnets({ subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS }).subnetIds,
});
```

## Step 3: COPY from S3 Parquet
> **Why:** `COPY` is Redshift's bulk loader — parallel, compressed, column-aware. Never insert row-by-row.

```sql
CREATE TABLE events (
  user_id VARCHAR(64),
  name VARCHAR(128),
  event VARCHAR(32),
  amount DECIMAL(10,2),
  dt DATE
)
DISTKEY (user_id)
SORTKEY (dt);

COPY events FROM 's3://datalake-curated-123456789012/events_parquet/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
FORMAT AS PARQUET;
```

## Step 4: Materialized view with auto-refresh
> **Why:** Precompute expensive aggregations. Redshift refreshes incrementally when the base table changes — dashboards query the MV instead of rescanning daily.

```sql
CREATE MATERIALIZED VIEW mv_daily_revenue
AUTO REFRESH YES
AS
SELECT dt, COUNT(*) as event_count, SUM(amount) as revenue
FROM events
WHERE event = 'purchase'
GROUP BY dt;

SELECT * FROM mv_daily_revenue ORDER BY dt DESC LIMIT 30;
```

## Step 5: Redshift Spectrum (query S3 directly)
> **Why:** When a dataset is too large to COPY into Redshift, or you need to join warehouse tables to cold S3 data, Spectrum queries Parquet in-place. Billed $5/TB scanned (same as Athena).

```sql
CREATE EXTERNAL SCHEMA spectrum_lake
FROM DATA CATALOG
DATABASE 'datalake'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftSpectrumRole';

SELECT e.dt, COUNT(*)
FROM spectrum_lake.events_parquet e
JOIN customers c ON e.user_id = c.user_id
GROUP BY e.dt;
```

## Step 6: EMR Serverless Spark job
> **Why:** EMR Serverless runs Spark without cluster management. Use for one-off heavy ETL jobs that would be slow in Glue or expensive in Athena.

`spark_agg.py`:
```python
from pyspark.sql import SparkSession, functions as F

spark = SparkSession.builder.appName('daily_agg').getOrCreate()

df = spark.read.parquet('s3://datalake-curated-123456789012/events_parquet/')
(df.filter(F.col('event') == 'purchase')
   .groupBy('dt')
   .agg(F.count('*').alias('cnt'), F.sum('amount').alias('rev'))
   .write.mode('overwrite')
   .parquet('s3://datalake-analytics-123456789012/daily_revenue/'))
```

Submit:
```bash
aws emr-serverless create-application \
  --release-label emr-7.1.0 --type SPARK \
  --name spark-app --region us-east-1

aws emr-serverless start-job-run \
  --application-id $APP_ID \
  --execution-role-arn arn:aws:iam::$ACCOUNT:role/EMRServerlessRole \
  --job-driver '{
    "sparkSubmit": {
      "entryPoint": "s3://scripts/spark_agg.py",
      "sparkSubmitParameters": "--conf spark.executor.cores=2 --conf spark.executor.memory=4g"
    }
  }'
```

## Step 7: QuickSight dashboard
> **Why:** Non-SQL users need dashboards, not queries. QuickSight connects to Athena/Redshift, caches in SPICE (in-memory), and serves interactive dashboards.

1. Console → QuickSight → **Manage data → New dataset → Athena**.
2. Pick `datalake.events_parquet`. Import to **SPICE** (in-memory, fast).
3. Create analysis → add visuals: **line chart (dt, COUNT)**, **bar (event, SUM amount)**, **KPI (today's revenue)**.
4. Add **filter control** on `dt` (date range).
5. **Publish → Share with group**.

## Cleanup
```bash
# Pause Redshift Serverless
aws redshift-serverless delete-workgroup --workgroup-name analytics-wg
aws redshift-serverless delete-namespace --namespace-name analytics-ns

# EMR Serverless app
aws emr-serverless stop-application --application-id $APP_ID
aws emr-serverless delete-application --application-id $APP_ID

# QuickSight subscription — unsubscribe in console (prorated)

cdk destroy
```

## Common Errors
- **Athena: `HIVE_PARTITION_SCHEMA_MISMATCH`** → crawler-inferred types differ from file. Force types in table DDL.
- **Redshift COPY: `S3 Query Exception Access Denied`** → attach `AmazonS3ReadOnlyAccess` to the Redshift IAM role AND trust relationship for `redshift.amazonaws.com`.
- **Redshift Serverless doesn't pause** → default auto-pause is 5 min of inactivity; long-running queries reset timer. Force idle or lower threshold.
- **Spectrum scan $$$** → missing partition predicate. `WHERE dt = '...'` or you scan every partition.
- **QuickSight "capacity exceeded"** → SPICE quota full. Increase or use direct-query mode.
- **EMR Serverless job stuck in `PENDING`** → VPC without NAT/gateway. App must reach S3.
