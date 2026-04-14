# Walkthrough — 02 EC2 Fundamentals

## About this service
**EC2 (Elastic Compute Cloud)** rents you virtual machines in AWS datacenters. You pick an **instance type** (CPU/RAM/network), an **AMI** (OS image), attach **EBS volumes** (disks), put it in a **VPC subnet**, open ports via **security groups**.

**Why it matters:** EC2 is the foundation for everything traditional — web servers, databases on self-managed VMs, bastion hosts, legacy apps. Even managed services (RDS, ECS-EC2, EKS node groups) run on EC2 internally.

**When to use EC2:** need full OS control, specific kernel/drivers, long-running persistent services, GPU workloads.
**When NOT to use EC2:** short-lived compute (use Lambda), containerized apps (use Fargate), managed DBs (use RDS). EC2 comes with patching, monitoring, scaling work you avoid by going managed.

## Estimated cost
- `t3.micro` on-demand: **$0.0104/hr (~$7.50/month)** — first 750 hrs/mo free tier for 12 months
- `m5.large`: **~$70/month**
- **EBS storage: $0.08/GB/mo for gp3** (8GB default root = ~$0.64/mo per instance)
- **Public IPv4**: **$0.005/hr (~$3.60/mo)** as of Feb 2024 — yes, public IPs cost money now
- Total for this lesson (running): **~$12/month**; destroy at end to pay nothing

---

## Step 1: Stack
> **Why:** A typical EC2 setup touches 5+ resources (instance, SG, role, user data, VPC). The CDK construct wires them together. Study each field — every one maps to a real decision.

`lib/ec2-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';

export class Ec2Stack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps & { vpc: ec2.IVpc }) {
    super(scope, id, props);

    const { vpc } = props;

    const sg = new ec2.SecurityGroup(this, 'WebSg', { vpc, allowAllOutbound: true });
    sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(80), 'HTTP');
    // Replace with your public IP:
    sg.addIngressRule(ec2.Peer.ipv4('YOUR.IP.0.0/32'), ec2.Port.tcp(22), 'SSH from home');

    const role = new iam.Role(this, 'Ec2Role', {
      assumedBy: new iam.ServicePrincipal('ec2.amazonaws.com'),
      managedPolicies: [iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSSMManagedInstanceCore')],
    });

    const userData = ec2.UserData.forLinux();
    userData.addCommands(
      'dnf install -y nginx',
      'systemctl enable --now nginx',
      'echo "hello from $(hostname)" > /usr/share/nginx/html/index.html',
    );

    const instance = new ec2.Instance(this, 'WebBox', {
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
      machineImage: ec2.MachineImage.latestAmazonLinux2023(),
      securityGroup: sg,
      role,
      userData,
    });

    new cdk.CfnOutput(this, 'PublicIp', { value: instance.instancePublicIp });
  }
}
```

`bin/app.ts`:
```typescript
import { Vpc } from 'aws-cdk-lib/aws-ec2';
import { Ec2Stack } from '../lib/ec2-stack';
import * as cdk from 'aws-cdk-lib';

const app = new cdk.App();
const env = { account: process.env.CDK_DEFAULT_ACCOUNT, region: 'us-east-1' };

class VpcStack extends cdk.Stack { vpc = new Vpc(this, 'V'); }
const v = new VpcStack(app, 'VpcStack', { env });
new Ec2Stack(app, 'Ec2Stack', { env, vpc: v.vpc });
```

```bash
curl -s ifconfig.me   # get your IP, paste in code
cdk deploy --all
```

## Step 2: curl it
> **Why:** Proves SG allows your traffic, user data ran, nginx is serving. If any one of these is broken, the test fails with a different error — good practice at distinguishing them.

```bash
IP=$(aws cloudformation describe-stacks --stack-name Ec2Stack \
  --query 'Stacks[0].Outputs[?OutputKey==`PublicIp`].OutputValue' --output text)
curl http://$IP
# hello from ip-10-0-x-x
```

## Step 3: Session Manager
> **Why:** Opening port 22 to the internet + managing SSH keys is 2010 thinking. SSM Session Manager gives you shell via AWS APIs (no key, no inbound port). You'll use this for every managed EC2 from now on.

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=Ec2Stack" \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
aws ssm start-session --target $INSTANCE_ID
```

## Step 4: Stop vs Terminate
> **Why:** Stopping preserves EBS (and you keep paying for it). Terminating removes it (unless `deleteOnTermination=false`). Real incident: "why is our storage bill so high?" → 500 stopped instances nobody remembered.

```bash
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 start-instances --instance-ids $INSTANCE_ID  # same EBS, new public IP
```

## Step 5: Instance type comparison
> **Why:** Graviton (ARM) is ~20% cheaper with comparable performance. Most modern workloads run on ARM — knowing how to switch is a real money saver.

Swap `InstanceClass.T3` → `InstanceClass.M7G`. Note the AMI must also be ARM:
```typescript
machineImage: ec2.MachineImage.latestAmazonLinux2023({ cpuType: ec2.AmazonLinuxCpuType.ARM_64 })
```

## Cleanup
```bash
cdk destroy --all
```

## Common Errors
- **`SSH: connection refused`** — nginx not up yet (user data runs async). Wait 60s.
- **Session Manager "Not connected"** — SSM agent missing, or no `AmazonSSMManagedInstanceCore` role, or no VPC endpoint (SSM + SSMMessages + EC2Messages) in private subnet.
- **User data didn't run** — check `/var/log/cloud-init-output.log`.
