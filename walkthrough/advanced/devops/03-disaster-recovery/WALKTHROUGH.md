# Walkthrough — 03 Disaster Recovery

## About this service
**DR on AWS** spans four strategies that trade cost for recovery time: **Backup/Restore** (hours-to-days RTO, cheapest), **Pilot Light** (tens of minutes RTO, core data replicated), **Warm Standby** (minutes RTO, scaled-down live stack), **Active/Active** (seconds RTO, full duplicate). Tools: **AWS Backup** centralizes plans across RDS/EBS/DDB/EFS/EC2/S3; **Aurora Global Database** gives <1s RPO cross-region; **Route53 ARC + Application Recovery Controller** handles failover routing; **AWS Fault Injection Service (FIS)** is managed chaos engineering.

**Why it matters:** RPO (max acceptable data loss) and RTO (max acceptable downtime) are business decisions, not technical ones. Running a DR drill quarterly — and measuring actual RTO vs target — is the only way to know if your runbook works. FIS catches the failure modes you didn't think of.
**When to use:** any production workload with an SLA; regulated workloads (HIPAA, PCI) require tested DR.
**When NOT to use:** dev/staging environments (snapshot-only is fine); ephemeral compute with no state.

## Estimated cost
- **AWS Backup storage (warm)**: **$0.05/GB-month** for EBS/RDS/DDB/EFS; **$0.0125/GB-month** cold tier.
- **Cross-region copy**: storage doubles + **$0.02/GB** inter-region transfer (one-time per copy).
- **Aurora Global Database**: secondary cluster minimum 1× db.r6g.large = **~$210/month** + replicated-write I/O **$0.20/million** + cross-region data transfer $0.02/GB.
- **Pilot Light stack** (scaled-to-minimum in DR region): 1× NAT ($33/mo) + 1× t3.micro ASG ($7.50/mo) + ALB ($16.20/mo) ≈ **$57/mo**.
- **Warm Standby**: full topology at 25% capacity ≈ 25% of prod bill.
- **FIS**: **$0.10 per action-minute** (e.g. stopping 20 instances for 5 min = $10).
- **Route 53 health checks**: $0.50/check-mo + $2.00/check-mo for HTTPS + string match.
- Total for this lesson (Backup + one FIS drill + a scaled-down Pilot Light): **~$75/month**.

---

## Step 1: AWS Backup plan — daily, 30-day retention, cross-region
> **Why:** Centralizing plans in AWS Backup beats per-service snapshot jobs — one policy covers RDS, EBS, DDB, EFS, FSx, and S3. Cross-region copy is the only thing that survives a regional outage.

```typescript
import { Stack, StackProps, Duration, RemovalPolicy } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as backup from 'aws-cdk-lib/aws-backup';
import * as events from 'aws-cdk-lib/aws-events';

export class BackupStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vault = new backup.BackupVault(this, 'PrimaryVault', {
      backupVaultName: 'prod-primary',
      removalPolicy: RemovalPolicy.RETAIN,
    });

    const plan = new backup.BackupPlan(this, 'Daily', {
      backupPlanName: 'daily-30d-xregion',
      backupVault: vault,
    });

    plan.addRule(new backup.BackupPlanRule({
      ruleName: 'Daily',
      scheduleExpression: events.Schedule.cron({ hour: '5', minute: '0' }), // 05:00 UTC
      startWindow: Duration.hours(1),
      completionWindow: Duration.hours(4),
      deleteAfter: Duration.days(30),
      copyActions: [{
        destinationBackupVault: backup.BackupVault.fromBackupVaultArn(
          this, 'DrVault',
          'arn:aws:backup:us-west-2:123456789012:backup-vault:prod-dr',
        ),
        deleteAfter: Duration.days(30),
      }],
    }));

    plan.addSelection('Tagged', {
      resources: [backup.BackupResource.fromTag('Backup', 'daily')],
    });
  }
}
```

Tag the resources you want covered:
```bash
aws rds add-tags-to-resource --resource-name arn:aws:rds:...:db:prod-orders \
  --tags Key=Backup,Value=daily
aws dynamodb tag-resource --resource-arn arn:aws:dynamodb:...:table/prod-orders \
  --tags Key=Backup,Value=daily
```

## Step 2: Restore test — measure RTO
> **Why:** An untested backup is hope, not DR. Restoring to an alt region validates both the copy and the IAM/KMS plumbing.

```bash
# list recovery points in DR region
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name prod-dr --region us-west-2

# start a restore
START=$(date +%s)
aws backup start-restore-job --region us-west-2 \
  --recovery-point-arn arn:aws:backup:us-west-2:...:recovery-point:abc \
  --iam-role-arn arn:aws:iam::123456789012:role/service-role/AWSBackupDefaultServiceRole \
  --metadata '{"DBInstanceIdentifier":"prod-orders-restored","Engine":"postgres","DBSubnetGroupName":"dr-subnets"}'

# poll
aws backup describe-restore-job --restore-job-id ...
# STATUS: COMPLETED
END=$(date +%s); echo "RTO: $((END - START))s"
# RTO: 1140s  (19 min for a 50 GB Postgres)
```

## Step 3: Pilot Light with Aurora Global Database
> **Why:** Aurora Global Database replicates at the storage layer (<1s RPO, typical) and promotes in ~1 minute. Secondary runs read-only — much cheaper than warm standby, far faster than restore-from-snapshot.

```typescript
import { aws_rds as rds, aws_ec2 as ec2 } from 'aws-cdk-lib';

const primary = new rds.DatabaseCluster(this, 'Primary', {
  engine: rds.DatabaseClusterEngine.auroraPostgres({ version: rds.AuroraPostgresEngineVersion.VER_16_2 }),
  writer: rds.ClusterInstance.provisioned('Writer', { instanceType: ec2.InstanceType.of(ec2.InstanceClass.R6G, ec2.InstanceSize.LARGE) }),
  readers: [rds.ClusterInstance.provisioned('Reader', { instanceType: ec2.InstanceType.of(ec2.InstanceClass.R6G, ec2.InstanceSize.LARGE) })],
  vpc: primaryVpc,
  storageEncrypted: true,
});

new rds.CfnGlobalCluster(this, 'Global', {
  globalClusterIdentifier: 'prod-orders-global',
  sourceDbClusterIdentifier: primary.clusterArn,
});
```

In the DR stack (us-west-2):
```typescript
new rds.CfnDBCluster(this, 'DrCluster', {
  engine: 'aurora-postgresql',
  globalClusterIdentifier: 'prod-orders-global',
  dbClusterIdentifier: 'prod-orders-dr',
  dbSubnetGroupName: drSubnetGroup.ref,
  vpcSecurityGroupIds: [drSg.securityGroupId],
});
```

Failover drill:
```bash
aws rds failover-global-cluster \
  --global-cluster-identifier prod-orders-global \
  --target-db-cluster-identifier arn:aws:rds:us-west-2:...:cluster:prod-orders-dr
# measure: time from start to first successful write in DR
```

## Step 4: Warm Standby with Route 53 failover
> **Why:** Warm Standby runs the full topology at reduced scale. Route 53 health-check-based failover flips DNS in 60s (TTL + check interval).

```typescript
import { aws_route53 as r53, aws_route53_targets as targets } from 'aws-cdk-lib';

const zone = r53.HostedZone.fromLookup(this, 'Zone', { domainName: 'example.com' });

const primaryHealth = new r53.CfnHealthCheck(this, 'PrimaryHc', {
  healthCheckConfig: {
    type: 'HTTPS',
    fullyQualifiedDomainName: 'primary.example.com',
    port: 443, resourcePath: '/healthz',
    requestInterval: 10, failureThreshold: 3,
  },
});

new r53.ARecord(this, 'PrimaryAlias', {
  zone, recordName: 'app',
  target: r53.RecordTarget.fromAlias(new targets.LoadBalancerTarget(primaryAlb)),
  setIdentifier: 'primary',
  // cast until the L1 prop is exposed:
  // failover: r53.Failover.PRIMARY  (use CfnRecordSet for full control)
});
```

Use `CfnRecordSet` when you need `Failover: PRIMARY/SECONDARY` + `HealthCheckId` wired together.

## Step 5: FIS experiments
> **Why:** Running experiments in prod (with blast-radius guards) surfaces failure modes no staging env can simulate — throttles, API errors, AZ loss. Blast radius = tags + stop conditions on CloudWatch alarms.

`fis/stop-half-asg.json`:
```json
{
  "description": "Stop 50% of prod-api ASG instances",
  "roleArn": "arn:aws:iam::123456789012:role/FISRole",
  "stopConditions": [
    { "source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:AlbHighErrorRate" }
  ],
  "targets": {
    "Instances": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": { "aws:autoscaling:groupName": "prod-api", "chaos": "eligible" },
      "selectionMode": "PERCENT(50)"
    }
  },
  "actions": {
    "StopInstances": {
      "actionId": "aws:ec2:stop-instances",
      "parameters": { "startInstancesAfterDuration": "PT5M" },
      "targets": { "Instances": "Instances" }
    }
  },
  "tags": { "project": "chaos-day" }
}
```

```bash
aws fis create-experiment-template --cli-input-json file://fis/stop-half-asg.json
aws fis start-experiment --experiment-template-id EXT123abc
# watch CloudWatch: ALB HealthyHostCount halves, then recovers as ASG replaces
```

Additional experiments to run:
```bash
# Inject 500ms network latency to RDS endpoint
aws fis create-experiment-template --cli-input-json file://fis/rds-latency.json
# Throttle DynamoDB reads on prod-orders
aws fis create-experiment-template --cli-input-json file://fis/ddb-throttle.json
```

## Step 6: Chaos day — AZ failure simulation
> **Why:** "What happens if one AZ disappears?" is the most common post-mortem question. Blocking the NAT in one AZ's route table mimics the subnet losing egress — your multi-AZ ASG should keep serving.

```bash
# save current default route for rollback
aws ec2 describe-route-tables --route-table-ids rtb-az1-private > rtb-backup.json

# blackhole 0.0.0.0/0 in AZ-1 private route table
aws ec2 replace-route --route-table-id rtb-az1-private \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0000000000000000  # non-existent = blackhole
# ... monitor ALB, CloudWatch, synthetic canary ...
# rollback
aws ec2 replace-route --route-table-id rtb-az1-private \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-abc123
```

## Step 7: DR runbook
> **Why:** At 3am during a real outage, no one remembers the AWS CLI flags. A runbook that's been tested quarterly is the difference between a 30-min outage and a 6-hour outage.

`dr-runbook.md` (outline):
```markdown
# Prod DR Failover Runbook
Target RTO: 30 min   Target RPO: 1 min (Aurora Global) / 24 h (S3/DDB)

## 0. Declare
- Incident commander names themselves. Open #incident-prod bridge.

## 1. Confirm primary is lost (not just flaky)
- [ ] Route 53 health check for app.example.com has been UNHEALTHY for >5 min
- [ ] AWS Health Dashboard shows regional event, OR
- [ ] 3 of 3 synthetic canaries in us-east-1 failing

## 2. Promote Aurora Global secondary
aws rds failover-global-cluster \
  --global-cluster-identifier prod-orders-global \
  --target-db-cluster-identifier arn:aws:rds:us-west-2:...:cluster:prod-orders-dr

## 3. Scale DR ASGs from 2 -> 20
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name prod-api-dr --desired-capacity 20 --region us-west-2

## 4. Flip Route 53
aws route53 change-resource-record-sets --hosted-zone-id Z... --change-batch file://failover.json

## 5. Validate
- [ ] curl https://app.example.com/healthz -> 200
- [ ] Synthetic canary in us-west-2 passing
- [ ] Error rate in CloudWatch < 1% for 5 min

## 6. Post-failover
- [ ] Update status page
- [ ] Open ticket to rebuild primary region
- [ ] Schedule failback window
```

## Cleanup
```bash
# Delete FIS templates
aws fis delete-experiment-template --id EXT123abc

# Aurora Global — detach secondary then delete
aws rds remove-from-global-cluster --global-cluster-identifier prod-orders-global \
  --db-cluster-identifier arn:aws:rds:us-west-2:...:cluster:prod-orders-dr
aws rds delete-db-cluster --db-cluster-identifier prod-orders-dr --skip-final-snapshot --region us-west-2
aws rds delete-global-cluster --global-cluster-identifier prod-orders-global

# CDK
cdk destroy BackupStack

# Backup vaults cannot be deleted while they hold recovery points.
# Option A: wait for retention to age out (30 days).
# Option B:
aws backup list-recovery-points-by-backup-vault --backup-vault-name prod-primary \
  --query 'RecoveryPoints[].RecoveryPointArn' --output text | \
  xargs -n1 aws backup delete-recovery-point --backup-vault-name prod-primary --recovery-point-arn
aws backup delete-backup-vault --backup-vault-name prod-primary
```

## Common Errors
- **`BackupVault` can't be deleted** → recovery points still present; delete them first or wait for retention.
- **Restore fails with `KMSKeyNotAccessibleFault`** → DR-region KMS key missing grant for `backup.amazonaws.com` service principal. Add a grant or use a multi-Region key.
- **Aurora Global `failover-global-cluster` returns `InvalidGlobalClusterStateFault`** → primary writer is still healthy; use `failover` for planned, or `remove-from-global-cluster` + promote for unplanned.
- **FIS experiment does nothing** → `selectionMode: ALL` found zero targets; check tag filters match real resources.
- **Route 53 failover flips but clients still hit primary** → browser/resolver caching TTL. Set record TTL to 60s.
- **Cross-region copy doubles your bill** → expected. Cold-tier cross-region copies (`moveToColdStorageAfter`) cut storage 4x but add restore latency.
