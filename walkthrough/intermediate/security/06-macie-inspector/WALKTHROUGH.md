# Walkthrough — 06 Macie & Inspector

## About this service
**Macie** is a managed service that scans S3 for **sensitive data** (PII, PHI, credentials). It uses ML + 150+ managed data identifiers (SSN, credit cards, passports, AWS keys) plus **custom identifiers** you write as regex. It runs **discovery jobs** (one-off or scheduled) and an always-on **automated sensitive data discovery** sampler. **Inspector** is the vulnerability scanner for compute: it continuously scans **EC2 instances** (SSM-agent-based), **ECR container images**, and **Lambda functions + layers** for CVEs and (Lambda) code issues, then produces findings with CVSS scores and "fix available in version X" guidance. Both services forward findings to **Security Hub** for unified review.

**Why it matters:** you can't protect data you don't know you have. Macie surfaces the forgotten CSV of customer SSNs in some old bucket. Inspector finds the `log4j` in your container before someone else does.

**When to use:** Macie — any account storing customer/employee data in S3; compliance (HIPAA, PCI, GDPR DSAR workflows). Inspector — every account running containers, EC2, or Lambda.
**When NOT to use:** Macie at 100% scan on huge buckets (cost explodes — sample first). Inspector on throw-away sandboxes.

## Estimated cost
- **Macie** account eval (S3 inventory): **~$0.10/month per 10k objects**
- **Macie** sensitive data discovery: **$1.00/GB** scanned (first 50 TB)
- **Inspector EC2**: **$1.258 per instance per month**
- **Inspector ECR**: **$0.09 per image scan** (initial) + **$0.01 per rescan**
- **Inspector Lambda**: **$0.30 per function/month** (standard), **$1.25** with code scanning
- Typical small account (this lesson): **~$5–15/month** — disable services after exercise

---

## Step 1: Enable Macie + run a classification job
> **Why:** Macie off = you don't know what's in your buckets. Running a one-off job on a single bucket is the cheapest way to see the service work without the always-on scanner racking up cost.

`lib/macie-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as macie from 'aws-cdk-lib/aws-macie';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class MacieStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new macie.CfnSession(this, 'Session', {
      status: 'ENABLED',
      findingPublishingFrequency: 'FIFTEEN_MINUTES',
    });

    const bucket = new s3.Bucket(this, 'PiiBucket', {
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });
    new cdk.CfnOutput(this, 'Bucket', { value: bucket.bucketName });
  }
}
```

Upload a synthetic PII CSV:
```bash
cat > /tmp/customers.csv <<'EOF'
name,ssn,email,card
Alice Example,123-45-6789,alice@example.com,4111 1111 1111 1111
Bob Example,987-65-4321,bob@example.com,5500 0000 0000 0004
EOF
aws s3 cp /tmp/customers.csv s3://$BUCKET/customers.csv
```

Create a classification job:
```bash
aws macie2 create-classification-job \
  --job-type ONE_TIME \
  --name "pii-scan-lab" \
  --s3-job-definition '{"bucketDefinitions":[{"accountId":"'$ACCT'","buckets":["'$BUCKET'"]}]}'
```

After ~5 minutes:
```bash
aws macie2 list-findings --query "findingIds" --output text | xargs -n1 | head -3
aws macie2 get-findings --finding-ids <id> \
  --query "findings[0].classificationDetails.result.sensitiveData"
```

Expected output:
```json
[
  {"category":"PERSONAL_INFORMATION","totalCount":4,
   "detections":[{"type":"USA_SOCIAL_SECURITY_NUMBER","count":2},
                 {"type":"CREDIT_CARD_NUMBER","count":2}]}
]
```

## Step 2: Custom data identifier
> **Why:** Managed identifiers cover common PII. Your internal employee IDs / customer IDs / API token formats need custom regex. This is how you teach Macie your schema.

```bash
aws macie2 create-custom-data-identifier \
  --name "employee-id" \
  --regex '^EMP-[0-9]{6}$' \
  --keywords "employee,staff" \
  --maximum-match-distance 50
```

Upload a file containing `EMP-123456` and rerun the job — findings include the custom identifier.

## Step 3: Enable Inspector for EC2 + ECR + Lambda
> **Why:** One command, three scan surfaces. Inspector auto-discovers resources — no agent config beyond SSM. Findings populate within minutes.

```typescript
import * as inspector from 'aws-cdk-lib/aws-inspectorv2';

new inspector.CfnCisScanConfiguration(this, 'Insp', {
  // Just enabling is usually done via SDK; use CLI:
} as any);
```

Via CLI (CFN for Inspector v2 is limited; CLI is canonical):
```bash
aws inspector2 enable --resource-types EC2 ECR LAMBDA LAMBDA_CODE
```

Expected:
```json
{"accounts":[{"accountId":"111111111111","resourceStatus":{
  "ec2":"ENABLING","ecr":"ENABLING","lambda":"ENABLING","lambdaCode":"ENABLING"}}]}
```

Launch a vulnerable EC2 (Ubuntu 18.04 unpatched AMI) with SSM agent:
```bash
aws ec2 run-instances --image-id ami-0old --instance-type t3.micro \
  --iam-instance-profile Name=SSMInstanceProfile ...
```

Within ~15 minutes:
```bash
aws inspector2 list-findings --filter-criteria '{
  "resourceType":[{"comparison":"EQUALS","value":"AWS_EC2_INSTANCE"}]
}' --max-results 3 --query "findings[].{Title:title,Sev:severity,Cve:packageVulnerabilityDetails.vulnerabilityId}"
```

Expected output:
```json
[
  {"Title":"CVE-2023-4911 in glibc","Sev":"HIGH","Cve":"CVE-2023-4911"},
  {"Title":"CVE-2022-3786 in openssl","Sev":"HIGH","Cve":"CVE-2022-3786"}
]
```

## Step 4: Push a vulnerable ECR image
> **Why:** Most prod workloads ship containers. Old base images silently rot — Inspector shows you the CVE count before your CISO does.

```bash
REPO=$(aws ecr create-repository --repository-name vuln-demo --query "repository.repositoryUri" --output text)
aws ecr get-login-password | docker login --username AWS --password-stdin $REPO

cat > Dockerfile <<'EOF'
FROM node:14-buster
RUN apt-get update && apt-get install -y curl
EOF
docker build -t $REPO:old .
docker push $REPO:old
```

Wait ~2 min, then:
```bash
aws inspector2 list-findings --filter-criteria '{
  "resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_CONTAINER_IMAGE"}]
}' --query "length(findings)"
# 50+ CVEs for an old Buster + Node 14 base
```

## Step 5: Export findings to Security Hub
> **Why:** Single pane of glass. Without this you'd hop between three consoles; with it, everything flows into Security Hub scoring + your existing EventBridge workflow.

Inspector v2 → Security Hub integration is automatic if Security Hub is enabled in the same region. Verify:
```bash
aws securityhub get-findings \
  --filters '{"ProductName":[{"Value":"Inspector","Comparison":"EQUALS"}]}' \
  --max-items 3 --query "Findings[].Title"
```

For Macie, same — it auto-publishes to Security Hub when both are on.

## Step 6: Remediate by rebuilding
> **Why:** Closing the loop. Update base image, push, and watch findings go from OPEN to CLOSED. This is the proof that Inspector is actually useful.

```bash
sed -i 's/node:14-buster/node:20-bookworm-slim/' Dockerfile
docker build -t $REPO:new .
docker push $REPO:new

# Wait ~2 min for rescan
aws inspector2 list-findings --filter-criteria '{
  "ecrImageTags":[{"comparison":"EQUALS","value":"new"}]
}' --query "length(findings)"
# 5-10 instead of 50+
```

Old image findings move to `status: CLOSED` once the image is deleted.

## Cleanup
```bash
aws inspector2 disable --resource-types EC2 ECR LAMBDA LAMBDA_CODE
aws macie2 update-macie-session --status PAUSED
# or fully:
aws macie2 disable-macie

aws ecr delete-repository --repository-name vuln-demo --force
cdk destroy
```

## Common Errors
- **Macie job stays `RUNNING` forever** → session paused / suspended; re-enable with `update-macie-session --status ENABLED`.
- **Macie finds nothing on a known-PII file** → file is encrypted with SSE-KMS and Macie role lacks `kms:Decrypt`. Grant the Macie service role.
- **Inspector EC2 shows `UNSUPPORTED_OS`** → SSM agent missing / instance not in managed instances. Fix IAM profile, restart agent.
- **ECR findings don't appear** → repo scan-on-push disabled, or image not tagged. Trigger manual scan: `aws ecr start-image-scan`.
- **Lambda findings missing for a new function** → Inspector discovers new functions within 15 minutes; or the function runtime isn't supported (check the supported-runtime list).
- **Cost spike from Macie** → you enabled automated sensitive data discovery on a 10TB bucket. Switch to job-based scanning on a subset.
