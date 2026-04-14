# 01 — VPC Basics

## Concept
Your private network in AWS. CIDR blocks, subnets (public = has route to IGW), route tables, Internet Gateway.

## Exercises
1. **CDK VPC**: create a VPC with CIDR `10.0.0.0/16`, 2 AZs, one public + one private subnet per AZ.
   ```ts
   new ec2.Vpc(this, 'LearnVpc', {
     ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
     maxAzs: 2,
     subnetConfiguration: [
       { name: 'public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
       { name: 'private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
     ],
   });
   ```
2. **Inspect**: in console find IGW, NAT Gateway, route tables. Draw the topology on paper.
3. **Manually** (console) create a second VPC with `10.1.0.0/16`, 3 public subnets only. No CDK.
4. **CIDR math**: compute how many IPs `/24` vs `/20` vs `/28` gives. AWS reserves 5 per subnet — know which 5.
5. **Delete NAT** (expensive!) after exercise — replace with `PRIVATE_ISOLATED` subnet type if keeping.

## Verification
- `aws ec2 describe-vpcs` shows both.
- Launch a test EC2 in public subnet → has public IP and works.

## Gotchas
- NAT Gateway = $32/mo just sitting. Destroy when done.
- Default VPC exists in every region — don't use for production.
- CIDR cannot overlap if you peer later.

## Cleanup
```bash
cdk destroy
# manually delete the second VPC
```
