# Walkthrough — 04 FinOps

## About this service
**FinOps on AWS** is the discipline of turning the AWS bill from a monthly surprise into a managed input. The core toolchain: **Cost Explorer** (query + visualize), **AWS Budgets** (alert on thresholds), **Cost Anomaly Detection** (ML-based spikes), **Compute Optimizer** (right-sizing recs for EC2/Lambda/EBS/ASG/ECS), **Savings Plans / Reserved Instances** (commit-for-discount), **Cost Allocation Tags** (slice by team/env/product), **CUR 2.0** (raw data in S3 for custom analysis).

**Why it matters:** Untagged, unmonitored AWS spend grows ~30%/year from drift alone. A tagging policy + budgets + monthly right-sizing review typically cuts 15–30% off the bill within a quarter, and Savings Plans cut another 20–40% on steady workloads.
**When to use:** the moment monthly spend passes ~$5k. Below that, the engineering time costs more than the savings.
**When NOT to use:** ephemeral PoCs / single-account sandboxes — tagging/budget overhead doesn't pay back.

## Estimated cost
- **Cost Explorer**: UI free; **$0.01 per API request** (programmatic access).
- **AWS Budgets**: first 2 budgets free/account, **$0.02/budget/day** beyond that (~$0.60/mo each).
- **Cost Anomaly Detection**: **free**.
- **Compute Optimizer**: free (enhanced recs for Lambda require opt-in but still free).
- **CUR 2.0 in S3**: data is free; S3 storage ~$0.023/GB-mo (a full year of CUR for a $50k/mo account is ~5 GB).
- **Savings Plans / RIs**: not a cost — an instrument. 1-yr no-upfront Compute SP typically saves **~27%** on Fargate/Lambda/EC2; 3-yr all-upfront ~**54%**.
- **S3 Intelligent-Tiering**: **$0.0025 per 1,000 objects/mo** monitoring fee; saves ~40–70% on cold data.
- Total for this lesson: **~$5/month** (5 budgets + a few CE API calls + CUR storage).

---

## Step 1: Tagging strategy enforced by SCP
> **Why:** You can't slice costs by a tag that isn't there. SCPs enforce tag-on-create at the Organizations boundary; AWS Config rules catch drift for taggable-later resources.

Required tags: `CostCenter`, `Environment`, `Owner`, `Application`.

`scps/deny-create-without-tags.json` (EC2/RDS/S3 example):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRunInstancesWithoutTags",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "Null": {
          "aws:RequestTag/CostCenter":  "true",
          "aws:RequestTag/Environment": "true",
          "aws:RequestTag/Owner":       "true",
          "aws:RequestTag/Application": "true"
        }
      }
    },
    {
      "Sid": "DenyCreateBucketWithoutTags",
      "Effect": "Deny",
      "Action": "s3:CreateBucket",
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/CostCenter":  "true",
          "aws:RequestTag/Environment": "true",
          "aws:RequestTag/Owner":       "true",
          "aws:RequestTag/Application": "true"
        }
      }
    }
  ]
}
```

Activate the tags as **cost-allocation tags** (required before they appear in CE):
```bash
aws ce update-cost-allocation-tags-status --cost-allocation-tags-status \
  'TagKey=CostCenter,Status=Active' \
  'TagKey=Environment,Status=Active' \
  'TagKey=Owner,Status=Active' \
  'TagKey=Application,Status=Active'
```

Config rule for drift detection:
```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName":"required-tags",
  "Source":{"Owner":"AWS","SourceIdentifier":"REQUIRED_TAGS"},
  "InputParameters":"{\"tag1Key\":\"CostCenter\",\"tag2Key\":\"Environment\",\"tag3Key\":\"Owner\",\"tag4Key\":\"Application\"}"
}'
```

## Step 2: Cost Explorer — top 10 services, group by CostCenter
> **Why:** The 80/20 is real: usually 3 services account for 80% of spend. Group-by-tag reveals which team/product is actually burning money.

```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-03-14,End=2026-04-14 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[].{svc:Keys[0],amt:Metrics.UnblendedCost.Amount}' \
  --output table

aws ce get-cost-and-usage \
  --time-period Start=2026-03-14,End=2026-04-14 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=CostCenter
```

Typical output:
```
--------------------------------
|  svc                   | amt |
|------------------------|-----|
| Amazon EC2             | 8120|
| Amazon RDS             | 2430|
| Amazon S3              |  940|
| AWS Lambda             |  610|
| Amazon CloudWatch      |  380|
--------------------------------
```

## Step 3: Cost Anomaly Detection
> **Why:** Budgets alert on thresholds you set; anomaly detection learns your baseline and catches the spikes you didn't think to alert on (runaway Lambda recursion, forgotten NAT, etc.).

```bash
# Monitor by service
aws ce create-anomaly-monitor --anomaly-monitor '{
  "MonitorName":"by-service",
  "MonitorType":"DIMENSIONAL",
  "MonitorDimension":"SERVICE"
}'

# Subscription — alert Slack/email on >$100 daily anomaly
aws ce create-anomaly-subscription --anomaly-subscription '{
  "SubscriptionName":"finops-slack",
  "Threshold": 100,
  "Frequency":"IMMEDIATE",
  "MonitorArnList":["arn:aws:ce::123456789012:anomalymonitor/..."],
  "Subscribers":[{"Type":"EMAIL","Address":"finops@example.com"}]
}'
```

Also create a second monitor on the `Application` tag so per-app spikes don't get averaged out.

## Step 4: Budgets — overall + per-application
> **Why:** One overall budget is necessary but blunt. Per-app budgets (via tag filter) send the alert directly to the team that can fix it — faster triage, less finger-pointing.

```bash
# Overall $100/mo, alert at 80%
aws budgets create-budget --account-id 123456789012 --budget '{
  "BudgetName":"overall-100",
  "BudgetType":"COST",
  "TimeUnit":"MONTHLY",
  "BudgetLimit":{"Amount":"100","Unit":"USD"}
}' --notifications-with-subscribers '[{
  "Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},
  "Subscribers":[{"SubscriptionType":"EMAIL","Address":"finops@example.com"}]
}]'

# Per-application budget, filtered by tag
aws budgets create-budget --account-id 123456789012 --budget '{
  "BudgetName":"app-orders-500",
  "BudgetType":"COST",
  "TimeUnit":"MONTHLY",
  "BudgetLimit":{"Amount":"500","Unit":"USD"},
  "CostFilters":{"TagKeyValue":["user:Application$orders"]}
}' --notifications-with-subscribers '[{
  "Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},
  "Subscribers":[{"SubscriptionType":"EMAIL","Address":"orders-team@example.com"}]
}]'
```

## Step 5: Compute Optimizer — resize one instance
> **Why:** CO looks at 14 days of CloudWatch metrics and recommends a smaller/different instance type. The recs are free; acting on one instance validates the workflow before you touch dozens.

```bash
aws compute-optimizer update-enrollment-status --status Active

# EC2 recs
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[?finding==`Overprovisioned`].{id:instanceArn,cur:currentInstanceType,rec:recommendationOptions[0].instanceType,savings:recommendationOptions[0].estimatedMonthlySavings.value}' \
  --output table
```

Typical:
```
------------------------------------------------------------------
| id                          | cur      | rec       | savings   |
|-----------------------------|----------|-----------|-----------|
| arn:...:instance/i-0abc123  | m5.xlarge| m6g.large | 76.50     |
```

Resize one instance and watch CloudWatch CPU/memory for a week to confirm. Lambda recs:
```bash
aws compute-optimizer get-lambda-function-recommendations \
  --query 'lambdaFunctionRecommendations[?finding==`NotOptimized`].{fn:functionArn,cur:currentMemorySize,rec:memorySizeRecommendationOptions[0].memorySize}' \
  --output table
```

## Step 6: Savings Plans analysis — don't buy yet
> **Why:** SP/RI commitments are 1 or 3 years and non-cancellable. Run the recommendation tool, understand the break-even math, but don't purchase as part of a learning lesson.

```bash
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.SavingsPlansPurchaseRecommendationDetails[0]' \
  --output json
```

Compare across options:
| Plan                 | Term | Upfront | Typical savings | Flexibility                     |
|----------------------|------|---------|-----------------|---------------------------------|
| Compute SP           | 1-yr | None    | ~27%            | EC2 + Fargate + Lambda, any region/family |
| Compute SP           | 3-yr | All     | ~54%            | same, biggest commitment        |
| EC2 Instance SP      | 1-yr | None    | ~40%            | locked to family + region       |
| Standard RI (EC2)    | 1-yr | None    | ~40%            | locked to family; tradeable     |
| RDS RI               | 1-yr | None    | ~35%            | locked to engine + class + AZ   |

Rule of thumb: buy SP to cover ~70% of your 30-day-minimum baseline; leave the peak on on-demand.

## Step 7: S3 Intelligent-Tiering via lifecycle
> **Why:** Intelligent-Tiering auto-moves objects to cheaper tiers after 30/90/180 days of no access. For unknown/mixed access patterns it always wins vs Standard — $0.0025/1k-object monitoring fee is trivial.

```typescript
import { Bucket, StorageClass } from 'aws-cdk-lib/aws-s3';
import { Duration } from 'aws-cdk-lib';

const data = new Bucket(this, 'Data', {
  lifecycleRules: [{
    transitions: [
      { storageClass: StorageClass.INTELLIGENT_TIERING, transitionAfter: Duration.days(0) },
    ],
  }],
  intelligentTieringConfigurations: [{
    name: 'archive-access',
    archiveAccessTierTime: Duration.days(90),
    deepArchiveAccessTierTime: Duration.days(180),
  }],
});
```

Bulk-apply to existing buckets:
```bash
for b in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
  aws s3api put-bucket-intelligent-tiering-configuration \
    --bucket "$b" --id default-it \
    --intelligent-tiering-configuration '{
      "Id":"default-it","Status":"Enabled",
      "Tierings":[
        {"Days":90,"AccessTier":"ARCHIVE_ACCESS"},
        {"Days":180,"AccessTier":"DEEP_ARCHIVE_ACCESS"}
      ]
    }'
done
```

## Step 8: Monthly FinOps review checklist
> **Why:** FinOps is a loop, not a one-time cleanup. A monthly 30-min review keeps spend in line without quarterly fire drills.

```markdown
# Monthly FinOps Review
Target: review by the 10th, act by the 15th.

- [ ] Top 10 services MoM delta — any >20% gainers?
- [ ] Per-app budget utilization — who's over 80%?
- [ ] Cost anomaly list for the month — each has a ticket/root cause?
- [ ] Compute Optimizer: accept at least 2 recs (EC2 or Lambda).
- [ ] Savings Plan utilization >95%? If <95%, stop buying SPs or migrate workload back onto SP-eligible services.
- [ ] Unused: idle ALBs (<1 req/min for 7d), unattached EBS volumes, old snapshots (>90d), detached EIPs. Delete.
- [ ] NAT Gateway data processing — anything to move to a VPC endpoint?
```

## Cleanup
```bash
# Budgets
aws budgets delete-budget --account-id 123456789012 --budget-name overall-100
aws budgets delete-budget --account-id 123456789012 --budget-name app-orders-500

# Anomaly detection
aws ce delete-anomaly-subscription --subscription-arn arn:aws:ce::...:anomalysubscription/...
aws ce delete-anomaly-monitor --monitor-arn arn:aws:ce::...:anomalymonitor/...

# DO NOT BUY SAVINGS PLANS OR RIs FOR THIS LESSON.
# They are non-cancellable 1–3 year commitments.
```

## Common Errors
- **Tag doesn't appear in Cost Explorer** → you forgot `update-cost-allocation-tags-status Active`, or the tag was activated <24h ago (CE backfills daily).
- **SCP `DenyRunInstances` blocks your own CI** → CI role also must include the tags in `aws:RequestTag`. Fix the CI IaC, not the SCP.
- **Budget alerts stop firing** → you hit the 2-free limit and billing is paused for the account; check Billing console.
- **Compute Optimizer returns `OptOutError`** → enrollment is account-level; run `update-enrollment-status --status Active` first, then wait 12 h for recs.
- **SP utilization dropped below 90%** → workload migrated off SP-eligible compute (e.g. moved Lambda to Fargate Spot). Move workload back or sell unused SPs in the marketplace (EC2 SP only; Compute SP is not sellable).
- **CUR in S3 shows no data** → S3 bucket policy missing the `billingreports.amazonaws.com` grant; re-create the report and AWS re-applies the policy.
- **`aws ce get-cost-and-usage` returns 0 for today** → CE has ~24h delay. Query yesterday or earlier.
