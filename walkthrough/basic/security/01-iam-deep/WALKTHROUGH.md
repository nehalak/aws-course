# Walkthrough — 01 IAM Deep

## About this service
**IAM (Identity and Access Management)** is AWS's permission system. Identities (**users**, **roles**, **groups**) get **policies** (JSON documents) that allow or deny actions on resources. Every AWS API call is checked against IAM.

**Why it matters:** IAM is the first line of defense. Misconfigured IAM = data breach. Overly strict IAM = dev team can't work. Almost every AWS problem touches IAM somewhere.

**When to use what:**
- **Users**: humans logging into console. Prefer SSO/Identity Center over IAM users in production.
- **Roles**: workloads (EC2, Lambda) or cross-account access. Default choice — no long-lived keys.
- **Groups**: bundle policies for collections of users.

**When NOT to use IAM users:** for real companies, federate with SSO/Okta/Azure AD via Identity Center. IAM users are fine for learning, third-party integrations, break-glass.

## Estimated cost
- IAM: **100% free** (users, roles, policies, all of it)
- Total: **$0.00**

---

## Step 1: Write a scoped S3 policy
> **Why:** Policies are the primitive. Reading/writing JSON policy is the most important skill in AWS security. Conditions (IP restrictions, MFA required) are how you add defense-in-depth.

`ip-scoped-reports.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/reports/*",
    "Condition": {
      "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
    }
  }]
}
```

## Step 2: User + group + attach
> **Why:** Groups are how you manage permissions at scale. Attaching policies to groups (not individual users) means onboarding/offboarding is trivial.

```bash
aws iam create-group --group-name readers
aws iam attach-group-policy --group-name readers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam create-user --user-name alice
aws iam add-user-to-group --user-name alice --group-name readers

aws iam create-login-profile --user-name alice --password "TempPass!23" --password-reset-required
```

Sign in as alice in incognito → S3 read works, put fails.

## Step 3: EC2 role
> **Why:** Roles eliminate long-lived credentials on servers. The EC2 instance gets temporary creds via the metadata service — rotated automatically, scoped to the role's policy. Never put access keys on an EC2 again.

CDK:
```typescript
const role = new iam.Role(this, 'EC2ReadS3', {
  assumedBy: new iam.ServicePrincipal('ec2.amazonaws.com'),
  managedPolicies: [
    iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonS3ReadOnlyAccess'),
    iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSSMManagedInstanceCore'),
  ],
});

const instance = new ec2.Instance(this, 'Box', { role, /*...*/ });
```

SSM into instance:
```bash
aws s3 ls            # works without configuring keys
aws sts get-caller-identity    # shows "assumed-role/EC2ReadS3/..."
```

## Step 4: AssumeRole
> **Why:** Role assumption is the primitive for cross-account access and privilege escalation workflows ("I'm a dev; assume the deployer role only when deploying"). Session credentials expire — a big safety win.

Create a role trusted by your user:
```typescript
const deployRole = new iam.Role(this, 'DeployRole', {
  assumedBy: new iam.AccountPrincipal(cdk.Stack.of(this).account),
  managedPolicies: [iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonS3ReadOnlyAccess')],
});
```

Assume:
```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::<acct>:role/DeployRole \
  --role-session-name test \
  --query 'Credentials')
export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.SessionToken')

aws sts get-caller-identity    # shows assumed-role/DeployRole/test
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

## Step 5: Identity vs resource policy
> **Why:** Two policy types that grant access: identity policies (attached to users/roles) and resource policies (attached to things like S3 buckets). Both can grant; explicit `Deny` always wins. Misunderstanding this causes hours of "but I granted permission" debugging.

```bash
aws s3api put-bucket-policy --bucket $BUCKET --policy '{
  "Version":"2012-10-17",
  "Statement":[{
    "Sid":"AllowAlice","Effect":"Allow",
    "Principal":{"AWS":"arn:aws:iam::<acct>:user/alice"},
    "Action":"s3:GetObject",
    "Resource":"arn:aws:s3:::'"$BUCKET"'/*"
  }]
}'
```

## Step 6: Policy Simulator
> **Why:** Trial-and-error testing IAM in production = risky. The simulator tells you exactly which statement matched (allow/deny).

Console → **IAM → Policy simulator**. Select alice → add action `s3:PutObject` → resource `arn:aws:s3:::my-bucket/reports/x.txt` → **Run**.

## Cleanup
```bash
aws iam remove-user-from-group --group-name readers --user-name alice
aws iam delete-login-profile --user-name alice
aws iam delete-user --user-name alice
aws iam detach-group-policy --group-name readers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-group --group-name readers
cdk destroy
```

## Common Errors
- **`AccessDenied` on assume-role** — trust policy missing principal or MFA required.
- **Explicit Deny wins** — an allow policy doesn't override a deny.
- **User can't sign in** — `create-login-profile` didn't run or password policy rejects default.
