# Walkthrough — 02 CloudTrail

## About this service
**CloudTrail** is AWS's API audit log. Every AWS API call (console click, CLI command, SDK call) is recorded: who, when, what, from where. Trails send records to S3 (permanent) and optionally CloudWatch Logs (searchable).

- **Management events**: API calls modifying your AWS environment (e.g., `CreateBucket`). Free, 90-day history by default.
- **Data events**: operations on data (`S3 PutObject`, `Lambda Invoke`, `DDB GetItem`). Not free, can be huge.
- **Insights events**: ML-detected anomalies (sudden API error spike).

**Why it matters:** incident response ("who deleted this?"), compliance (SOC2, PCI require audit trails), security (detect compromised credentials by unusual API patterns).

**When to use CloudTrail:** always — management trail in every account. It's effectively required for any serious work.
**When NOT to enable data events:** unless you need them. Can 100x your CloudTrail bill.

## Estimated cost
- **Management events to S3: free** (first copy)
- **Data events: $0.10 per 100k events** (adds up fast on busy S3/Lambda)
- **S3 storage of trail logs: ~pennies/month**
- **CloudTrail Insights: $0.35 per 100k management events analyzed**
- Total for this lesson: **<$1/month** (management events only)

---

## Step 1: Trail
> **Why:** Multi-region + log file validation are defaults you should always set. Without them, a compromised account in one region could evade auditing, and tampering would be harder to detect.

```typescript
import * as cloudtrail from 'aws-cdk-lib/aws-cloudtrail';
import * as s3 from 'aws-cdk-lib/aws-s3';

const bucket = new s3.Bucket(this, 'TrailBucket', {
  encryption: s3.BucketEncryption.S3_MANAGED,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});

new cloudtrail.Trail(this, 'MgmtTrail', {
  bucket,
  isMultiRegionTrail: true,
  enableFileValidation: true,
  sendToCloudWatchLogs: true,
  cloudWatchLogsRetention: cloudtrail.logs.RetentionDays.ONE_MONTH,
});
```

## Step 2: Generate events
> **Why:** You need events to query. Any read-only API call you make ends up in the trail. Wait time is built in — CloudTrail is near-real-time but not instant.

```bash
aws ec2 describe-instances >/dev/null
aws s3 ls >/dev/null
aws iam list-users >/dev/null
```

Wait 10 min.

## Step 3: Query via Athena
> **Why:** Trail logs are compressed JSON in S3 — painful to grep. Athena lets you SQL them. This is how real incident response works: "give me every API call from IP X in the last 7 days".

Console → **CloudTrail → Event history → Create Athena table**.

```sql
SELECT eventTime, userIdentity.userName, eventName, sourceIPAddress, awsRegion
FROM cloudtrail_logs
WHERE eventName = 'DeleteBucket'
  AND eventTime > date_add('day', -7, current_date)
ORDER BY eventTime DESC;
```

## Step 4: Data events (ONE bucket)
> **Why:** Data events are incredibly detailed — every S3 GET is logged. Too much for the whole account, but invaluable on a sensitive bucket. Enable selectively.

Console → **CloudTrail → Trails → MgmtTrail → Edit → Data events** → pick one bucket, Read + Write.

```bash
aws s3 cp README.md s3://<tracked-bucket>/
```

## Step 5: Insights
> **Why:** Insights ML-detects anomalies (sudden API error rate jump = possible breach or misconfiguration). Not real-time but great for post-hoc analysis.

Enable **CloudTrail Insights**. Burst a bunch of API calls. Check next day.

## Cleanup
- Turn OFF data events (they cost).
- Keep management trail running; it's cheap and valuable.

## Common Errors
- **Athena `Permission denied`** — workgroup needs S3 read + Glue catalog access.
- **Events not appearing** — 5–15 min lag normal.
- **Insights charges surprise** — $0.35 per 100k events analyzed.
