# Walkthrough — 01 VPC Advanced

## About this service
Once you're past the default VPC, the interesting networking levers kick in: **NAT Gateways** (managed egress for private subnets), **VPC endpoints** (private AWS-service access without touching the internet), **VPC Peering** (1:1 connection between two VPCs), **Transit Gateway** (hub-and-spoke for many VPCs and on-prem), and **Flow Logs** (packet-header audit trail). Together these shape how traffic actually moves across your account.

**Why it matters:** a badly-sized NAT or missing endpoint is the #1 source of "why is my AWS bill $400 this month?" The answer is usually data-processing charges on the NAT Gateway that a $0 S3 Gateway Endpoint would have eliminated.

**When to use each:**
- NAT GW: private subnet needs outbound internet (package installs, third-party APIs).
- Gateway endpoint (S3, DynamoDB): always. Free. No reason not to.
- Interface endpoint: private-only access to SSM, ECR, Secrets Manager, etc.
- Peering: 2–3 VPCs, full mesh manageable.
- Transit Gateway: 4+ VPCs, on-prem VPN/DX, or shared services topologies.

**When NOT to use:**
- Don't deploy a NAT Gateway if the workload is 100% covered by gateway/interface endpoints.
- Don't use peering at scale — it's O(n²) route table maintenance.
- Don't enable Flow Logs on ALL traffic in a chatty VPC — filter to REJECT only to save $.

## Estimated cost
- **NAT Gateway: ~$32/month** + $0.045/GB processed
- **Interface endpoint: ~$7.30/month/AZ** + $0.01/GB
- **Gateway endpoint (S3, DynamoDB): free**
- **Transit Gateway: ~$36/month per attachment** + $0.02/GB
- **VPC Peering: free per hour**, $0.01/GB cross-AZ
- **Flow Logs to S3:** $0.50/GB ingested
- Total for this lesson if left running: **~$110/month** (NAT + TGW + 2 interface endpoints). Destroy promptly.

---

## Step 1: Scaffold and deploy a VPC with NAT
> **Why:** We need a baseline VPC with private subnets to exercise all the egress patterns. One NAT GW shared across AZs to keep cost down during learning.

```bash
mkdir vpc-advanced && cd vpc-advanced
cdk init app --language=typescript
npm install aws-cdk-lib
```

`lib/vpc-advanced-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as logs from 'aws-cdk-lib/aws-logs';

export class VpcAdvancedStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'AdvVpc', {
      ipAddresses: ec2.IpAddresses.cidr('10.10.0.0/16'),
      maxAzs: 2,
      natGateways: 1,
      subnetConfiguration: [
        { name: 'public',   subnetType: ec2.SubnetType.PUBLIC,               cidrMask: 24 },
        { name: 'private',  subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,  cidrMask: 24 },
        { name: 'isolated', subnetType: ec2.SubnetType.PRIVATE_ISOLATED,     cidrMask: 24 },
      ],
    });

    // --- Step 2: Gateway endpoints (free) ---
    vpc.addGatewayEndpoint('S3Endpoint', {
      service: ec2.GatewayVpcEndpointAwsService.S3,
      subnets: [{ subnetType: ec2.SubnetType.PRIVATE_ISOLATED }],
    });
    vpc.addGatewayEndpoint('DynamoEndpoint', {
      service: ec2.GatewayVpcEndpointAwsService.DYNAMODB,
      subnets: [{ subnetType: ec2.SubnetType.PRIVATE_ISOLATED }],
    });

    // --- Step 3: Interface endpoints for SSM Session Manager ($$) ---
    const ssmSg = new ec2.SecurityGroup(this, 'SsmEpSg', { vpc, allowAllOutbound: true });
    ssmSg.addIngressRule(ec2.Peer.ipv4(vpc.vpcCidrBlock), ec2.Port.tcp(443));

    for (const svc of [
      ec2.InterfaceVpcEndpointAwsService.SSM,
      ec2.InterfaceVpcEndpointAwsService.SSM_MESSAGES,
      ec2.InterfaceVpcEndpointAwsService.EC2_MESSAGES,
    ]) {
      vpc.addInterfaceEndpoint(`Ep-${svc.shortName}`, {
        service: svc,
        subnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
        securityGroups: [ssmSg],
        privateDnsEnabled: true,
      });
    }

    // --- Test EC2 in isolated subnet (no NAT path) ---
    const role = new iam.Role(this, 'Ec2Role', {
      assumedBy: new iam.ServicePrincipal('ec2.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonSSMManagedInstanceCore'),
        iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonS3ReadOnlyAccess'),
      ],
    });

    new ec2.Instance(this, 'IsolatedHost', {
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
      machineImage: ec2.MachineImage.latestAmazonLinux2023(),
      role,
    });

    // --- Step 6: Flow Logs (REJECT only) ---
    const flowBucket = new s3.Bucket(this, 'FlowLogs', {
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });
    new ec2.FlowLog(this, 'VpcFlow', {
      resourceType: ec2.FlowLogResourceType.fromVpc(vpc),
      destination: ec2.FlowLogDestination.toS3(flowBucket),
      trafficType: ec2.FlowLogTrafficType.REJECT,
    });

    new cdk.CfnOutput(this, 'VpcId', { value: vpc.vpcId });
    new cdk.CfnOutput(this, 'FlowLogsBucket', { value: flowBucket.bucketName });
  }
}
```

```bash
cdk bootstrap
cdk deploy
```

## Step 2: Gateway endpoints for S3 + DynamoDB
> **Why:** Gateway endpoints are free and sit in your route tables as prefix lists. Traffic to `s3.us-east-1.amazonaws.com` from the isolated subnet flows through the endpoint (no NAT data-processing cost). This is the single biggest easy win for cost.

Verify via Session Manager to the isolated host:
```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=VpcAdvancedStack" \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)

aws ssm start-session --target $INSTANCE_ID
# inside the session:
aws s3 ls
curl -v https://dynamodb.us-east-1.amazonaws.com 2>&1 | head
```

Expected — S3 lists without a public IP or NAT route:
```
2024-...  my-bucket-a
2024-...  my-bucket-b
```

Inspect route tables — the isolated RT now has a managed prefix list entry:
```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[].Routes[?GatewayId!=null && starts_with(GatewayId, `vpce-`)]'
```

## Step 3: Interface endpoints for SSM
> **Why:** Session Manager from an isolated (no-NAT) subnet requires three interface endpoints: `ssm`, `ssmmessages`, `ec2messages`. Each costs ~$7.30/mo/AZ — so a 2-AZ deploy of all three is ~$44/mo. Real lesson: pick AZs carefully.

The CDK above created them. Confirm:
```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'VpcEndpoints[].[ServiceName,VpcEndpointType,State]' --output table
```

Expected:
```
---------------------------------------------------------------------
|                      DescribeVpcEndpoints                         |
+-----------------------------------------------+---------+---------+
|  com.amazonaws.us-east-1.s3                   | Gateway | available |
|  com.amazonaws.us-east-1.dynamodb             | Gateway | available |
|  com.amazonaws.us-east-1.ssm                  | Interface| available|
|  com.amazonaws.us-east-1.ssmmessages          | Interface| available|
|  com.amazonaws.us-east-1.ec2messages          | Interface| available|
+-----------------------------------------------+---------+---------+
```

Open Session Manager on `IsolatedHost` — it works even though the instance has no route to `0.0.0.0/0`.

## Step 4: VPC Peering between two VPCs
> **Why:** Peering is the cheapest way to connect two VPCs. Non-overlapping CIDRs are a hard requirement — this is where you feel the cost of picking `10.0.0.0/16` everywhere earlier.

Add a second VPC and a peering to the stack:

```typescript
const vpcB = new ec2.Vpc(this, 'VpcB', {
  ipAddresses: ec2.IpAddresses.cidr('10.20.0.0/16'),
  maxAzs: 2,
  natGateways: 0,
  subnetConfiguration: [
    { name: 'private', subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 24 },
  ],
});

const peering = new ec2.CfnVPCPeeringConnection(this, 'Peering', {
  vpcId: vpc.vpcId,
  peerVpcId: vpcB.vpcId,
});

// Add routes both sides
vpc.isolatedSubnets.forEach((s, i) => {
  new ec2.CfnRoute(this, `RouteAtoB${i}`, {
    routeTableId: s.routeTable.routeTableId,
    destinationCidrBlock: '10.20.0.0/16',
    vpcPeeringConnectionId: peering.ref,
  });
});
vpcB.isolatedSubnets.forEach((s, i) => {
  new ec2.CfnRoute(this, `RouteBtoA${i}`, {
    routeTableId: s.routeTable.routeTableId,
    destinationCidrBlock: '10.10.0.0/16',
    vpcPeeringConnectionId: peering.ref,
  });
});
```

`cdk deploy`. Launch a probe instance in VPC B, `ping 10.10.x.x` from it. Works after SGs are opened.

## Step 5: Transit Gateway for 3 VPCs
> **Why:** Peering is O(n²). For 3 VPCs you'd need 3 peerings. TGW converts that to 1 TGW + N attachments. Above 3–4 VPCs the switch is economic, not just aesthetic.

```typescript
const tgw = new ec2.CfnTransitGateway(this, 'Tgw', {
  description: 'Learning TGW',
  defaultRouteTableAssociation: 'enable',
  defaultRouteTablePropagation: 'enable',
});

function attach(vpcRef: ec2.IVpc, id: string) {
  return new ec2.CfnTransitGatewayAttachment(this, id, {
    transitGatewayId: tgw.ref,
    vpcId: vpcRef.vpcId,
    subnetIds: vpcRef.isolatedSubnets.map(s => s.subnetId),
  });
}
```

After deploy, add TGW routes in each VPC's subnet route tables pointing `10.0.0.0/8 → tgw-xxx`. Verify 3-way routing with ping.

## Step 6: Flow Logs and Athena
> **Why:** You can't debug networking you can't see. Flow Logs give you per-flow records (srcIP, dstIP, ports, action). Filtering to REJECT only catches security-group drops — the usual suspect.

The stack wrote logs to S3. Create an Athena table (replace bucket name):

```sql
CREATE EXTERNAL TABLE vpc_flow_logs (
  version int, account_id string, interface_id string,
  srcaddr string, dstaddr string, srcport int, dstport int,
  protocol int, packets int, bytes bigint, start int, `end` int,
  action string, log_status string
)
PARTITIONED BY (dt string)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '
LOCATION 's3://<flow-logs-bucket>/AWSLogs/<account>/vpcflowlogs/us-east-1/';
```

Query rejects:
```sql
SELECT srcaddr, dstaddr, dstport, count(*) c
FROM vpc_flow_logs
WHERE action='REJECT'
GROUP BY 1,2,3 ORDER BY c DESC LIMIT 20;
```

## Cleanup
> **Why:** TGW attachments ($36/mo each) and interface endpoints ($7/mo/AZ each) are the expensive bits. Destroy.

```bash
cdk destroy
# confirm no orphaned endpoint ENIs or TGW attachments remain:
aws ec2 describe-vpc-endpoints --query 'VpcEndpoints[].[VpcEndpointId,State]' --output table
aws ec2 describe-transit-gateway-attachments --query 'TransitGatewayAttachments[].State'
```

## Common Errors
- **"RouteAlreadyExists"** on peering routes → a previous deploy left them; delete the conflicting route manually, redeploy.
- **SSM session fails to open from isolated subnet** → missing `ssmmessages` or `ec2messages` endpoint, or SG blocking 443 from VPC CIDR.
- **"InvalidVpcPeeringConnectionID.NotFound"** → CDK applied routes before peering became `active`; add a dependency (`peering.node.addDependency(...)`) or retry.
- **Gateway endpoint created but S3 still slow/NAT-billed** → endpoint not associated with the subnet's route table. Check `subnets:` parameter.
- **TGW attachments stuck `pending`** → subnets in the attachment must be in different AZs; one per AZ max per VPC.
