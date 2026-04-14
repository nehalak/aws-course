# Walkthrough — 03 EC2 Deep

## About this service
**EC2** is AWS's original compute service, but under the hood it's now almost entirely **Nitro**: a purpose-built hypervisor + hardware cards that offload networking, storage, and security to dedicated silicon. Nitro gives you bare-metal-class performance with virtualization flexibility. On top of Nitro, AWS layers **placement groups** (cluster/spread/partition) for physical topology control, **Spot** for 70-90% discount with <2-min interruption notice, **dedicated hosts** for BYOL/compliance, and **burstable (T-series)** instances with CPU credits for bursty low-baseline workloads.

**Why it matters:** advanced EC2 knowledge is how you squeeze another 50% out of the bill without compromising reliability. Most teams run everything on on-demand m5.large and leave money on the table.
**When to use these patterns:** HPC (cluster PG), HA tiers (spread PG), batch jobs (spot), stateful singletons on licensed software (dedicated host), low-baseline dev boxes (T-series).
**When NOT to use:** if your workload fits Fargate/Lambda, the operational overhead of EC2 is rarely worth it. Don't use dedicated hosts unless license compliance forces you.

## Estimated cost
- **m5.large on-demand: $0.096/hr = $70/month** (us-east-1, Linux)
- **m4.large (non-Nitro): $0.10/hr = $73/month** — slightly more expensive and slower
- **Spot m5.large: $0.035/hr = $25/month** (~65% off, interruptible)
- **Dedicated host (m5): $4.608/hr = $3370/month** per host (fits ~48 m5.large worth of vCPU)
- **Cluster PG**: free, but same-AZ only
- **t3.small: $0.021/hr = $15/month** standard; **t3.unlimited** can silently add $0.05/vCPU-hour in burst overage
- **EBS gp3: $0.08/GB-month** + $0.005 per provisioned IOPS over 3000
- Lesson example: 3× m5.large cluster + 1 spot fleet of 10 × 4 hrs = **~$20** for the whole exercise
- Total for this lesson: **~$25 one-time** if you destroy after each step. Dedicated host step alone is $5/hour — terminate immediately.

---

## Step 1: VPC + security group scaffold (CDK)
> **Why:** Every EC2 experiment needs a VPC with SSH access and internet egress. Putting this in CDK keeps the teardown clean. We enable SSM so you don't need a keypair — `aws ssm start-session` is the modern way in.

`lib/ec2-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';

export class Ec2Stack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;
  public readonly role: iam.Role;
  public readonly sg: ec2.SecurityGroup;

  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    this.vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });
    this.role = new iam.Role(this, 'Role', {
      assumedBy: new iam.ServicePrincipal('ec2.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSSMManagedInstanceCore'),
      ],
    });
    this.sg = new ec2.SecurityGroup(this, 'Sg', { vpc: this.vpc, allowAllOutbound: true });
    this.sg.addIngressRule(this.sg, ec2.Port.allTraffic(), 'self');
  }
}
```

## Step 2: Nitro vs non-Nitro boot time + EBS throughput
> **Why:** Nitro instances boot in ~30s vs 90s+ for m4, and EBS throughput caps are 10x higher (m5.large: 4750 Mbps vs m4.large: 450 Mbps). Seeing it lets you justify migrating legacy m4/c4 workloads. m4 is being deprecated anyway — confirm you don't have any.

```typescript
new ec2.Instance(this, 'Nitro', {
  vpc: this.vpc, vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
  instanceType: new ec2.InstanceType('m5.large'),
  machineImage: ec2.MachineImage.latestAmazonLinux2023(),
  role: this.role, securityGroup: this.sg,
  blockDevices: [{
    deviceName: '/dev/xvda',
    volume: ec2.BlockDeviceVolume.ebs(30, { volumeType: ec2.EbsDeviceVolumeType.GP3 }),
  }],
});

new ec2.Instance(this, 'NonNitro', {
  vpc: this.vpc, vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
  instanceType: new ec2.InstanceType('m4.large'),
  machineImage: ec2.MachineImage.latestAmazonLinux2023(),
  role: this.role, securityGroup: this.sg,
});
```

Measure boot:
```bash
START=$(date +%s)
aws ec2 start-instances --instance-ids $ID
aws ec2 wait instance-status-ok --instance-ids $ID
echo "Boot $(( $(date +%s) - START ))s"
# m5: ~35s, m4: ~95s
```

EBS throughput:
```bash
aws ssm start-session --target $ID
# inside:
sudo dd if=/dev/zero of=/tmp/test bs=1M count=1024 oflag=direct
# m5.large: ~500 MB/s, m4.large: ~55 MB/s
```

## Step 3: Cluster placement group — 10 Gbps between nodes
> **Why:** Cluster PG physically co-locates instances on the same spine switch. Latency drops from ~1ms to <0.1ms, and flows hit 10/25/100 Gbps instead of the default 5 Gbps cap. Required for HPC, distributed training, tightly-coupled databases.

```typescript
const clusterPg = new ec2.CfnPlacementGroup(this, 'ClusterPg', { strategy: 'cluster' });
for (let i = 0; i < 3; i++) {
  new ec2.CfnInstance(this, `Node${i}`, {
    instanceType: 'c5n.4xlarge',  // required size class for 25 Gbps
    subnetId: this.vpc.privateSubnets[0].subnetId,
    placementGroupName: clusterPg.ref,
    imageId: ec2.MachineImage.latestAmazonLinux2023().getImage(this).imageId,
    iamInstanceProfile: new iam.CfnInstanceProfile(this, `Ip${i}`, {
      roles: [this.role.roleName],
    }).ref,
    securityGroupIds: [this.sg.securityGroupId],
  });
}
```

```bash
# On node A
iperf3 -s

# On node B
iperf3 -c <A-private-ip> -t 30 -P 8
# Expected: ~23 Gbit/sec within cluster PG
# Cross-AZ default: ~5 Gbit/sec
```

## Step 4: Spread placement group for HA
> **Why:** Spread PG guarantees each instance lives on separate hardware (different rack + power + network). Max 7 instances per AZ, but zero correlated failure — ideal for quorum systems (Zookeeper, etcd, leader election) where losing 2 of 3 is catastrophic.

```typescript
new ec2.CfnPlacementGroup(this, 'SpreadPg', { strategy: 'spread' });
// launch 7 instances referencing it; AWS refuses instance #8
```

Verify:
```bash
aws ec2 describe-instances --filters Name=placement-group-name,Values=spread-pg \
  --query 'Reservations[].Instances[].[InstanceId,Placement.HostId]' --output table
# HostId differs for every instance
```

## Step 5: EC2 Fleet with capacity-optimized Spot
> **Why:** `capacity-optimized` picks the pool with the deepest capacity — 3-5× lower interruption rate than `lowest-price`. Across 5 instance types × 2 AZs you get 10 diverse pools, so a single pool draining doesn't kill your fleet. This is the modern replacement for the deprecated "Spot Fleet" name.

`fleet.json`:
```json
{
  "LaunchTemplateConfigs": [{
    "LaunchTemplateSpecification": { "LaunchTemplateId": "lt-xxx", "Version": "1" },
    "Overrides": [
      { "InstanceType": "m5.large", "SubnetId": "subnet-a" },
      { "InstanceType": "m5a.large", "SubnetId": "subnet-a" },
      { "InstanceType": "m5n.large", "SubnetId": "subnet-a" },
      { "InstanceType": "m6i.large", "SubnetId": "subnet-b" },
      { "InstanceType": "m6a.large", "SubnetId": "subnet-b" }
    ]
  }],
  "TargetCapacitySpecification": {
    "TotalTargetCapacity": 10,
    "DefaultTargetCapacityType": "spot"
  },
  "SpotOptions": { "AllocationStrategy": "capacity-optimized" },
  "Type": "instant"
}
```

```bash
aws ec2 create-fleet --cli-input-json file://fleet.json
# Simulate interruption on one instance:
aws ec2 send-spot-instance-interruption \
  --target { "InstanceId": "i-xxx" } \
  --instance-interruption-behavior terminate
```

## Step 6: Spot interruption handler
> **Why:** You get a 2-minute warning via instance metadata before Spot reclaim. A handler that drains the load balancer, flushes buffers, and exits gracefully turns Spot interruptions from incidents into non-events. Without it, your users see 503s.

`/opt/spot-handler.sh`:
```bash
#!/bin/bash
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')
while true; do
  RESP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/spot/instance-action)
  if [ -n "$RESP" ]; then
    echo "Interruption: $RESP" | logger -t spot
    # Deregister from target group
    aws elbv2 deregister-targets --target-group-arn $TG_ARN \
      --targets Id=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
        http://169.254.169.254/latest/meta-data/instance-id)
    sleep 90
    systemctl stop myapp
    exit 0
  fi
  sleep 5
done
```

Install as a systemd unit via user-data. Test:
```bash
aws ec2 send-spot-instance-interruption --target '{"InstanceId":"i-xxx"}' \
  --instance-interruption-behavior terminate
# Within 5s: /var/log/messages shows "Interruption: {...action: stop...}"
```

## Step 7: Dedicated host (briefly — it's expensive)
> **Why:** Two real reasons: BYOL for Windows Server / SQL Server / Oracle with socket-based licensing, or compliance regimes that forbid multi-tenant hardware. You pay per **host**, not per instance — ~$3400/month for an m5 host whether you launch 1 or 48 instances on it.

```bash
aws ec2 allocate-hosts --instance-type m5.large \
  --availability-zone us-east-1a --quantity 1
# Host ID: h-xxx
aws ec2 run-instances --placement HostId=h-xxx --instance-type m5.large \
  --image-id ami-xxx --tenancy host
# ... verify, then immediately:
aws ec2 terminate-instances --instance-ids i-xxx
aws ec2 release-hosts --host-ids h-xxx   # stops the $4.608/hr meter
```

## Step 8: Burstable credits on t3 — exhaustion and `unlimited` mode
> **Why:** T-series instances accrue CPU credits at a baseline rate (t3.small: 24/hr for 40% of 2 vCPU) and spend them when CPU exceeds baseline. Hitting zero credits = throttled to baseline (crawls). `t3.unlimited` lets you keep bursting for $0.05/vCPU-hour overage — great for real workloads, terrifying if a bug pins CPU at 100%.

```bash
# Launch t3.small (standard mode)
aws ec2 run-instances --instance-type t3.small --image-id ami-xxx \
  --credit-specification CpuCredits=standard

# Burn credits
aws ssm start-session --target <id>
stress-ng --cpu 2 --timeout 3600  # exhausts ~150 credits in ~1hr

# Watch CloudWatch: CPUCreditBalance → 0, then CPU clipped to 20%
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUCreditBalance --dimensions Name=InstanceId,Value=<id> \
  --start-time $(date -u -d '1 hour ago' +%FT%TZ) --end-time $(date -u +%FT%TZ) \
  --period 60 --statistics Average

# Switch to unlimited
aws ec2 modify-instance-credit-specification \
  --instance-credit-specifications "InstanceId=<id>,CpuCredits=unlimited"
# CPU immediately returns to 100%, but surplus credits now bill at $0.05/vCPU-hr
```

## Cleanup
```bash
# Kill any running spot fleets first
aws ec2 delete-fleets --fleet-ids fleet-xxx --terminate-instances
# Release dedicated hosts (critical — $3400/mo each)
aws ec2 release-hosts --host-ids h-xxx
# Destroy the rest
cdk destroy
```

## Common Errors
- **`InsufficientInstanceCapacity` on cluster PG** → that AZ + instance type pool is exhausted. Try another AZ, another size in the same family, or fall back to `c5n.9xlarge`.
- **Spread PG refuses 8th instance** → hard limit is 7 per AZ. Use multiple spread PGs across AZs, or switch to partition PG.
- **Spot fleet 0 instances launched, `SpotMaxPriceTooLow`** → rare at capacity-optimized; more likely no subnet in the AZ where capacity is available. Add subnets in all AZs.
- **`send-spot-instance-interruption` returns `DryRunOperation`** → this API only works if the instance was launched from AWS Fault Injection Simulator or you're in an account with FIS enabled. Otherwise you can't self-trigger — wait for a real interruption.
- **`unlimited` t3 mysterious bill spike** → one process pinned CPU at 100% for days. Set a CloudWatch alarm on `CPUSurplusCreditsCharged > 0`.
- **m4 launch fails: `Unsupported`** → m4 is being retired in most regions. Move to m6i (intel) or m7g (Graviton).
- **Dedicated host shows "pending" for hours** → capacity exhausted in that AZ. Try another AZ or family. Host allocation isn't instant.
- **SSM session fails: `TargetNotConnected`** → instance profile missing `AmazonSSMManagedInstanceCore`, or subnet has no NAT/S3 endpoint so SSM agent can't phone home.
