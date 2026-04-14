# Walkthrough — 01 VPC Basics

## About this service
**VPC (Virtual Private Cloud)** is your private network in AWS. Every EC2, RDS, Lambda (in VPC mode) lives in one. You define the IP address space (CIDR), divide it into **subnets** (public = has route to internet via IGW; private = no inbound internet), and control traffic with **route tables**, **security groups**, and **NACLs**.

**Why it matters:** networking is the foundation. Get the CIDR wrong once and you can't peer with another VPC later. Put a database in a public subnet and you'll be on the news.

**When to use VPC:** every account uses VPC — even Lambda can be VPC-attached. For learning, create one; for production, plan CIDRs carefully up front.
**When NOT to customize:** serverless-only stacks can often use the default VPC (which AWS creates in each region). Not ideal but fine for throwaway.

## Estimated cost
- VPC itself, subnets, route tables, IGW: **free**
- **NAT Gateway: ~$32/month + $0.045/GB** processed (this is the big one — destroy when done!)
- Elastic IP for NAT: free while attached, $0.005/hr if unattached
- Total for this lesson if you keep it running: **~$35/month** due to NAT. Destroy after exercise.

---

## Step 1: Scaffold
> **Why:** Fresh CDK project keeps lessons isolated. Easier to destroy + recreate without entangling other resources.

```bash
mkdir vpc-basics && cd vpc-basics
cdk init app --language=typescript
npm install aws-cdk-lib
```

## Step 2: VPC stack
> **Why:** CDK's L2 `Vpc` construct creates sane defaults (public+private subnets per AZ, route tables, IGW, NAT). We override `natGateways: 1` to save ~$32/mo vs the default of one-per-AZ.

`lib/vpc-basics-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export class VpcBasicsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'LearnVpc', {
      ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
      maxAzs: 2,
      natGateways: 1,                // save $ — one NAT across AZs
      subnetConfiguration: [
        { name: 'public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
        { name: 'private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
      ],
    });

    new cdk.CfnOutput(this, 'VpcId', { value: vpc.vpcId });
    new cdk.CfnOutput(this, 'PublicSubnets', {
      value: vpc.publicSubnets.map(s => s.subnetId).join(','),
    });
  }
}
```

Deploy:
```bash
cdk deploy
```

Expect creation of: 1 VPC, 4 subnets, 1 IGW, 1 NAT GW, 4 route tables, Elastic IP for NAT.

## Step 3: Inspect
> **Why:** CDK hides the details. The only way to build real networking intuition is to poke at what got created and trace the route tables by hand.

```bash
aws ec2 describe-vpcs --filters "Name=tag:aws:cloudformation:stack-name,Values=VpcBasicsStack"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxx" \
  --query 'Subnets[].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' --output table
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-xxxx"
```

Sketch the topology:
```
           Internet
              |
            [IGW]
              |
  +-----------+-----------+
  |           |           |
 Public     Public      (RT: 0.0.0.0/0 → IGW)
 AZ-a       AZ-b
   |
 [NAT GW]
   |
  +-----------+-----------+
  |           |           |
 Private    Private     (RT: 0.0.0.0/0 → NAT)
 AZ-a       AZ-b
```

## Step 4: Manual second VPC
> **Why:** Creating a VPC by hand (without CDK) shows you what CDK is doing. Also, you'll hit real AWS in production where not everything is CDK-managed.

Console: **VPC → Create VPC → VPC only** → CIDR `10.1.0.0/16`, no subnets yet.
Then **Subnets → Create subnet** ×3 (public, one per AZ).
Attach an IGW; edit main route table to add `0.0.0.0/0 → igw-xxx`.

## Step 5: CIDR math
> **Why:** CIDR overlaps block VPC peering — you'll feel this pain if you pick `10.0.0.0/16` for every VPC and then try to peer them. Budget your IP space now.

| CIDR   | Total IPs | Usable (AWS reserves 5) |
|--------|-----------|--------------------------|
| /28    | 16        | 11                       |
| /24    | 256       | 251                      |
| /20    | 4096      | 4091                     |
| /16    | 65536     | 65531                    |

AWS reserves: `.0` (network), `.1` (VPC router), `.2` (DNS), `.3` (future), `.255` (broadcast).

## Step 6: Cleanup
> **Why:** NAT Gateway at $32/mo is the silent killer of AWS learning accounts. Destroy.

```bash
cdk destroy
# also delete the second VPC via console
```

## Common Errors
- **"VpcLimitExceeded"** — default 5 VPCs per region. Request quota increase or delete unused.
- **NAT Gateway creation fails** — usually no Elastic IP quota left.
- **Subnets don't auto-assign public IPs** — check `MapPublicIpOnLaunch` attribute.
