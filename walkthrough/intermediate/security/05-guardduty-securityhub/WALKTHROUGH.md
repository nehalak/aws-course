# Walkthrough — 05 GuardDuty & Security Hub

## About this service
**GuardDuty** is AWS's managed threat detection service. It continuously analyzes **CloudTrail management events**, **VPC flow logs**, **DNS logs**, **S3 data events**, **EKS audit logs**, and **EBS volumes** (malware scan) using machine learning + threat intel feeds. Findings look like `UnauthorizedAccess:EC2/SSHBruteForce`, `CryptoCurrency:EC2/BitcoinTool.B`, `Recon:IAMUser/MaliciousIPCaller`. **Security Hub** is the aggregator — it ingests findings from GuardDuty, Inspector, Macie, IAM Access Analyzer, Config, and third-party tools, then grades your account against compliance **standards** (AWS Foundational Security Best Practices, CIS v1.4/v3, PCI DSS, NIST). Together they give you "what's on fire" (GuardDuty) + "where am I weak" (Security Hub).

**Why it matters:** manual log review doesn't scale. GuardDuty catches the 3am crypto-miner in a compromised IAM key; Security Hub tells you your S3 buckets allow public access before an attacker does.

**When to use:** every production account, multi-account orgs (delegate admin to a security account), any regulated workload.
**When NOT to use:** ephemeral sandboxes where the cost > value; dev accounts with no real data (enable at the org level on prod only).

## Estimated cost
- GuardDuty CloudTrail analysis: **$4/million events** (first 500M), drops to $2/M after
- GuardDuty VPC flow logs + DNS: **$1.00/GB** analyzed (first 500GB)
- GuardDuty S3 protection: **$0.80/million events**
- GuardDuty malware scan: **$0.05/GB scanned**
- Security Hub: **$0.0010 per security check** + **$0.00003 per finding ingested** (after free tier)
- Typical small account: **~$3–10/month** GuardDuty, **~$5–15/month** Security Hub
- Total for this lesson: **~$10–25/month** — disable after the exercise

---

## Step 1: Enable GuardDuty with CDK
> **Why:** One resource, one region, immediate threat detection on CloudTrail + VPC + DNS. Add S3 and malware-scan modules explicitly — they bill separately so you opt in.

`lib/threat-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as gd from 'aws-cdk-lib/aws-guardduty';
import { Construct } from 'constructs';

export class ThreatStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const detector = new gd.CfnDetector(this, 'Detector', {
      enable: true,
      findingPublishingFrequency: 'FIFTEEN_MINUTES',
      dataSources: {
        s3Logs: { enable: true },
        kubernetes: { auditLogs: { enable: false } },
        malwareProtection: {
          scanEc2InstanceWithFindings: { ebsVolumes: true },
        },
      },
    });

    new cdk.CfnOutput(this, 'DetectorId', { value: detector.ref });
  }
}
```

For org-wide deployment, delegate admin to a security account:
```bash
aws organizations enable-aws-service-access --service-principal guardduty.amazonaws.com
aws guardduty enable-organization-admin-account \
  --admin-account-id 222222222222
```

## Step 2: Generate sample findings
> **Why:** You need findings to build detection + response wiring. Sample findings are synthetic but identical in shape to real ones — perfect for EventBridge rule tuning.

```bash
DETECTOR=$(aws guardduty list-detectors --query "DetectorIds[0]" --output text)
aws guardduty create-sample-findings --detector-id $DETECTOR \
  --finding-types \
    "Recon:IAMUser/MaliciousIPCaller" \
    "UnauthorizedAccess:EC2/SSHBruteForce" \
    "CryptoCurrency:EC2/BitcoinTool.B!DNS"
```

List them:
```bash
aws guardduty list-findings --detector-id $DETECTOR --query "FindingIds" --output text \
  | xargs -n1 | head -5
```

Expected output:
```
5ec3fecb0a9f5... 7cb3a92e8ff41... 3a11d...
```

## Step 3: Enable Security Hub with standards
> **Why:** Security Hub is useless without standards enabled. Turn on AWS FSBP + CIS v1.4 — you get ~200 automated checks across S3, IAM, EC2, RDS.

```typescript
import * as sh from 'aws-cdk-lib/aws-securityhub';

const hub = new sh.CfnHub(this, 'Hub', {
  enableDefaultStandards: true,  // turns on AWS FSBP
});

new sh.CfnStandard(this, 'Cis', {
  standardsArn: `arn:aws:securityhub:${this.region}::standards/cis-aws-foundations-benchmark/v/1.4.0`,
});
```

After ~1 hour, check score:
```bash
aws securityhub get-findings \
  --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}]}' \
  --max-items 5 --query "Findings[].{Title:Title,Severity:Severity.Label}"
```

Expected:
```json
[
  {"Title":"S3.8 S3 Block Public Access setting should be enabled at the bucket-level","Severity":"HIGH"},
  {"Title":"EC2.7 EBS default encryption should be enabled","Severity":"MEDIUM"},
  {"Title":"IAM.3 IAM users' access keys should be rotated every 90 days","Severity":"MEDIUM"}
]
```

## Step 4: Fix three failing controls
> **Why:** Findings without remediation = dashboard theater. These three are the most common "free" wins: you flip switches and score jumps.

```bash
# 1. Block all public S3 at account level
aws s3control put-public-access-block \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# 2. Enable default EBS encryption
aws ec2 enable-ebs-encryption-by-default

# 3. Require MFA for root (check in IAM credential report)
aws iam get-account-summary --query "SummaryMap.AccountMFAEnabled"
# If 0: go to root console and enable a hardware / virtual MFA
```

Security Hub re-evaluates within 24h; findings move to `PASSED`.

## Step 5: Auto-remediate SSH brute-force
> **Why:** Detection without response = slow. Wire an EventBridge rule on the brute-force finding type to a Lambda that updates the security group. From finding to mitigation in seconds.

```typescript
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';

const blocker = new lambda.Function(this, 'SshBlocker', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  timeout: cdk.Duration.seconds(30),
  code: lambda.Code.fromInline(`
import boto3, os
ec2 = boto3.client('ec2')
SG_ID = os.environ['SG_ID']
def handler(event, _):
    detail = event['detail']
    ip = detail['service']['action']['networkConnectionAction']['remoteIpDetails']['ipAddressV4']
    print(f"Blocking {ip}")
    try:
        ec2.revoke_security_group_ingress(
            GroupId=SG_ID,
            IpPermissions=[{'IpProtocol':'tcp','FromPort':22,'ToPort':22,
                            'IpRanges':[{'CidrIp':'0.0.0.0/0'}]}])
    except Exception: pass
    ec2.authorize_security_group_ingress(
        GroupId=SG_ID,
        IpPermissions=[{'IpProtocol':'-1',
                        'IpRanges':[{'CidrIp': f'{ip}/32','Description':'AutoBlock GD'}]}])
  `),
  environment: { SG_ID: 'sg-xxxxxxxx' },
});
blocker.addToRolePolicy(new iam.PolicyStatement({
  actions: ['ec2:AuthorizeSecurityGroupIngress','ec2:RevokeSecurityGroupIngress'],
  resources: ['*'],
}));

new events.Rule(this, 'BruteForceRule', {
  eventPattern: {
    source: ['aws.guardduty'],
    detailType: ['GuardDuty Finding'],
    detail: { type: ['UnauthorizedAccess:EC2/SSHBruteForce'] },
  },
  targets: [new targets.LambdaFunction(blocker)],
});
```

Trigger the sample finding again and watch CloudWatch Logs for the Lambda — it prints `Blocking 198.51.100.42`.

## Step 6: Suppression rules
> **Why:** Noisy recurring findings (e.g., a known pen-test IP) drown signal. Suppression filters them out of the console without disabling detection.

```bash
aws guardduty create-filter --detector-id $DETECTOR \
  --name "SuppressLabBox" --action ARCHIVE \
  --finding-criteria '{
    "Criterion": {
      "resource.instanceDetails.instanceId": {"Eq": ["i-0abc123"]}
    }
  }'
```

Findings matching the filter are auto-archived (still stored, hidden from default view).

## Cleanup
```bash
# Remove the detector (costs stop immediately)
aws guardduty delete-detector --detector-id $DETECTOR

# Disable Security Hub (keeps findings for 90 days)
aws securityhub disable-security-hub

cdk destroy
```

## Common Errors
- **`BadRequestException: The request is rejected because the detector already exists`** → one detector per region. List + reuse.
- **Security Hub shows no findings for 24h** → standards take up to 24 hours for initial evaluation. Be patient.
- **EventBridge rule never fires** → event pattern case-sensitive; `type` must match exactly. Copy from a real finding JSON.
- **Bill spike after enabling** → usually VPC Flow Logs volume. Disable per-VPC enrichment or downsize.
- **Delegated admin not working in org** → must enable trusted service access first (`organizations enable-aws-service-access`).
