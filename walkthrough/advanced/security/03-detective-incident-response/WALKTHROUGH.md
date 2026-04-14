# Walkthrough — 03 Detective & Incident Response

## About this service
**Amazon Detective** ingests VPC Flow Logs, CloudTrail, GuardDuty, and EKS audit logs into a time-series behavior graph so you can investigate *who did what, from where, how long* without writing Athena queries. Paired with an **Incident Response (IR) runbook** — typically SSM Automation documents + Lambda + EventBridge — you get both the "what happened" (Detective) and the "make it stop now" (runbook).

**Why it matters:** GuardDuty gives you findings, but a finding like "EC2 making DNS queries to known-bad domain" doesn't tell you scope. Detective answers "did this instance contact others? who's the IAM role it used? was that role used elsewhere?" in seconds. An IR runbook makes the containment step (isolate SG, snapshot disk, capture memory) deterministic and fast — manual containment at 2am is where mistakes happen.
**When to use:** Any org with > a few AWS accounts; any regulated workload; whenever GuardDuty is on.
**When NOT to use:** Solo dev accounts with no real data (Detective costs stack up fast). Don't enable Detective without tight scoping.

## Estimated cost
- GuardDuty: **~$4/GB** of VPC Flow + CloudTrail (first 500 GB); **$1/million** DNS events
- Amazon Detective: **$2.70/GB** ingested (first 1 TB) — can be $500+/month in a medium account if left unscoped
- SSM Automation: **free** (just pay for underlying EC2/Lambda)
- S3 object-lock bucket for forensics: **$0.023/GB** + replication
- Total for this lesson (lab with small traffic): **~$30/month** — disable Detective the moment you're done

---

## Step 1: Enable Detective and link GuardDuty
> **Why:** Detective needs GuardDuty running for ≥48h before it's useful — it back-fills from GuardDuty's data sources. You enable it once per region; in an Organization you do it from the Security/Audit account with delegated admin.

```bash
# In your designated security account, as org admin
aws organizations register-delegated-administrator \
  --account-id 111122223333 --service-principal detective.amazonaws.com

# From security account
aws detective create-graph --tags Env=prod
GRAPH=$(aws detective list-graphs --query 'GraphList[0].Arn' --output text)

# Add member accounts
aws detective create-members --graph-arn $GRAPH \
  --accounts '[{"AccountId":"444455556666","EmailAddress":"sec@corp.com"}]' \
  --message "Security investigation graph"
```

After ~24h, open **Console → Detective → Summary**. You'll see active resources, new behavior, and the "findings to investigate" queue sourced from GuardDuty.

## Step 2: Investigate a GuardDuty sample finding with Detective
> **Why:** The investigation graph is Detective's whole value prop. A finding says "IAM user `cicd-deploy` called `CreateAccessKey` from an unusual IP" — Detective immediately shows you: that IP's other activity, the role's API history over 30 days, what resources that role touched, and whether any of those touched resources show up in other findings.

Generate sample findings:
```bash
aws guardduty create-sample-findings \
  --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text) \
  --finding-types UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom
```

In the console:
1. **GuardDuty → Findings →** open the sample finding → **Investigate in Detective**.
2. Detective opens to the IAM user profile: graph shows spikes in API calls.
3. Click the unusual IP → see *every* principal that ever called from it in the last 90 days.
4. Pivot to "Resources affected" → list of S3 buckets, EC2 instances touched.

Write the scope in a doc: principals involved, resources touched, time window. This becomes the containment scope for Step 3.

## Step 3: SSM Automation runbook for instance isolation
> **Why:** "Isolate a compromised EC2" is a 4-step sequence: swap its security group to a deny-all, snapshot the EBS volume for forensics, capture memory (via SSM `AWS-CreateImage` or a memory-dump agent), and notify. Doing it manually at 3am, under pressure, you will forget step 2 or apply step 1 to the wrong instance. SSM runbooks make it one command.

`quarantine-instance.yaml`:
```yaml
schemaVersion: '0.3'
description: Isolate compromised EC2 — swap SG, snapshot, memdump, notify
assumeRole: '{{ AutomationAssumeRole }}'
parameters:
  InstanceId: { type: String }
  QuarantineSG: { type: String }
  TopicArn: { type: String }
  AutomationAssumeRole: { type: String }
mainSteps:
  - name: SwapSecurityGroup
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: ModifyInstanceAttribute
      InstanceId: '{{ InstanceId }}'
      Groups: ['{{ QuarantineSG }}']

  - name: DisableIMDSForBadActors
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: ModifyInstanceMetadataOptions
      InstanceId: '{{ InstanceId }}'
      HttpTokens: required
      HttpPutResponseHopLimit: 1

  - name: SnapshotVolumes
    action: aws:executeScript
    inputs:
      Runtime: python3.11
      Handler: handler
      Script: |
        import boto3
        def handler(events, ctx):
          ec2 = boto3.client('ec2')
          vols = [m['Ebs']['VolumeId'] for m in ec2.describe_instances(
            InstanceIds=[events['InstanceId']])['Reservations'][0]
            ['Instances'][0]['BlockDeviceMappings']]
          snaps = [ec2.create_snapshot(VolumeId=v,
            Description=f'forensics-{events["InstanceId"]}',
            TagSpecifications=[{'ResourceType':'snapshot','Tags':[
              {'Key':'Forensics','Value':'true'},
              {'Key':'Source','Value':events['InstanceId']}]}])['SnapshotId']
            for v in vols]
          return {'Snapshots': snaps}
      InputPayload: { InstanceId: '{{ InstanceId }}' }

  - name: CaptureMemory
    action: aws:runCommand
    inputs:
      DocumentName: AWS-RunShellScript
      InstanceIds: ['{{ InstanceId }}']
      Parameters:
        commands:
          - 'sudo apt-get install -y lime-forensics-dkms || true'
          - 'sudo insmod /lib/modules/$(uname -r)/updates/dkms/lime.ko "path=/tmp/mem.lime format=lime"'
          - 'aws s3 cp /tmp/mem.lime s3://forensics-evidence-bucket/{{ InstanceId }}/mem.lime'

  - name: Notify
    action: aws:executeAwsApi
    inputs:
      Service: sns
      Api: Publish
      TopicArn: '{{ TopicArn }}'
      Subject: 'IR: {{ InstanceId }} quarantined'
      Message: 'Instance isolated, snapshots pending, memdump to S3.'
```

Register and run:
```bash
aws ssm create-document --name IR-Quarantine --document-type Automation \
  --document-format YAML --content file://quarantine-instance.yaml

aws ssm start-automation-execution --document-name IR-Quarantine \
  --parameters "InstanceId=i-0abc,QuarantineSG=sg-deny-all,TopicArn=arn:aws:sns:...,AutomationAssumeRole=arn:aws:iam::...:role/IRAutomation"
```

Target: isolation in under 30 seconds.

## Step 4: Tag-based auto-quarantine via EventBridge
> **Why:** SOC wants to click one button or add one tag and have quarantine happen. EventBridge rule on "tag added = `Quarantine:true`" fires a Lambda that kicks the SSM runbook. This moves analysts out of the CLI.

```typescript
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as lambda from 'aws-cdk-lib/aws-lambda';

const quarantineFn = new lambda.Function(this, 'Quarantine', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    const { SSM } = require('@aws-sdk/client-ssm');
    const ssm = new SSM({});
    exports.handler = async (e) => {
      const instanceId = e.detail['resource-id'];
      await ssm.startAutomationExecution({
        DocumentName: 'IR-Quarantine',
        Parameters: {
          InstanceId: [instanceId],
          QuarantineSG: [process.env.QUARANTINE_SG],
          TopicArn: [process.env.TOPIC_ARN],
          AutomationAssumeRole: [process.env.ROLE_ARN],
        },
      });
    };`),
  environment: { QUARANTINE_SG: 'sg-xxx', TOPIC_ARN: topic.topicArn, ROLE_ARN: role.roleArn },
});

new events.Rule(this, 'OnQuarantineTag', {
  eventPattern: {
    source: ['aws.tag'],
    detailType: ['Tag Change on Resource'],
    detail: {
      service: ['ec2'],
      'resource-type': ['instance'],
      'changed-tag-keys': ['Quarantine'],
      tags: { Quarantine: ['true'] },
    },
  },
  targets: [new targets.LambdaFunction(quarantineFn)],
});
```

Trigger:
```bash
aws ec2 create-tags --resources i-0abc --tags Key=Quarantine,Value=true
# → EventBridge → Lambda → SSM runbook runs
```

## Step 5: Forensics evidence bucket (immutable + cross-account)
> **Why:** Evidence has to survive the attacker deleting it — even if the attacker compromised the account the instance lived in. Standard pattern: evidence lives in a **separate security account**, in an S3 bucket with **Object Lock in Compliance mode** (not even root can delete), KMS-encrypted, with access logging to a third audit account.

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

const evidence = new s3.Bucket(this, 'Forensics', {
  bucketName: `forensics-${this.account}-${this.region}`,
  encryption: s3.BucketEncryption.KMS,
  encryptionKey: forensicsKey,
  objectLockEnabled: true,
  objectLockDefaultRetention: s3.ObjectLockRetention.compliance(cdk.Duration.days(2555)), // 7y
  versioned: true,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  enforceSSL: true,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});

// Allow the IR Lambda (in the prod account) to PUT, nothing else
evidence.addToResourcePolicy(new iam.PolicyStatement({
  principals: [new iam.ArnPrincipal('arn:aws:iam::PROD-ACCT:role/IRAutomation')],
  actions: ['s3:PutObject'],
  resources: [evidence.arnForObjects('*')],
  conditions: { StringEquals: { 's3:x-amz-object-lock-mode': 'COMPLIANCE' } },
}));
```

**Compliance vs Governance mode:** Compliance = nobody, not even root, can shorten/remove the lock. Governance = a privileged IAM principal with `s3:BypassGovernanceRetention` can. For real forensics, Compliance — but you cannot ever reverse it, so pilot with Governance first.

## Step 6: Tabletop — compromised IAM key scenario
> **Why:** The runbook is only useful if the team knows when to run it. Tabletop = pen-and-paper walkthrough of a scenario, identifying gaps before a real incident. Write it down so on-call has a checklist.

`incident-playbook-ir-compromised-key.md` (drafted in your repo):
```
Scenario: GuardDuty finding "AccessKey used from Tor exit node"

T+0  On-call paged.
T+1  Confirm finding real (not sample). Note Principal + AccessKeyId.
T+2  In IAM: `update-access-key --status Inactive --access-key-id AKIA...`
T+3  In CloudTrail Lake / Detective: scope — what APIs did this key call in the last 24h?
T+5  For each resource touched: check integrity (Config snapshot, S3 versioning).
T+10 Rotate / revoke any downstream credentials the attacker may have exfiltrated.
T+15 Delete the access key, force MFA reset on the user.
T+30 Notify: #security-incidents, security-leadership@, legal if PII.
T+60 Preserve evidence: CloudTrail events exported to forensics bucket; Detective URL archived.
T+1d Postmortem: how was the key leaked? commit scan of repos? laptop compromise?
```

Run the tabletop quarterly. Time the steps — if T+2 takes 15 minutes because nobody knows which role has `iam:UpdateAccessKey`, that's a finding.

## Cleanup
```bash
# Stop Detective bills ASAP — they add up
aws detective delete-graph --graph-arn $GRAPH

# Keep the forensics bucket (Object Lock = can't delete contents for retention period anyway)
# Delete the runbook/Lambda if truly done:
cdk destroy
```

## Common Errors
- **Detective shows no data** → GuardDuty must be enabled in the same region for 48h first. Also check org delegation.
- **SSM automation fails `AccessDenied`** → the `AutomationAssumeRole` needs `ec2:ModifyInstanceAttribute`, `ec2:CreateSnapshot`, `ssm:SendCommand`, `sns:Publish` — most people miss one.
- **Object Lock Compliance cannot be disabled** → if you set it in Compliance for 7 years by accident on a test bucket, that bucket is effectively permanent. Always pilot with Governance.
- **Memory dump step fails on Windows instances** → LiME is Linux-only. Use `AWS-RunPowerShellScript` with `WinPMem` for Windows.
- **EventBridge rule doesn't fire** → Tag Change events only fire when using `tag-on-create` or a subsequent `create-tags` call that *changes* the value; setting same value is a no-op.
- **Quarantine SG still lets out traffic** → "deny all" SG still allows egress by default if you leave the default rule. Remove it explicitly.
