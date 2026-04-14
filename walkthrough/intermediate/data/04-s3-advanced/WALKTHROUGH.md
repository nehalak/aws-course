# Walkthrough — 04 S3 Advanced

## About this service
**S3** is object storage that scales to exabytes. Beyond the basic `PUT`/`GET`, S3 has a deep toolkit: *Lifecycle* auto-tiers data between storage classes; *Replication* copies objects cross-region or cross-account; *Object Lambda* transforms content on-the-fly during GET; *Access Points* give per-team policies on a shared bucket; *S3 Select* queries objects with SQL without downloading.

**Why it matters:** data lakes, compliance archives, SaaS multi-tenant isolation, and cost optimization at petabyte scale all hinge on these features. Teams that ignore lifecycle rules often pay 10x for storage.

**When to use:** data lakes, static hosting, backups, media, cross-region DR, ML training datasets, log archives.
**When NOT to use:** low-latency random access <10 ms (use EFS/EBS), transactional data (use a database), tiny frequently-updated files (fees add up).

## Estimated cost
- **S3 Standard: $0.023/GB/month**, IA: $0.0125, Glacier Instant: $0.004, Glacier Deep: $0.00099 (us-east-1)
- **Requests: $0.005 per 1K PUT**, $0.0004 per 1K GET
- **CRR: $0.02/GB replication data transfer** + destination storage + $0.015 per 10K replicated objects
- **Replication Time Control (RTC): +$0.015/GB** 15-min SLA surcharge
- **Object Lambda: $0.005 per GB returned** + Lambda compute
- **S3 Select: $0.002/GB scanned + $0.0007/GB returned**
- Total for this lesson: **~$1-3/month** at demo volumes. Destroy after!

---

## Step 1: Bucket with lifecycle rules
> **Why:** Standard → IA after 30 days → Glacier after 90 → delete after 365. An object accessed twice a month still pays full Standard; unloved objects silently tier down. This is the #1 S3 cost-reduction lever.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class S3AdvancedStack extends cdk.Stack {
  public readonly src: s3.Bucket;
  public readonly dst: s3.Bucket;

  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.src = new s3.Bucket(this, 'Src', {
      versioned: true,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [{
        id: 'tier-and-expire',
        enabled: true,
        transitions: [
          { storageClass: s3.StorageClass.INFREQUENT_ACCESS, transitionAfter: cdk.Duration.days(30) },
          { storageClass: s3.StorageClass.GLACIER,           transitionAfter: cdk.Duration.days(90) },
        ],
        expiration: cdk.Duration.days(365),
        abortIncompleteMultipartUploadAfter: cdk.Duration.days(7),
      }],
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    new cdk.CfnOutput(this, 'SrcBucket', { value: this.src.bucketName });
  }
}
```

Upload a file; next day console shows "Storage class: STANDARD, transitions in 29 days".

## Step 2: Cross-region replication
> **Why:** DR across regions or compliance-driven data residency. Replication copies *new* objects only — use S3 Batch Replication for backfilling existing objects. RTC adds a 15-min SLA for regulated workloads.

```typescript
import * as iam from 'aws-cdk-lib/aws-iam';

// dst bucket in same stack for simplicity — production puts it in another region/stack
this.dst = new s3.Bucket(this, 'Dst', {
  versioned: true,    // replication REQUIRES versioning on both sides
  encryption: s3.BucketEncryption.S3_MANAGED,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
});

const replRole = new iam.Role(this, 'ReplRole', {
  assumedBy: new iam.ServicePrincipal('s3.amazonaws.com'),
});
this.src.grantRead(replRole);
this.dst.grantWrite(replRole);

const cfnSrc = this.src.node.defaultChild as s3.CfnBucket;
cfnSrc.replicationConfiguration = {
  role: replRole.roleArn,
  rules: [{
    id: 'crr-all',
    status: 'Enabled',
    priority: 1,
    filter: { prefix: '' },
    deleteMarkerReplication: { status: 'Enabled' },
    destination: {
      bucket: this.dst.bucketArn,
      // replicationTime + metrics together enable RTC
      replicationTime: { status: 'Enabled', time: { minutes: 15 } },
      metrics:         { status: 'Enabled', eventThreshold: { minutes: 15 } },
    },
  }],
};
```

Upload to src; `aws s3 ls s3://dst-bucket/` shows the object within seconds (up to 15 min under RTC SLA).

## Step 3: Object Lambda
> **Why:** Transform object content on GET without rewriting the data. Example: redact PII from CSVs on-the-fly. App calls the *access point* URL instead of S3 directly; Lambda streams the transformed response. Original bytes never change.

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as s3olap from 'aws-cdk-lib/aws-s3objectlambda';

const ap = new s3.CfnAccessPoint(this, 'Ap', {
  bucket: this.src.bucketName,
  name: 'src-ap',
});

const redactor = new lambda.Function(this, 'Redactor', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
import boto3, urllib.request, re
s3 = boto3.client('s3')
def handler(event, ctx):
    r = event['getObjectContext']
    body = urllib.request.urlopen(r['inputS3Url']).read().decode()
    redacted = re.sub(r'\\d{3}-\\d{2}-\\d{4}', 'XXX-XX-XXXX', body)
    s3.write_get_object_response(
        Body=redacted, RequestRoute=r['outputRoute'], RequestToken=r['outputToken'])
    return {'statusCode': 200}
`),
});

new s3olap.CfnAccessPoint(this, 'OlapAp', {
  name: 'src-olap',
  objectLambdaConfiguration: {
    supportingAccessPoint: `arn:aws:s3:${this.region}:${this.account}:accesspoint/src-ap`,
    transformationConfigurations: [{
      actions: ['GetObject'],
      contentTransformation: { AwsLambda: { FunctionArn: redactor.functionArn } },
    }],
  },
});
```

`aws s3api get-object --bucket arn:aws:s3-object-lambda:...:accesspoint/src-olap --key ssn.csv out.csv` — SSNs appear as `XXX-XX-XXXX`.

## Step 4: Access Points for team isolation
> **Why:** One bucket, many teams, separate policies. Each access point has its own policy and hostname. Revoking a team = delete their AP. Much simpler than a 10-KB bucket policy with 50 conditions.

```typescript
new s3.CfnAccessPoint(this, 'TeamA', {
  bucket: this.src.bucketName,
  name: 'team-a',
  policy: {
    Version: '2012-10-17',
    Statement: [{
      Effect: 'Allow',
      Principal: { AWS: 'arn:aws:iam::123456789012:role/team-a-role' },
      Action: 's3:GetObject',
      Resource: `arn:aws:s3:${this.region}:${this.account}:accesspoint/team-a/object/team-a/*`,
    }],
  },
});
```

## Step 5: S3 Select
> **Why:** Query a 10 GB CSV for one column WITHOUT downloading it. S3 runs the scan inside its own infra and returns only matching rows. Charged per GB *scanned* — column-oriented Parquet scans far cheaper than CSV.

```bash
aws s3api select-object-content \
  --bucket $SRC --key data.csv \
  --expression "SELECT s.name FROM S3Object s WHERE s.age > 30" \
  --expression-type SQL \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE"},"CompressionType":"NONE"}' \
  --output-serialization '{"CSV":{}}' \
  out.csv
```

Expected: `out.csv` contains only the `name` values for rows with age > 30.

## Step 6: Multi-Region Access Point
> **Why:** One global endpoint that routes each request to the closest replica. Readers in EU hit the EU replica; readers in US hit the US replica. Requires CRR already configured between the buckets.

```bash
aws s3control create-multi-region-access-point \
  --account-id $ACCT \
  --details '{
    "Name":"global-ap",
    "Regions":[{"Bucket":"src-bucket"},{"Bucket":"dst-bucket"}]
  }'
```

Apps use `accesspoint/global-ap.mrap.accesspoint.s3-global.amazonaws.com`.

## Cleanup
```bash
# Empty buckets first (autoDeleteObjects set — but MRAP/AP must be deleted manually)
aws s3control delete-multi-region-access-point --account-id $ACCT --details Name=global-ap
cdk destroy
```

## Common Errors
- **Replication configured but no copies appear** — versioning not enabled on BOTH source and destination. Replication is a no-op without it.
- **Batch Replication needed for existing objects** — replication only copies NEW objects after enablement.
- **Object Lambda returns 403** — Lambda role missing `s3-object-lambda:WriteGetObjectResponse` permission.
- **Lifecycle rule not firing** — S3 evaluates rules at most once per day at UTC midnight; transitions may lag by a day.
- **S3 Select "Invalid column name"** — CSV without headers but `FileHeaderInfo: USE` configured. Change to `NONE` and use `s._1`, `s._2`.
- **MRAP: "buckets must be in different regions"** — source and destination in same region; MRAP requires geo-distributed buckets.
