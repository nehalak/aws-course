# Walkthrough — 01 AWS Account Setup

## About this lesson
We're not setting up "a service" here — we're building the **baseline** every AWS account needs before you touch anything else. This means locking down the root user, creating a working IAM user for CDK, enabling billing alerts, and turning on CloudTrail (the audit log for every API call).

**Why it matters:** a leaked root credential = game over. A missing budget alert = surprise $10,000 bill from a runaway EKS cluster. CloudTrail off = you can't answer "who deleted the database?"

**When to do this:** day 1 of any new AWS account.
**When to skip:** never. Even for personal accounts.

## Estimated cost
- IAM, Organizations, Budgets: **free**
- CloudTrail management events: **free** (90-day history built-in; S3 storage is pennies/month for trail logs)
- Total for this lesson: **~$0.10/month** for S3 storage of trail logs

---

## Step 1: Lock the root user
> **Why:** Root has unlimited power and can't be scoped. Any compromise = full account takeover. Anthropic/AWS/every cloud vendor recommends: enable MFA, delete access keys, then never log in as root except for the ~5 tasks that require it.

1. Sign in as root → top-right → **Security credentials**.
2. **Multi-factor authentication (MFA)** → **Assign MFA device**.
   - Virtual (Authy/Google Authenticator) or YubiKey.
   - Scan QR, enter two consecutive 6-digit codes.
3. **Access keys**: if any exist for root → **Delete**. Root must never have access keys.
4. **IAM → Account settings → Password policy** → Edit:
   - Min length: 14
   - Require uppercase, lowercase, numbers, symbols
   - Password expiration: 90 days
   - Prevent reuse: last 5

## Step 2: Create IAM user `cdk-dev`
> **Why:** You need a day-to-day identity that isn't root. `AdministratorAccess` is fine for *learning*, but in a real org you'd scope tighter. The access keys you generate here go into `~/.aws/credentials` so the AWS CLI and CDK can act as you.

Console:
1. **IAM → Users → Create user** → name: `cdk-dev`.
2. **Permissions → Attach policies directly → `AdministratorAccess`**.
3. After creation → **Security credentials → Create access key** → select **CLI** → download CSV.
4. Assign MFA for this user too.

Configure CLI locally:
```bash
aws configure
# AWS Access Key ID: AKIA...
# Secret: ...
# Region: us-east-1
# Output: json

aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/cdk-dev"
}
```

## Step 3: Billing guardrails
> **Why:** AWS bills you whether you wanted the resource or not. A forgotten NAT Gateway is $32/mo. A runaway training job is $500/hr. Budget alerts save you from yourself.

1. **Account (top-right) → Account** → scroll to **IAM User and Role Access to Billing Information** → Edit → Activate.
2. **Billing → Budgets → Create budget**:
   - Template: **Monthly cost budget**
   - Budget amount: `$20`
   - Email recipient: your email
   - Thresholds: 50%, 80%, 100% of actual
3. **Cost Explorer** → Launch (takes 24h to populate).

## Step 4: Enable Organizations
> **Why:** Even a single-account setup benefits from Organizations — it's free, gives you SCPs (policies that cap what even admins can do), and is required if you ever add a second account later. Cheaper to enable now than migrate later.

1. **AWS Organizations → Create an organization** → Enable all features.
2. Leave single account for now. Explore **Policies → Service Control Policies** (read-only for now).

## Step 5: CloudTrail
> **Why:** Every API call (console, CLI, SDK) is logged. Without it you have no audit trail during an incident — "we think the leak happened Tuesday" becomes an existential problem. Management events are free for 90 days of history; sending to S3 gives you permanent storage.

1. **CloudTrail → Trails → Create trail**:
   - Name: `management-events`
   - Storage: new S3 bucket `cloudtrail-logs-<account>-<random>`
   - Enable log file validation
   - Multi-region: ON
   - Management events: Read + Write
2. Wait 10–15 min. Check the S3 bucket for `AWSLogs/…/CloudTrail/…json.gz` files.

## Verification

```bash
aws sts get-caller-identity              # shows cdk-dev
aws cloudtrail describe-trails           # shows your trail
aws budgets describe-budgets --account-id $(aws sts get-caller-identity --query Account --output text)
```

## Common Errors
- **"User is not authorized to perform: aws-portal:ViewBilling"** → You skipped Step 3.1 (enable IAM access to billing).
- **MFA prompts every CLI call** → Add `mfa_serial` to `~/.aws/config` and use `aws sts get-session-token`.
