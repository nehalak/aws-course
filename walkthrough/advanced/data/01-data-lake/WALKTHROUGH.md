# Walkthrough — 01 Data Lake

## About this service
A **data lake** on AWS is a pattern, not a single service: **S3** stores raw/curated/analytics zones, **Glue** discovers and transforms data, the **Glue Data Catalog** holds schema metadata, **Lake Formation** grants fine-grained access (table / column / row / tag), and **Athena** runs serverless SQL on top. The pattern separates storage (cheap) from compute (on-demand), and lets multiple engines (Athena, EMR, Redshift Spectrum, SageMaker) share one governed dataset.

**Why it matters:** you can dump TBs of CSV/JSON/logs into S3 for pennies/GB and query them years later without ever spinning up a warehouse. With Lake Formation you can also satisfy GDPR/HIPAA column-level access — without rewriting pipelines.
**When to use:** heterogeneous sources, unknown-future-access patterns, audit/compliance requirements, shared data across many teams.
**When NOT to use:** you already know your queries and need sub-second latency (use Redshift or a serving DB); small (<100 GB) workload — just use Postgres.

## Estimated cost
- **S3 Standard:** $0.023/GB/mo; **S3 IA:** $0.0125/GB/mo; **Glacier Instant:** $0.004/GB/mo
- **Glue Crawler:** $0.44/DPU-hr (min 2 DPU, billed per second, 10 min minimum)
- **Glue ETL (Spark):** $0.44/DPU-hr (min 2 DPU, 1 min minimum) — **flag: a single 10-min job = ~$0.15, but left-on dev endpoints burn $$$**
- **Athena:** $5.00/TB scanned — columnar + partitions cut this by 100x
- **Lake Formation:** free (permissions layer); underlying services still billed
- Total for this lesson (10 GB sample, a few crawler/ETL runs, ~50 Athena queries on <1 GB): **~$2–3 one-time**

---

## Step 1: Three-zone bucket layout
> **Why:** Separating raw/curated/analytics keeps immutable source-of-truth (raw) away from transformed data (curated) and aggregates (analytics). Lifecycle rules tier cold raw data to Glacier automatically — raw often dominates storage bill.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class DataLakeStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const raw = new s3.Bucket(this, 'RawBucket', {
      bucketName: `datalake-raw-${this.account}`,
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
      lifecycleRules: [{
        transitions: [
          { storageClass: s3.StorageClass.INFREQUENT_ACCESS, transitionAfter: cdk.Duration.days(30) },
          { storageClass: s3.StorageClass.GLACIER_INSTANT_RETRIEVAL, transitionAfter: cdk.Duration.days(90) },
        ],
      }],
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    const curated = new s3.Bucket(this, 'CuratedBucket', {
      bucketName: `datalake-curated-${this.account}`,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    const analytics = new s3.Bucket(this, 'AnalyticsBucket', {
      bucketName: `datalake-analytics-${this.account}`,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    new cdk.CfnOutput(this, 'RawBucketName', { value: raw.bucketName });
  }
}
```

## Step 2: Upload sample CSV to raw zone
> **Why:** A crawler needs data to inspect. We land a small CSV of fake user events partitioned by date — the folder layout (`dt=YYYY-MM-DD`) becomes partition keys the crawler auto-detects.

```bash
mkdir -p sample/dt=2026-04-01
cat > sample/dt=2026-04-01/events.csv <<EOF
user_id,name,ssn,event,amount
u1,Alice,111-22-3333,purchase,42.50
u2,Bob,222-33-4444,view,0
u3,Carol,333-44-5555,purchase,99.00
EOF

aws s3 cp sample/ s3://datalake-raw-$ACCOUNT/events/ --recursive
```

## Step 3: Glue database + crawler
> **Why:** The crawler reads files in S3, infers schema, and registers a table in the Glue Data Catalog — turning a pile of CSVs into a queryable table. Running it on a schedule keeps schema in sync as new partitions land.

```typescript
import * as glue from 'aws-cdk-lib/aws-glue';
import * as iam from 'aws-cdk-lib/aws-iam';

const db = new glue.CfnDatabase(this, 'LakeDb', {
  catalogId: this.account,
  databaseInput: { name: 'datalake' },
});

const crawlerRole = new iam.Role(this, 'CrawlerRole', {
  assumedBy: new iam.ServicePrincipal('glue.amazonaws.com'),
  managedPolicies: [
    iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSGlueServiceRole'),
  ],
});
raw.grantRead(crawlerRole);

new glue.CfnCrawler(this, 'RawCrawler', {
  role: crawlerRole.roleArn,
  databaseName: 'datalake',
  targets: { s3Targets: [{ path: `s3://${raw.bucketName}/events/` }] },
  schemaChangePolicy: { updateBehavior: 'UPDATE_IN_DATABASE', deleteBehavior: 'LOG' },
  // schedule: { scheduleExpression: 'cron(0 * * * ? *)' }, // hourly if you want
});
```

Run it:
```bash
aws glue start-crawler --name <crawler-name>
aws glue get-table --database-name datalake --name events
```

## Step 4: Glue ETL — CSV to partitioned Parquet
> **Why:** CSV on S3 is readable but terrible for analytics: Athena scans the whole file for every query. Parquet is columnar + compressed → Athena scans only needed columns, and partition pruning skips unneeded days. Typical savings: 50–200x less scanned bytes.

`etl_csv_to_parquet.py` (upload to `s3://<scripts-bucket>/etl_csv_to_parquet.py`):
```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'SOURCE_DB', 'SOURCE_TABLE', 'TARGET_PATH'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext); job.init(args['JOB_NAME'], args)

df = glueContext.create_dynamic_frame.from_catalog(
    database=args['SOURCE_DB'], table_name=args['SOURCE_TABLE']
).toDF()

# write Parquet, partitioned by dt
df.write.mode('overwrite').partitionBy('dt').parquet(args['TARGET_PATH'])

job.commit()
```

CDK job definition:
```typescript
new glue.CfnJob(this, 'EtlJob', {
  name: 'csv-to-parquet',
  role: crawlerRole.roleArn,
  command: { name: 'glueetl', pythonVersion: '3', scriptLocation: `s3://${scripts.bucketName}/etl_csv_to_parquet.py` },
  glueVersion: '4.0',
  workerType: 'G.1X',
  numberOfWorkers: 2,
  defaultArguments: {
    '--SOURCE_DB': 'datalake',
    '--SOURCE_TABLE': 'events',
    '--TARGET_PATH': `s3://${curated.bucketName}/events_parquet/`,
    '--enable-metrics': 'true',
  },
});
```

## Step 5: Athena — partition pruning proof
> **Why:** This is the payoff. Same query, two formats: you'll see Parquet scan kilobytes vs. CSV scanning megabytes. Over a terabyte of logs, this is the difference between $5 and $0.02 per query.

Register curated table (run a second crawler on `curated/events_parquet/` or `MSCK REPAIR TABLE`), then in Athena:

```sql
-- scans whole CSV
SELECT COUNT(*) FROM datalake.events WHERE dt = '2026-04-01';
-- Data scanned: ~2 MB

-- scans only Parquet partition
SELECT COUNT(*) FROM datalake.events_parquet WHERE dt = '2026-04-01';
-- Data scanned: ~8 KB  (250x less)
```

## Step 6: Lake Formation — column-level security
> **Why:** Real data has PII. IAM lets you grant access to a whole table; Lake Formation lets you hide the `ssn` column from analysts while still letting them query `name` and `event`. This is how you ship a single governed dataset to engineering + finance + data-science.

```bash
# Register the bucket with LF (one-time per bucket)
aws lakeformation register-resource \
  --resource-arn arn:aws:s3:::datalake-curated-$ACCOUNT \
  --use-service-linked-role

# Grant alice SELECT on non-PII columns only
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::$ACCOUNT:user/alice \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "datalake",
      "Name": "events_parquet",
      "ColumnNames": ["user_id","name","event","amount","dt"]
    }
  }' \
  --permissions SELECT
```

As `alice`: `SELECT ssn FROM events_parquet` → **permission denied**. `SELECT name, amount FROM events_parquet` → works.

## Step 7: LF-tags for scale
> **Why:** Granting columns table-by-table doesn't scale. Tag tables `PII=yes|no` and grant a principal access to the tag value `PII=no` — new tables automatically respect it.

```bash
aws lakeformation create-lf-tag --tag-key PII --tag-values yes no
aws lakeformation add-lf-tags-to-resource \
  --resource '{"Table":{"DatabaseName":"datalake","Name":"events_parquet"}}' \
  --lf-tags TagKey=PII,TagValues=no
```

## Step 8: Glue Data Quality
> **Why:** Detect bad data before downstream dashboards break. DQ rules evaluate on a schedule; failures land in CloudWatch + EventBridge → page someone.

```
Rules = [
  RowCount > 0,
  IsComplete "user_id",
  Uniqueness "user_id" > 0.95,
  ColumnValues "amount" between 0 and 10000
]
```

## Cleanup
```bash
# Empty then destroy
aws s3 rm s3://datalake-raw-$ACCOUNT --recursive
aws s3 rm s3://datalake-curated-$ACCOUNT --recursive
aws s3 rm s3://datalake-analytics-$ACCOUNT --recursive
cdk destroy
```

## Common Errors
- **`AccessDeniedException` from Athena even though IAM allows it** → Lake Formation permissions are ANDed with IAM. Grant in LF too.
- **Crawler creates many tiny tables** → files in the same folder have different schemas. Fix layout or set `TableGroupingPolicy=CombineCompatibleSchemas`.
- **Parquet query returns 0 rows** → partitions not registered. Run `MSCK REPAIR TABLE` or re-crawl.
- **Glue ETL stuck "Starting"** → cold-start takes 1–3 min; not stuck. For short jobs, use **Python Shell** instead of Spark ($0.44 vs. $0.88+/hr).
- **LF "Default settings include IAMAllowedPrincipals"** → legacy mode bypasses LF. Remove that grant before column-level will take effect.
