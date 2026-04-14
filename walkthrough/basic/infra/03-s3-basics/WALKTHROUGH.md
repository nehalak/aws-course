# Walkthrough — 03 S3 Basics

## About this service
**S3 (Simple Storage Service)** is AWS's object store. Files (called objects) live in **buckets**. Buckets are globally unique by name and region-scoped for data. S3 gives you 99.999999999% (11 nines) durability and scales to exabytes without you managing anything.

**Why it matters:** S3 is everywhere — static website hosting, data lakes, application backups, ML training datasets, logs, artifacts. It's often the cheapest durable storage AWS offers.

**When to use S3:** unstructured data, any file that's read more than written, static site hosting, data pipelines.
**When NOT to use S3:** low-latency random access (use DynamoDB or EBS), POSIX-filesystem semantics (use EFS), frequently-modified files (use EBS/EFS — S3 is write-once-read-many friendly).

## Estimated cost
- **STANDARD: $0.023/GB/mo** — hot data
- **STANDARD_IA: $0.0125/GB/mo** — infrequent, + retrieval charges
- **GLACIER: $0.004/GB/mo** — archive, + retrieval hours
- **Requests**: $0.0004/1000 GET, $0.005/1000 PUT (negligible unless at scale)
- **Data transfer out to internet**: $0.09/GB (this is the real cost trap)
- Total for this lesson: **<$1/month** with small test files

---

## Step 1: Stack
> **Why:** This single construct sets up production-grade defaults: encrypted at rest, versioned, public access blocked, lifecycle tiering to save cost. Memorize this pattern.

`lib/s3-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class S3Stack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new s3.Bucket(this, 'LearnBucket', {
      versioned: true,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [{
        id: 'tier-and-expire',
        enabled: true,
        transitions: [{
          storageClass: s3.StorageClass.INTELLIGENT_TIERING,
          transitionAfter: cdk.Duration.days(30),
        }],
        noncurrentVersionExpiration: cdk.Duration.days(90),
      }],
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });
  }
}
```

```bash
cdk deploy
BUCKET=$(aws s3 ls | grep learnbucket | awk '{print $3}')
echo $BUCKET
```

## Step 2: Upload + version
> **Why:** Versioning is critical — without it, `PUT` overwrites irrevocably. With it, you can restore any prior version. Many ransomware/mistake scenarios are solved by versioning alone.

```bash
echo "v1" > file.txt && aws s3 cp file.txt s3://$BUCKET/
echo "v2" > file.txt && aws s3 cp file.txt s3://$BUCKET/

aws s3api list-object-versions --bucket $BUCKET --query 'Versions[].[Key,VersionId,IsLatest]' --output table
```

## Step 3: Restore prior version
> **Why:** The restore is a copy-in-place of an older version — this is the recovery move during an incident. Rehearse it now.

```bash
OLD_VERSION=$(aws s3api list-object-versions --bucket $BUCKET \
  --query 'Versions[?IsLatest==`false`].VersionId' --output text)
aws s3api copy-object --bucket $BUCKET --key file.txt \
  --copy-source "$BUCKET/file.txt?versionId=$OLD_VERSION"
aws s3 cp s3://$BUCKET/file.txt -  # prints "v1"
```

## Step 4: Storage class comparison
> **Why:** Picking the right storage class is 80% of S3 cost optimization. Feel the difference in pricing; set up lifecycle rules appropriately in real projects.

```bash
dd if=/dev/urandom of=1gb.bin bs=1M count=1024
aws s3 cp 1gb.bin s3://$BUCKET/std/ --storage-class STANDARD
aws s3 cp 1gb.bin s3://$BUCKET/ia/  --storage-class STANDARD_IA
aws s3 cp 1gb.bin s3://$BUCKET/gla/ --storage-class GLACIER
```

## Step 5: Presigned URL
> **Why:** Presigned URLs are how you let users upload/download without giving them AWS credentials. Core pattern for web apps with user files.

```bash
aws s3 presign s3://$BUCKET/file.txt --expires-in 300
# copy URL, curl it → works
# wait 5 min → 403
```

## Step 6: Static website
> **Why:** Simplest possible static hosting. For real production, you'd put CloudFront in front — but knowing the raw mechanism matters.

```bash
cat > index.html <<EOF
<html><body><h1>static site</h1></body></html>
EOF
aws s3 cp index.html s3://$BUCKET/
aws s3 website s3://$BUCKET/ --index-document index.html
```

## Cleanup
> **Why:** `cdk destroy` only deletes the bucket if empty. Versioned buckets keep old versions + delete markers. Clean those first.

```bash
aws s3 rm s3://$BUCKET --recursive
aws s3api delete-objects --bucket $BUCKET \
  --delete "$(aws s3api list-object-versions --bucket $BUCKET \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
cdk destroy
```

## Common Errors
- **`BucketAlreadyExists`** — name is global. Add suffix.
- **`AccessDenied` on presigned URL** — signed with IAM that lacks `GetObject`, or URL expired.
- **Lifecycle not triggering** — rules run once daily, not immediately.
