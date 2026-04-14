# Walkthrough — 01 Multi-Account Landing Zone

## About this service
A **landing zone** is the multi-account baseline you build on AWS before any workload lands. It uses **AWS Organizations** (the container that groups accounts), **Control Tower** (opinionated blueprint that sets up log-archive/audit accounts plus guardrails), and **Account Factory for Terraform (AFT)** (GitOps account vending). You group accounts into **Organizational Units (OUs)** and attach **Service Control Policies (SCPs)** — the org-wide IAM ceiling that even the root user cannot escape.

**Why it matters:** accounts are the only *hard* isolation boundary in AWS. IAM boundaries within one account are porous; a separate account gives you independent quotas, independent blast radius, clean billing, and a place where a compromise doesn't leak into prod. Every serious AWS footprint is multi-account.

**When to use:** any time you have more than one team, more than one environment (dev/stage/prod), or compliance requirements (PCI, HIPAA, SOC2). Start multi-account on day 1 — migrating later is brutal.
**When NOT to use:** pure throwaway sandbox, solo learning account. A single account is fine until you need the isolation.

## Estimated cost
- AWS Organizations: **free**
- Control Tower: **free** (you pay for underlying CloudTrail/Config/SNS — typically $5-15/month per enrolled account)
- AWS Config (mandatory): **$0.003 per configuration item** + $0.001/rule evaluation — **~$5/month per account** at low activity
- CloudTrail management events: **free** for the first copy; org-trail to log archive bucket ~$2/month S3 storage at low volume
- S3 log archive bucket: **~$0.023/GB** (us-east-1 Standard) — **~$3/month** for learning volumes
- IAM Identity Center (SSO): **free**
- Total for this lesson (3 accounts + mgmt): **~$25-40/month** while running. Close accounts to stop charges.

---

## Step 1: Enable AWS Organizations in the management account
> **Why:** Organizations is the root container. You must create it from the account that will be the management (payer) account — that role cannot be transferred later without a migration. Enable **all features** (not just consolidated billing) so you can use SCPs.

```bash
# From the account you want as the management/payer
aws organizations create-organization --feature-set ALL

# Confirm
aws organizations describe-organization
```

Expected output:
```json
{
  "Organization": {
    "Id": "o-abcd1234ef",
    "Arn": "arn:aws:organizations::111111111111:organization/o-abcd1234ef",
    "FeatureSet": "ALL",
    "MasterAccountId": "111111111111",
    "MasterAccountEmail": "you+aws-mgmt@example.com"
  }
}
```

## Step 2: Design the OU structure
> **Why:** OUs are how you scope SCPs and delegated admins. Get the hierarchy wrong and you'll be moving dozens of accounts later. The canonical layout below is what Control Tower expects and what AWS Well-Architected recommends.

Target structure:
```
Root
├── Security/               (← Control Tower creates this)
│   ├── Log Archive         (immutable CloudTrail/Config sink)
│   └── Audit               (Security Hub, GuardDuty delegated admin)
├── Sandbox/                (experiments, destroy freely, deny prod data)
├── Workloads/
│   ├── Dev                 (shared dev accounts per team)
│   └── Prod                (prod accounts, strictest SCPs)
├── Infrastructure/         (shared services: DNS, transit, CI/CD)
└── Suspended/              (deprovisioned accounts pending closure)
```

Create the OUs:
```bash
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)

for OU in Sandbox Workloads Infrastructure Suspended; do
  aws organizations create-organizational-unit --parent-id "$ROOT_ID" --name "$OU"
done

WORKLOADS_ID=$(aws organizations list-organizational-units-for-parent \
  --parent-id "$ROOT_ID" --query "OrganizationalUnits[?Name=='Workloads'].Id" --output text)

aws organizations create-organizational-unit --parent-id "$WORKLOADS_ID" --name Dev
aws organizations create-organizational-unit --parent-id "$WORKLOADS_ID" --name Prod
```

## Step 3: Enable Control Tower
> **Why:** Control Tower wires up the audit + log-archive accounts, CloudTrail org trail, Config aggregator, and ~20 mandatory guardrails automatically. Doing this by hand takes days. **This step is effectively one-way** — tearing down Control Tower requires a documented decommission runbook, so do it in a throwaway org if you're just learning.

Console path (no CLI for initial landing zone setup):

1. Open **AWS Control Tower** in us-east-1 (home region).
2. **Set up landing zone** → choose home region + additional governed regions (pick 1 extra for the exercise, e.g. `us-west-2`).
3. Email for **log archive** account: `you+aws-logs@example.com`
4. Email for **audit** account: `you+aws-audit@example.com`
5. Review mandatory guardrails (all "Strongly recommended" preventive + detective).
6. Launch. Setup takes **60-90 minutes**.

What gets created:
- `Security` OU with `Log Archive` and `Audit` accounts
- Org CloudTrail writing to log archive S3 bucket
- Config aggregator in audit account
- IAM Identity Center enabled with `AWSControlTowerAdmins` permission set
- ~20 SCPs attached to OUs

## Step 4: Vend 3 workload accounts via Account Factory
> **Why:** Account Factory gives you a repeatable, audited account-creation pipeline. Clicking "Create account" in the Organizations console bypasses Control Tower's baseline and you'll end up with ungoverned accounts.

Console: **Control Tower → Account Factory → Create account** ×3:

| Account name | Email                         | OU              |
|--------------|-------------------------------|-----------------|
| learn-dev    | you+aws-dev@example.com       | Workloads/Dev   |
| learn-stage  | you+aws-stage@example.com     | Workloads/Dev   |
| learn-prod   | you+aws-prod@example.com      | Workloads/Prod  |

Each account takes ~15 minutes to provision. When done, list them:
```bash
aws organizations list-accounts --query 'Accounts[].[Name,Id,Email,Status]' --output table
```

## Step 5: Custom SCPs (CDK)
> **Why:** The default guardrails are generic. You always need company-specific SCPs: deny root, deny region sprawl, deny disabling CloudTrail, deny leaving the org. Managing these as code (CDK) beats clicking through the console.

`lib/org-scps-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as organizations from 'aws-cdk-lib/aws-organizations';

export interface ScpStackProps extends cdk.StackProps {
  readonly workloadsOuId: string;
  readonly prodOuId: string;
  readonly allowedRegions: string[];
}

export class OrgScpsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: ScpStackProps) {
    super(scope, id, props);

    // 1. Deny root user actions anywhere
    const denyRoot = new organizations.CfnPolicy(this, 'DenyRootUser', {
      name: 'DenyRootUserActions',
      type: 'SERVICE_CONTROL_POLICY',
      targetIds: [props.workloadsOuId],
      content: {
        Version: '2012-10-17',
        Statement: [{
          Sid: 'DenyRoot',
          Effect: 'Deny',
          Action: '*',
          Resource: '*',
          Condition: { StringLike: { 'aws:PrincipalArn': 'arn:aws:iam::*:root' } },
        }],
      },
    });

    // 2. Region allow-list (fail-closed outside approved regions)
    new organizations.CfnPolicy(this, 'RegionAllowList', {
      name: 'RegionAllowList',
      type: 'SERVICE_CONTROL_POLICY',
      targetIds: [props.workloadsOuId],
      content: {
        Version: '2012-10-17',
        Statement: [{
          Sid: 'DenyNonApprovedRegions',
          Effect: 'Deny',
          NotAction: [
            'iam:*', 'organizations:*', 'route53:*',
            'cloudfront:*', 'support:*', 'sts:*', 'budgets:*',
          ],
          Resource: '*',
          Condition: {
            StringNotEquals: { 'aws:RequestedRegion': props.allowedRegions },
          },
        }],
      },
    });

    // 3. Prevent leaving the org / disabling CloudTrail / Config
    new organizations.CfnPolicy(this, 'ProtectGuardrails', {
      name: 'ProtectGuardrails',
      type: 'SERVICE_CONTROL_POLICY',
      targetIds: [props.workloadsOuId],
      content: {
        Version: '2012-10-17',
        Statement: [{
          Effect: 'Deny',
          Action: [
            'organizations:LeaveOrganization',
            'cloudtrail:StopLogging',
            'cloudtrail:DeleteTrail',
            'config:DeleteConfigurationRecorder',
            'config:StopConfigurationRecorder',
            'guardduty:DeleteDetector',
            'guardduty:DisassociateFromMasterAccount',
          ],
          Resource: '*',
        }],
      },
    });

    // 4. Prod-only: require MFA for destructive actions
    new organizations.CfnPolicy(this, 'ProdMfaOnDestructive', {
      name: 'ProdRequireMfaDestructive',
      type: 'SERVICE_CONTROL_POLICY',
      targetIds: [props.prodOuId],
      content: {
        Version: '2012-10-17',
        Statement: [{
          Effect: 'Deny',
          Action: [
            'ec2:TerminateInstances', 'rds:DeleteDBInstance',
            's3:DeleteBucket', 'dynamodb:DeleteTable',
          ],
          Resource: '*',
          Condition: { BoolIfExists: { 'aws:MultiFactorAuthPresent': 'false' } },
        }],
      },
    });
  }
}
```

`bin/app.ts`:
```typescript
new OrgScpsStack(app, 'OrgScpsStack', {
  env: { account: '111111111111', region: 'us-east-1' }, // management account
  workloadsOuId: 'ou-abcd-workloads',
  prodOuId: 'ou-abcd-prod',
  allowedRegions: ['us-east-1', 'us-west-2'],
});
```

Deploy from the management account:
```bash
cdk deploy OrgScpsStack
```

## Step 6: Delegate Security Hub + GuardDuty to the audit account
> **Why:** You don't want to run security tooling from the management account (blast radius) or copy findings around manually. **Delegated administration** lets the audit account act as the org-wide security plane while the mgmt account stays minimal.

```bash
AUDIT_ACCOUNT_ID=222222222222

# Delegate GuardDuty
aws organizations register-delegated-administrator \
  --service-principal guardduty.amazonaws.com \
  --account-id "$AUDIT_ACCOUNT_ID"

# Delegate Security Hub
aws organizations register-delegated-administrator \
  --service-principal securityhub.amazonaws.com \
  --account-id "$AUDIT_ACCOUNT_ID"

# Delegate Config aggregator
aws organizations register-delegated-administrator \
  --service-principal config.amazonaws.com \
  --account-id "$AUDIT_ACCOUNT_ID"
```

Then, **from the audit account**, enable org-wide GuardDuty auto-enroll:
```bash
aws guardduty create-detector --enable
DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)
aws guardduty update-organization-configuration \
  --detector-id "$DETECTOR_ID" --auto-enable-organization-members ALL
```

## Step 7: CDK cross-account bootstrap + pipeline
> **Why:** Each account needs `cdk bootstrap` before it can receive deploys, and you want a central tooling account to push to all of them. The `--trust` flag says "this tooling account may assume the CDK deploy roles here".

```bash
TOOLING=333333333333   # the account running your pipeline
PIPELINE_EXEC=arn:aws:iam::$TOOLING:role/cdk-pipeline-role

for ACCT in 444444444444 555555555555 666666666666; do  # dev/stage/prod
  cdk bootstrap aws://$ACCT/us-east-1 \
    --trust $TOOLING \
    --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess \
    --profile account-$ACCT
done
```

Minimal CDK pipeline stack (`lib/landing-zone-pipeline.ts`):
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { CodePipeline, CodePipelineSource, ShellStep } from 'aws-cdk-lib/pipelines';

export class LzPipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    const pipeline = new CodePipeline(this, 'Pipeline', {
      pipelineName: 'landing-zone',
      synth: new ShellStep('Synth', {
        input: CodePipelineSource.connection('me/landing-zone', 'main', {
          connectionArn: 'arn:aws:codeconnections:us-east-1:333333333333:connection/abc',
        }),
        commands: ['npm ci', 'npm run build', 'npx cdk synth'],
      }),
    });

    // Add stages per account with manual approval before prod
    pipeline.addStage(new AppStage(this, 'Dev',   { env: { account: '444444444444', region: 'us-east-1' } }));
    pipeline.addStage(new AppStage(this, 'Stage', { env: { account: '555555555555', region: 'us-east-1' } }));
    pipeline.addStage(new AppStage(this, 'Prod',  { env: { account: '666666666666', region: 'us-east-1' } }),
      { pre: [new ManualApprovalStep('PromoteToProd')] });
  }
}
```

## Step 8: Verify
> **Why:** Paper guardrails don't stop attackers — you need to prove the SCP actually denies in the child account before you trust it.

From the dev account, try a forbidden region:
```bash
aws ec2 describe-instances --region eu-west-1
# Expected: An error occurred (UnauthorizedOperation) ... explicit deny in a service control policy
```

From log archive, confirm logs are flowing:
```bash
aws s3 ls s3://aws-controltower-logs-222222222222-us-east-1/AWSLogs/ | head
# Expected: one prefix per member account ID
```

## Cleanup
> **Why:** Control Tower cannot be fully unwound from the CLI — this is the most expensive lesson to leave running because of accumulated Config + CloudTrail charges across accounts.

1. Console: **Control Tower → Landing zone settings → Decommission**. Takes ~2 hours.
2. Close each vended account: **My Account → Close Account** (90-day suspension before real deletion).
3. `cdk destroy OrgScpsStack` in mgmt account.
4. Leave/disable the organization if you no longer need it:
   ```bash
   aws organizations delete-organization
   ```
   (Only works when all member accounts are removed.)

## Common Errors
- **`AccessDenied` when deploying to child account** → you forgot `cdk bootstrap --trust <tooling>` in the target account, or the pipeline role lacks `sts:AssumeRole` for the cdk deploy role.
- **SCP "deny all regions" locks you out of your own session** → always include `iam:*`, `organizations:*`, `sts:*`, `route53:*`, `cloudfront:*` in `NotAction` (global services). Recover via the management account.
- **"Cannot close account — pending transactions"** → wait for the current billing cycle, then retry. Accounts enter 90-day SUSPENDED state before deletion.
- **Control Tower setup fails at "Creating log archive"** → email already in use, or mgmt account has pre-existing CloudTrail conflict. Delete the stray trail and retry.
- **GuardDuty delegation fails with `OrganizationNotInAllFeaturesMode`** → your org is billing-only. Run `aws organizations enable-all-features` (requires invite acceptance from every member).
