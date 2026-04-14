# Walkthrough — 04 Compliance

## About this service
Compliance on AWS is a set of primitives, not a checkbox. **AWS Artifact** hands you AWS's own audit reports (SOC 2, ISO, PCI AoC). **AWS Audit Manager** automates evidence collection for your controls. **AWS Config** + **Conformance Packs** continuously evaluate resource state against rule sets (PCI, HIPAA, NIST 800-53). **Service Control Policies (SCPs)** in Organizations prevent non-compliant actions from ever succeeding. The core idea is **Shared Responsibility**: AWS is compliant for what's "of" the cloud; you are compliant for what you build "in" the cloud.

**Why it matters:** You cannot pass SOC 2 / PCI / HIPAA audits by pointing at AWS's certifications — auditors want *your* evidence for *your* workload. Audit Manager + Config automate 60-80% of evidence; the rest is policy/procedure docs.
**When to use:** Any customer-facing business handling regulated data (payments → PCI DSS, health → HIPAA, EU PII → GDPR, enterprise SaaS → SOC 2).
**When NOT to use:** Side projects. Audit Manager costs real money and is overkill for "I want my SaaS to look secure."

## Estimated cost
- AWS Artifact: **free**
- AWS Config: **$0.003 per configuration item recorded** + **$0.001 per rule evaluation** (can be **$50-500/month** in a busy account)
- Audit Manager: **$1.25 per resource assessed per month** (a 200-resource account ≈ $250/month)
- Conformance Pack: no extra Audit Manager cost, just Config costs
- SCPs: free
- Total for this lesson (small lab account): **~$30/month** — Audit Manager dominates; disable when done

---

## Step 1: Download AWS Artifact reports and read the CRM
> **Why:** The Customer Responsibility Matrix (CRM) in each report tells you *exactly* which controls AWS covers and which are yours. Auditors ask you to produce this — if you skip, you inherit controls you didn't realize you owned.

```bash
# Console path — Artifact doesn't have a full CLI for downloads
# Sign in → AWS Artifact → Reports → "SOC 2 Type II" → Agreement accept → Download PDF
```

Open the SOC 2 Type II report. Skim sections:
- **Scope:** which regions and services are in scope this period. If you use a service NOT in scope, that service isn't SOC-2-covered for you.
- **Complementary User Entity Controls (CUECs):** the controls the report assumes *you* have. Examples: "user provisioning/deprovisioning", "data classification", "log retention policies". These become your own control list.

Also grab: PCI DSS 4.0 AoC (if processing cards), HIPAA BAA (requires executing a BAA first — Artifact → Agreements → AWS BAA).

## Step 2: Enable Audit Manager and add the PCI DSS framework
> **Why:** Audit Manager runs assessments: it maps framework controls (e.g., "PCI Req 10.2.1 – all access to cardholder data is logged") to automated data sources (CloudTrail, Config rules, Security Hub findings) and collects evidence on a schedule. At audit time you export a PDF of evidence — dramatically faster than a spreadsheet + screenshots approach.

```bash
aws auditmanager register-account --kms-key alias/audit-manager

# List available frameworks
aws auditmanager list-assessment-frameworks --framework-type Standard \
  --query 'frameworkMetadataList[?contains(name, `PCI`)]'

# Create an assessment (reusing the built-in PCI DSS 4.0 framework)
aws auditmanager create-assessment \
  --name "PCI-DSS-Q2-2026" \
  --framework-id <framework-id> \
  --assessment-reports-destination "destination=s3://audit-evidence-bucket,destinationType=S3" \
  --scope '{"awsAccounts":[{"id":"123456789012"}],"awsServices":[{"serviceName":"ec2"},{"serviceName":"s3"},{"serviceName":"rds"}]}' \
  --roles '[{"roleType":"PROCESS_OWNER","roleArn":"arn:aws:iam::123456789012:role/ComplianceOwner"}]'
```

Wait 24h. Open **Console → Audit Manager → Assessments → your assessment → Controls**. For each control you'll see "Automated evidence" (CloudTrail events, Config snapshots, Security Hub findings) auto-populating. For "Manual evidence" controls (e.g., "background checks on employees"), you upload PDFs/attestations yourself.

Generate a report:
```bash
aws auditmanager create-assessment-report --name "PCI-Q2-Evidence" \
  --assessment-id <id> --description "Q2 2026 evidence pack"
# PDF lands in the S3 destination bucket
```

## Step 3: HIPAA-eligible architecture — paper design
> **Why:** HIPAA isn't a flag you turn on. It's an architectural obligation: encrypt PHI everywhere, log every access, restrict tenancy (for some interpretations), and use **only HIPAA-eligible services**. The BAA is the legal doc, but architecture is where you actually comply.

**Constraints:**
1. **BAA signed** with AWS via Artifact first. Without it, any PHI you store is a breach.
2. **Only HIPAA-eligible services** — the list is maintained by AWS (~130 services as of 2026, including S3, RDS, DynamoDB, Lambda, ECS, EKS, API Gateway, KMS, CloudFront). Services NOT on the list (e.g., some new-launch services, Workmail in certain regions) cannot touch PHI.
3. **Encryption at rest**: every PHI-bearing store uses KMS CMK with key rotation. No SSE-S3 only — use SSE-KMS.
4. **Encryption in transit**: TLS 1.2+ for every hop. Enforce via ALB/NLB TLS policies and S3 bucket policies requiring `aws:SecureTransport`.
5. **Access logging**: CloudTrail org-wide with log file validation; S3 access logs for PHI buckets; VPC Flow Logs to CloudWatch/S3.
6. **Least privilege**: IAM roles scoped by `aws:ResourceTag`, no `*` actions on PHI buckets/tables.
7. **Dedicated tenancy** (optional, per your risk appetite): EC2 dedicated instances. Most customers skip this since AWS BAA covers shared tenancy.

**Reference diagram:**
```
   ┌────────────────────────────────────────────────────────┐
   │  VPC (PHI-tier, flow-logs-enabled)                     │
   │                                                        │
   │   ALB (TLS 1.2+)  →  ECS Fargate (app)  →  RDS Aurora  │
   │   WAF in front        KMS-encrypted disks   KMS-encrypted│
   │                       task role scoped      Performance │
   │                       to rds:Connect only   Insights off│
   │                                                        │
   │   All traffic logged: CloudTrail + VPC FL + ALB access │
   └────────────────────────────────────────────────────────┘
        │                                     │
        ▼                                     ▼
   S3 (PHI, Object Lock, SSE-KMS,       KMS CMK (7y rotation,
   bucket policy deny !SecureTransport) key policy MFA for delete)

   All of the above under an Org SCP that denies non-eligible services.
```

**What makes this HIPAA-eligible in practice:**
- Every service used is on the AWS HIPAA-eligible list.
- PHI never flows to a non-eligible service (e.g., no Kinesis Data Analytics if it's not eligible in your region).
- BAA in place.
- Encryption + logging uniformly applied.

## Step 4: SCP for GDPR data residency (EU-only PII)
> **Why:** GDPR Article 44 restricts cross-border transfer of EU personal data. Policy-as-code via SCP prevents an engineer from *accidentally* creating S3 replication to us-east-1, or spinning up RDS in ap-south-1, when the workload handles EU PII. SCPs are enforced at the Organization level — they override any identity-level allow.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonEURegions",
      "Effect": "Deny",
      "Action": [
        "s3:CreateBucket", "s3:PutBucketReplication",
        "rds:CreateDBInstance", "rds:CreateDBCluster",
        "dynamodb:CreateTable", "dynamodb:CreateGlobalTable",
        "ec2:RunInstances",
        "kms:CreateKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["eu-west-1", "eu-central-1", "eu-north-1", "eu-west-3"]
        },
        "StringEquals": {
          "aws:ResourceTag/DataClass": "EU-PII"
        }
      }
    },
    {
      "Sid": "DenyGlobalEndpointServices",
      "Effect": "Deny",
      "Action": [
        "cloudfront:CreateDistribution",
        "globalaccelerator:CreateAccelerator"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/DataClass": "EU-PII" }
      }
    }
  ]
}
```

Attach to the OU that owns EU workloads:
```bash
aws organizations create-policy --type SERVICE_CONTROL_POLICY \
  --name EU-PII-Residency --content file://scp.json

aws organizations attach-policy --policy-id p-abc --target-id ou-eu
```

## Step 5: Config Conformance Pack for PCI + remediation
> **Why:** Conformance packs bundle many Config rules mapped to a standard. You deploy once, and every resource is continuously checked. Bonus: attach remediation via SSM Automation so failing resources auto-fix.

```bash
# Deploy the sample PCI conformance pack AWS ships
aws configservice put-conformance-pack \
  --conformance-pack-name PCI-DSS-v4 \
  --template-s3-uri s3://awsconfigconforms-samples-us-east-1/pci-dss-v4.yaml \
  --delivery-s3-bucket aws-config-conforms-<acct>
```

Wait ~30 min. Check the dashboard:
```bash
aws configservice describe-conformance-pack-compliance \
  --conformance-pack-name PCI-DSS-v4
```

Pick 3 failing rules and remediate:

**Rule 1: `s3-bucket-server-side-encryption-enabled`**
```bash
aws configservice put-remediation-configurations --remediation-configurations '[{
  "ConfigRuleName":"s3-bucket-server-side-encryption-enabled",
  "TargetType":"SSM_DOCUMENT",
  "TargetId":"AWS-EnableS3BucketEncryption",
  "Parameters":{
    "BucketName":{"ResourceValue":{"Value":"RESOURCE_ID"}},
    "SSEAlgorithm":{"StaticValue":{"Values":["AES256"]}}
  },
  "Automatic":true,
  "MaximumAutomaticAttempts":3
}]'
```

**Rule 2: `restricted-ssh`** (security groups that allow 0.0.0.0/0 on 22) — use `AWS-DisablePublicAccessForSecurityGroup`.

**Rule 3: `rds-storage-encrypted`** — can't remediate a running DB in place; emit a finding to Security Hub and require a migration.

Target: conformance pack compliance > 80% in a week.

## Step 6: PCI scope reduction patterns
> **Why:** "PCI scope" = everything that stores/processes/transmits cardholder data. Every in-scope resource is your audit burden. The cheapest compliance strategy is to reduce scope — by outsourcing the card data itself.

**Pattern 1: Stripe / Braintree iframe tokenization**
Browser → Stripe iframe captures card → Stripe returns a token to your backend. Your backend never sees PAN. Your PCI scope drops from "full CDE" to **SAQ A** (the smallest questionnaire).

**Pattern 2: Network tokenization (Visa/MC Token Service)**
Card is replaced with a merchant-scoped token by the network. Your vault stores tokens, not PANs. Re-authorization without touching the real card.

**Pattern 3: Dedicated CDE account + AWS PrivateLink**
If you *must* store PAN, put the CDE in a dedicated AWS account, no internet gateway, access via PrivateLink from the app account only. The rest of your infra stays out of scope.

**Pattern 4: Point-to-point encryption (P2PE)**
Card terminal encrypts at swipe with a key you don't have. PAN exits your environment already-encrypted. Drops physical store scope to **SAQ P2PE**.

Pick the right pattern before writing code — "we'll do PCI later" is how you end up with 80 in-scope servers.

## Cleanup
```bash
# Audit Manager assessments bill per resource per month
aws auditmanager delete-assessment --assessment-id <id>
aws auditmanager deregister-account

aws configservice delete-conformance-pack --conformance-pack-name PCI-DSS-v4
# Keep SCPs — they're free.
cdk destroy
```

## Common Errors
- **"HIPAA eligible" ≠ "HIPAA compliant"** → eligible = AWS will sign a BAA for that service. Compliance is your architecture + policies. Don't confuse.
- **BAA not executed → PHI stored → breach** → sign the BAA in Artifact *before* any PHI touches the account.
- **SCP blocks the Org admin too** → SCPs don't apply to the management account, but they do apply to all member accounts — including your break-glass admin role. Keep a management-account-only operator.
- **Conformance Pack stuck `CREATE_IN_PROGRESS`** → Config recorder isn't enabled in the account/region. Enable Config first.
- **Audit Manager evidence missing** → Security Hub and CloudTrail must be on; Audit Manager pulls from them. First evidence takes ~24h to show.
- **Cross-region data transfer despite SCP** → the SCP denies *new* resources; existing replication kept running. Audit existing resources during remediation.
- **GDPR "adequate protection" for US transfers** → SCP blocks regions, but DPAs + SCCs (Standard Contractual Clauses) are the legal layer. Talk to legal.
