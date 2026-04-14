# Walkthrough — 01 IAM Advanced

## About this service
**Advanced IAM** goes beyond basic users/roles/policies. **Permission boundaries** cap the maximum permissions an identity can ever have, even if granted more. **SCPs (Service Control Policies)** do the same thing at the AWS Organizations level — a guardrail nothing in the account can escape. **ABAC (Attribute-Based Access Control)** uses tags to drive decisions (e.g. a user tagged `Team=alpha` can only touch resources tagged the same). **Condition keys** (`aws:SourceIp`, `aws:MultiFactorAuthPresent`) let you narrow policies to specific contexts. **Access Analyzer** automatically flags resources shared outside your account/org.

**Why it matters:** basic IAM lets you grant; advanced IAM lets you *contain*. Every serious AWS security story uses boundaries + SCPs + ABAC to make mistakes impossible instead of relying on everyone writing perfect policies.

**When to use:** multi-team accounts where devs need freedom but can't accidentally spin up EC2 in Tokyo; enforcing MFA/IP guardrails; scaling permissions without policy explosion.
**When NOT to use:** tiny personal accounts — the overhead isn't worth it. Stick with plain roles.

## Estimated cost
- IAM (boundaries, roles, policies): **$0.00** — free
- Organizations + SCPs: **$0.00** — free
- Access Analyzer: **$0.00** for external access analyzer; unused-access analyzer **$0.20 per IAM role/user/month**
- Total for this lesson: **~$0.00–$2/month** (depends on Access Analyzer type)

---

## Step 1: Create a permission boundary
> **Why:** Boundaries are the single most useful IAM feature you've never used. A user with `AdministratorAccess` + a boundary of `s3:*,logs:*` can only do S3 and CloudWatch Logs. This is how you delegate "create roles for Lambdas" without letting someone escalate to admin.

`lib/iam-advanced-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export class IamAdvancedStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 1) Permission boundary: max perms = S3 + Logs only
    const boundary = new iam.ManagedPolicy(this, 'DevBoundary', {
      managedPolicyName: 'DevBoundary',
      statements: [
        new iam.PolicyStatement({
          actions: ['s3:*', 'logs:*'],
          resources: ['*'],
        }),
      ],
    });

    // 2) User with Admin identity policy but bounded
    const user = new iam.User(this, 'BoundedAdmin', {
      userName: 'bounded-admin',
      managedPolicies: [iam.ManagedPolicy.fromAwsManagedPolicyName('AdministratorAccess')],
      permissionsBoundary: boundary,
    });

    new cdk.CfnOutput(this, 'UserArn', { value: user.userArn });
  }
}
```

Verify the boundary wins:
```bash
aws iam create-access-key --user-name bounded-admin
# configure profile with those keys, then:
aws s3 ls --profile bounded             # works
aws dynamodb list-tables --profile bounded  # AccessDenied — boundary blocks it
```

Expected output on DynamoDB call:
```
An error occurred (AccessDeniedException) when calling the ListTables operation:
User: arn:aws:iam::123456789012:user/bounded-admin is not authorized to perform: dynamodb:ListTables
because no permissions boundary allows the action
```

## Step 2: ABAC with tag matching
> **Why:** Instead of writing per-team S3 policies, tag users and buckets and let one policy route access. Add 50 more teams — zero policy changes. This is how platform teams scale IAM without going mad.

```typescript
const abacPolicy = new iam.ManagedPolicy(this, 'TeamBucketAccess', {
  statements: [
    new iam.PolicyStatement({
      actions: ['s3:GetObject', 's3:PutObject', 's3:ListBucket'],
      resources: ['arn:aws:s3:::*'],
      conditions: {
        StringEquals: {
          'aws:ResourceTag/Team': '${aws:PrincipalTag/Team}',
        },
      },
    }),
  ],
});

const alpha = new iam.User(this, 'AlphaUser', {
  userName: 'alice-alpha',
  managedPolicies: [abacPolicy],
});
cdk.Tags.of(alpha).add('Team', 'alpha');
```

Create test buckets and tag them `Team=alpha` / `Team=beta`. Alice accesses only `alpha` buckets.

## Step 3: Condition keys — MFA + SourceIP
> **Why:** Defense-in-depth. Even if creds leak, attacker can't use them outside the office IP. Even if they're in the office, they still need MFA. Two locks on one door.

```typescript
const restrictedPolicy = new iam.ManagedPolicy(this, 'RestrictedS3', {
  statements: [
    new iam.PolicyStatement({
      effect: iam.Effect.DENY,
      actions: ['s3:*'],
      resources: ['*'],
      conditions: {
        NotIpAddress: { 'aws:SourceIp': ['203.0.113.0/24'] },
        BoolIfExists: { 'aws:MultiFactorAuthPresent': 'false' },
      },
    }),
  ],
});
```

Test from outside the CIDR — all S3 calls fail with explicit deny.

## Step 4: SCP in Organizations (region lock)
> **Why:** SCPs live at the org level. No role, user, or even the root user of a member account can override them. Perfect for "no EC2 outside us-east-1" — a developer can't accidentally create a $2k/month instance in Tokyo.

Requires AWS Organizations enabled. In the management account:
```bash
aws organizations enable-policy-type \
  --root-id r-abcd \
  --policy-type SERVICE_CONTROL_POLICY

cat > region-lock.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "ec2:*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": { "aws:RequestedRegion": "us-east-1" }
    }
  }]
}
EOF

aws organizations create-policy \
  --name RegionLock \
  --type SERVICE_CONTROL_POLICY \
  --content file://region-lock.json

aws organizations attach-policy \
  --policy-id p-xxxxxxxx \
  --target-id ou-xxxx-xxxxxxxx
```

From a member account: `aws ec2 run-instances --region ap-northeast-1` fails with `UnauthorizedOperation` — SCP blocks before IAM even evaluates.

## Step 5: Role chaining
> **Why:** Role chain = assumed role assuming another role. Common in cross-account deploys (user → DeployRole in account A → DeployRole in account B). Session max drops to 1 hour when chained — surprises people.

```typescript
const roleA = new iam.Role(this, 'RoleA', {
  assumedBy: new iam.AccountPrincipal(this.account),
  maxSessionDuration: cdk.Duration.hours(4),
});

const roleB = new iam.Role(this, 'RoleB', {
  assumedBy: new iam.ArnPrincipal(roleA.roleArn),
  maxSessionDuration: cdk.Duration.hours(4),
});
roleA.addToPolicy(new iam.PolicyStatement({
  actions: ['sts:AssumeRole'],
  resources: [roleB.roleArn],
}));
```

```bash
# Assume A, then from A assume B
aws sts assume-role --role-arn <RoleA> --role-session-name one --duration-seconds 14400
# ... export creds ...
aws sts assume-role --role-arn <RoleB> --role-session-name two --duration-seconds 14400
# Max 3600 returned — chaining caps at 1 hour regardless of MaxSessionDuration
```

## Step 6: Enable Access Analyzer
> **Why:** Finds buckets/roles/KMS keys accidentally shared outside your account or org. Free for external-access findings. Run it once; fix the surprises.

```typescript
import * as accessanalyzer from 'aws-cdk-lib/aws-accessanalyzer';

new accessanalyzer.CfnAnalyzer(this, 'Analyzer', {
  type: 'ACCOUNT',
  analyzerName: 'ext-access',
});
```

Create an S3 bucket with a cross-account policy. Within ~30 min a finding appears in Access Analyzer console listing the bucket + principal.

## Cleanup
```bash
cdk destroy
aws organizations detach-policy --policy-id p-xxxxxxxx --target-id ou-xxxx-xxxxxxxx
aws organizations delete-policy --policy-id p-xxxxxxxx
aws iam delete-access-key --user-name bounded-admin --access-key-id AKIA...
```

## Common Errors
- **Boundary user can't do X even though identity policy allows** → expected. Boundary caps the ceiling. Add action to boundary.
- **`PrincipalTag` condition always fails** → user has no `Team` tag, or tag key casing differs. IAM tags are case-sensitive.
- **SCP attached but actions still succeed** → SCP doesn't apply to the management account. Test from a member account.
- **Role chain session only 1 hour** → hard AWS limit for chained sessions, not a bug.
- **Access Analyzer shows no findings** → give it 10–30 min to scan; verify it's in the region holding your resources.
