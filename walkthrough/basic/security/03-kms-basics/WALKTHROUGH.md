# Walkthrough — 03 KMS Basics

## About this service
**KMS (Key Management Service)** is AWS's managed encryption key service. You create **CMKs (Customer Master Keys)** — actually never leave the KMS HSM boundary. KMS uses **envelope encryption**: a data key encrypts your data; KMS encrypts the data key. This lets you encrypt huge blobs without round-tripping to KMS for every byte.

**Why it matters:** compliance (SOC2, HIPAA, PCI) effectively requires encryption at rest with customer-controlled keys. KMS integrates with S3, EBS, RDS, Lambda env vars, Secrets Manager — everywhere.

**When to use KMS:** always, for data at rest. Even when not required, encrypt-by-default is cheap insurance.
**When NOT to manage your own keys:** most use cases can use AWS-managed keys (free tier of KMS, you don't see them). Create your own CMKs only when you need key policies, rotation, cross-account sharing, or audit trails for key use.

## Estimated cost
- **CMK: $1/month per key**
- **Requests: $0.03 per 10,000 requests** (Encrypt/Decrypt/GenerateDataKey)
- **Rotation: free** (auto-rotated yearly if enabled)
- **AWS-managed keys: free** (the ones like `aws/s3`, `aws/rds`)
- Total for this lesson: **~$1/month** per CMK you create

---

## Step 1: CDK key
> **Why:** `enableKeyRotation` rotates underlying material yearly — transparent to users of the key. Always on for production. `pendingWindow` means you have 7 days to cancel a delete (don't accidentally nuke keys).

```typescript
import * as kms from 'aws-cdk-lib/aws-kms';
import * as iam from 'aws-cdk-lib/aws-iam';

const key = new kms.Key(this, 'AppKey', {
  enableKeyRotation: true,
  pendingWindow: cdk.Duration.days(7),
  alias: 'alias/app-data',
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

key.grantEncryptDecrypt(new iam.AccountPrincipal(cdk.Stack.of(this).account));
```

## Step 2: Encrypt/decrypt via CLI
> **Why:** Shows the primitive ops. In practice, SDKs handle this; but you should know the direct API.

```bash
KEY_ID=$(aws kms describe-key --key-id alias/app-data --query 'KeyMetadata.KeyId' --output text)

aws kms encrypt --key-id $KEY_ID --plaintext "hello" \
  --query CiphertextBlob --output text | base64 -d > enc.bin

aws kms decrypt --ciphertext-blob fileb://enc.bin \
  --query Plaintext --output text | base64 -d
# hello
```

## Step 3: S3 with SSE-KMS
> **Why:** The most common KMS integration. Shows how S3 + KMS combine — you need BOTH S3 permissions AND KMS permissions. This is where people get stuck.

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

const bucket = new s3.Bucket(this, 'KmsBucket', {
  encryption: s3.BucketEncryption.KMS,
  encryptionKey: key,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
});
```

Modify key policy to deny your own decrypt → S3 GetObject fails despite S3 permissions allowing it. Lesson: KMS is a second gate.

## Step 4: Grants
> **Why:** Grants are like "temp permission slips" for short-lived principals (typically Lambda execution contexts). Revocable independently of the key policy.

```bash
GRANT_TOKEN=$(aws kms create-grant --key-id $KEY_ID \
  --grantee-principal arn:aws:iam::<acct>:role/some-lambda-role \
  --operations Decrypt \
  --query 'GrantToken' --output text)
aws kms revoke-grant --key-id $KEY_ID --grant-id <grant-id>
```

## Step 5: Alias use
> **Why:** Aliases are stable pointers to keys. Reference `alias/app-data` in code; swap the underlying key without touching code. Important for key rotation beyond the automatic yearly.

```typescript
const reuseKey = kms.Alias.fromAliasName(this, 'K', 'alias/app-data');
```

## Cleanup
```bash
cdk destroy     # CMK enters 7-day pending delete
```

To cancel deletion:
```bash
aws kms cancel-key-deletion --key-id $KEY_ID
```

## Common Errors
- **`AccessDenied: kms:GenerateDataKey`** — missing on the KMS key policy; S3 needs it.
- **Key policy lockout** — if you remove all principals, only root can fix (via AWS support).
- **KMS costs sneak up** — $1/CMK/month + per-request.
