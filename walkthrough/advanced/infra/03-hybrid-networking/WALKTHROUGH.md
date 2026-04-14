# Walkthrough — 03 Hybrid Networking

## About this service
Hybrid networking connects your on-premises data center (or another cloud) to AWS. The three main tools: **Site-to-Site VPN** (IPsec tunnels over the public internet, ~1.25 Gbps per tunnel, minutes to provision), **Direct Connect (DX)** (dedicated fiber into an AWS edge location, 1/10/100 Gbps, weeks to provision), and **AWS Cloud WAN** (a managed SD-WAN overlay that stitches together VPCs, VPNs, and DX attachments across regions with a central policy). **Transit Gateway (TGW)** is the hub that most of these land on — one attachment per VPC/VPN/DX, one route table per segmentation domain.

**Why it matters:** most enterprises aren't cloud-native. You'll need to reach the mainframe in a Jersey City data center, the Active Directory in the Frankfurt office, or a partner's private API. Getting this right means planning ASNs, overlapping CIDRs, and encryption-in-transit up front.

**When to use VPN:** low-throughput, dev/test, or as a DX backup. Minutes to provision.
**When to use DX:** steady >1 Gbps traffic, latency-sensitive, or regulatory "no public internet" rule. Typically pair DX with a VPN as backup.
**When to use Cloud WAN:** 3+ regions, many VPCs, central policy-driven routing across a global footprint.
**When NOT to use:** if nothing lives on-prem, skip all of this and use VPC peering / PrivateLink / TGW within AWS.

## Estimated cost
- **VPN connection: $0.05/hour ≈ $36/month per connection** (2 tunnels included)
- VPN data transfer out: **$0.09/GB** (standard egress)
- **Transit Gateway: $0.05/attachment/hour ≈ $36/month per attachment** + $0.02/GB data processing
- For this lesson (TGW + 2 VPC attachments + 1 VPN + "on-prem" EC2 simulation): **~$120/month**
- Real Direct Connect: **$0.30/hour port charge + $0.02/GB egress** — 1 Gbps dedicated port ≈ **$220/month** just for the port (plus carrier loop-back fees, ~$500-2000/month from your telco). **We do NOT provision real DX in this lesson.**
- Cloud WAN: **core network edge $0.50/hour per region × number of regions** ≈ $360/month for 2 regions, plus attachments. Skip unless you have budget.

---

## Step 1: Scaffold + two VPCs (AWS + "on-prem" sim)
> **Why:** You don't have a real data center handy, so you simulate one with a second VPC running strongSwan on EC2. This is the standard trick in every AWS networking lab — and it actually teaches more than having real hardware because you can break things freely.

```bash
mkdir hybrid-net && cd hybrid-net
cdk init app --language=typescript
```

`lib/vpcs-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export class VpcsStack extends cdk.Stack {
  public readonly awsVpc: ec2.Vpc;
  public readonly onpremVpc: ec2.Vpc;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // AWS side — the "real" VPC
    this.awsVpc = new ec2.Vpc(this, 'AwsVpc', {
      ipAddresses: ec2.IpAddresses.cidr('10.10.0.0/16'),
      maxAzs: 2,
      natGateways: 1,
    });

    // Simulated on-prem — completely disjoint CIDR
    this.onpremVpc = new ec2.Vpc(this, 'OnpremVpc', {
      ipAddresses: ec2.IpAddresses.cidr('192.168.0.0/16'),
      maxAzs: 1,
      natGateways: 1,
      subnetConfiguration: [
        { name: 'public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
        { name: 'private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
      ],
    });

    new cdk.CfnOutput(this, 'OnpremCidr', { value: this.onpremVpc.vpcCidrBlock });
    new cdk.CfnOutput(this, 'AwsCidr', { value: this.awsVpc.vpcCidrBlock });
  }
}
```

## Step 2: Transit Gateway
> **Why:** TGW is the hub you'll grow into. Even in this small lab, terminating the VPN on TGW (instead of a Virtual Private Gateway on a single VPC) means future VPCs plug in with one attachment and get the same on-prem route for free.

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';

const tgw = new ec2.CfnTransitGateway(this, 'Tgw', {
  amazonSideAsn: 64512,                  // BGP ASN for AWS side
  autoAcceptSharedAttachments: 'enable',
  defaultRouteTableAssociation: 'enable',
  defaultRouteTablePropagation: 'enable',
  dnsSupport: 'enable',
  description: 'Learning TGW',
});

// Attach the AWS VPC
new ec2.CfnTransitGatewayAttachment(this, 'AwsVpcAttach', {
  transitGatewayId: tgw.ref,
  vpcId: awsVpc.vpcId,
  subnetIds: awsVpc.privateSubnets.map(s => s.subnetId),
});

// Route 192.168.0.0/16 (on-prem) from the AWS VPC to the TGW
for (const subnet of awsVpc.privateSubnets) {
  new ec2.CfnRoute(this, `RouteToOnprem-${subnet.node.id}`, {
    routeTableId: subnet.routeTable.routeTableId,
    destinationCidrBlock: '192.168.0.0/16',
    transitGatewayId: tgw.ref,
  });
}
```

## Step 3: strongSwan "on-prem" appliance
> **Why:** The Customer Gateway (CGW) on the AWS side is just a reference to an IP — the actual IPsec termination has to happen on a device you control. strongSwan is the open-source Linux IPsec implementation that AWS's own sample configs target.

`lib/onprem-vpn-stack.ts`:
```typescript
const onpremEip = new ec2.CfnEIP(this, 'OnpremEip');

const vpnAppliance = new ec2.Instance(this, 'VpnAppliance', {
  vpc: onpremVpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
  instanceType: new ec2.InstanceType('t3.small'),
  machineImage: ec2.MachineImage.latestAmazonLinux2023(),
  sourceDestCheck: false,  // critical — must route for other hosts
  userData: ec2.UserData.forLinux(),
});

vpnAppliance.userData.addCommands(
  'dnf install -y strongswan',
  'systemctl enable strongswan',
  // ipsec.conf + ipsec.secrets are filled in after VPN creates tunnels
);

new ec2.CfnEIPAssociation(this, 'AssocEip', {
  allocationId: onpremEip.attrAllocationId,
  instanceId: vpnAppliance.instanceId,
});

// Allow IPsec ports
vpnAppliance.connections.allowFromAnyIpv4(ec2.Port.udp(500));
vpnAppliance.connections.allowFromAnyIpv4(ec2.Port.udp(4500));

// Test host in the private subnet
const testHost = new ec2.Instance(this, 'OnpremHost', {
  vpc: onpremVpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  instanceType: new ec2.InstanceType('t3.micro'),
  machineImage: ec2.MachineImage.latestAmazonLinux2023(),
});
```

## Step 4: Customer Gateway + Site-to-Site VPN on TGW
> **Why:** The CGW is AWS's object representing your on-prem router. The VPN connection spawns two tunnels (for HA across AZs on AWS's side). Attaching it to the TGW instead of the VPC is the modern pattern.

```typescript
const cgw = new ec2.CfnCustomerGateway(this, 'Cgw', {
  bgpAsn: 65000,                             // on-prem ASN
  ipAddress: onpremEip.ref,                  // public IP of strongSwan box
  type: 'ipsec.1',
});

const vpn = new ec2.CfnVPNConnection(this, 'Vpn', {
  customerGatewayId: cgw.ref,
  type: 'ipsec.1',
  transitGatewayId: tgw.ref,
  staticRoutesOnly: false,                   // we're using BGP
});

new cdk.CfnOutput(this, 'VpnId', { value: vpn.ref });
```

Deploy:
```bash
cdk deploy VpcsStack TgwStack OnpremVpnStack VpnStack
```

## Step 5: Configure strongSwan with the generated tunnel parameters
> **Why:** AWS generates the pre-shared keys, tunnel inside CIDRs, and BGP neighbor IPs *after* the VPN resource creates. You must pull them down and render strongSwan's config — there's no way to fully automate this in CDK.

```bash
VPN_ID=$(aws cloudformation describe-stacks --stack-name VpnStack \
  --query "Stacks[0].Outputs[?OutputKey=='VpnId'].OutputValue" --output text)

aws ec2 describe-vpn-connections --vpn-connection-ids "$VPN_ID" \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' --output text \
  > vpn-config.xml

# Use the AWS-provided sample for strongSwan
aws ec2 get-vpn-connection-device-sample-configuration \
  --vpn-connection-id "$VPN_ID" \
  --vpn-connection-device-type-id <device-type-id-from-list-device-types> \
  --output text > strongswan.conf
```

Then SSH into the appliance, write `/etc/strongswan/ipsec.conf`, `/etc/strongswan/ipsec.secrets`, enable IP forwarding (`sysctl -w net.ipv4.ip_forward=1`), start strongSwan, and start FRR/Quagga for BGP on the TGW tunnel-inside IPs.

Verify tunnels come up:
```bash
aws ec2 describe-vpn-connections --vpn-connection-ids "$VPN_ID" \
  --query 'VpnConnections[0].VgwTelemetry[].[OutsideIpAddress,Status]' --output table
```

Expected once strongSwan + BGP are configured:
```
-----------------------------------
|     DescribeVpnConnections      |
+-----------------+---------------+
|  52.x.x.1       |  UP           |
|  52.x.x.2       |  UP           |
+-----------------+---------------+
```

## Step 6: BGP advertisement check
> **Why:** Static routes work but BGP is what you want in production so that convergence is automatic when a tunnel dies. You should see the on-prem CIDR propagated into the TGW route table by BGP, not by manual entry.

```bash
TGW_RT=$(aws ec2 describe-transit-gateway-route-tables \
  --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' --output text)

aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id "$TGW_RT" \
  --filters Name=type,Values=propagated \
  --query 'Routes[].[DestinationCidrBlock,State,Type]' --output table
```

Expected:
```
|  192.168.0.0/16  |  active  |  propagated  |   <-- learned via BGP from on-prem
|  10.10.0.0/16    |  active  |  propagated  |
```

## Step 7: End-to-end ping test
> **Why:** Route tables can be perfect and the tunnel still drop real packets because of a forgotten SG rule or a missing return route. The only test that matters is ICMP from one side to the other.

From the AWS private EC2:
```bash
ping 192.168.1.50   # private IP of on-prem test host
# Expected: 64 bytes from 192.168.1.50: icmp_seq=1 ttl=253 time=3.2 ms
```

From on-prem:
```bash
ping 10.10.1.20     # private IP of AWS test host
```

## Step 8: Cloud WAN (paper design)
> **Why:** Cloud WAN at $0.50/hr per regional edge is too expensive to leave running for a lab. The architecture insight is what matters. Document a design where a **global network** contains a **core network** with two regional edges (us-east-1, eu-west-1), a **core network policy** (JSON) defining three segments (prod, dev, shared), and attachments (VPCs + VPNs) automatically mapped to segments by tag.

Sample policy sketch:
```json
{
  "version": "2021.12",
  "core-network-configuration": {
    "asn-ranges": ["64512-64555"],
    "edge-locations": [
      { "location": "us-east-1" },
      { "location": "eu-west-1" }
    ]
  },
  "segments": [
    { "name": "prod",   "require-attachment-acceptance": true },
    { "name": "dev",    "require-attachment-acceptance": false },
    { "name": "shared", "require-attachment-acceptance": false }
  ],
  "segment-actions": [
    { "action": "share", "mode": "attachment-route", "segment": "shared", "share-with": "*" }
  ],
  "attachment-policies": [
    { "rule-number": 100, "conditions": [{ "type": "tag-value", "key": "segment", "value": "prod" }], "action": { "association-method": "constant", "segment": "prod" } }
  ]
}
```

Submit as `decision.md` covering: why Cloud WAN over TGW peering (centralized policy), cost comparison, and migration path.

## Step 9: Private NAT for overlapping CIDRs (paper design)
> **Why:** Acquisitions always produce CIDR collisions — both sides use `10.0.0.0/16`. You can't peer or route between overlapping networks. **Private NAT Gateway** rewrites the source IP into a non-overlapping translation CIDR so routing works without renumbering entire subnets.

Architecture narrative:
- Your VPC: `10.0.0.0/16`. Partner VPC: `10.0.0.0/16`. Direct peering impossible.
- Add a **non-overlapping transit VPC** (e.g. `100.64.0.0/16` — RFC 6598 carrier-grade NAT space).
- Deploy a Private NAT GW in the transit VPC with IP `100.64.1.1`.
- Your VPC routes `100.64.0.0/16 → TGW`. Transit VPC routes `10.0.0.0/16 → partner TGW peering`.
- Traffic from you to partner: source IP rewritten to `100.64.x.x` as it traverses the Private NAT → partner sees a non-overlapping source → can reply normally.
- Bidirectional requires two NATs (one each direction) or a translator appliance.

## Cleanup
> **Why:** TGW attachments accrue even when idle ($36/mo each). VPN connections accrue at $36/mo. EIPs cost $3.65/month when unassociated.

```bash
cdk destroy VpnStack OnpremVpnStack TgwStack VpcsStack
```

Verify no stragglers:
```bash
aws ec2 describe-vpn-connections --filters Name=state,Values=available
aws ec2 describe-transit-gateway-attachments --filters Name=state,Values=available
aws ec2 describe-addresses
```

## Common Errors
- **VPN tunnels stuck in DOWN** → most often `sourceDestCheck` still enabled on the strongSwan EC2, or missing UDP 500/4500 in the security group.
- **BGP neighbor never comes up** → wrong inside tunnel IP on one side. The sample config AWS gives you is authoritative — copy the neighbor IPs exactly.
- **Pings fail even though tunnels are UP** → missing static or propagated route on the AWS VPC private subnet route table pointing `192.168.0.0/16 → tgw-xxx`.
- **`InvalidCustomerGatewayDuplicateIpAddress`** → you already have a CGW with the same public IP and ASN. Reuse or delete the old one.
- **TGW attachment stuck in `pending`** → you attached before the TGW was in `available` state. Delete and recreate in order.
- **Cross-account TGW sharing fails** → must share via **Resource Access Manager (RAM)** first; the receiving account has to accept the RAM invitation.
- **"IpAddressMismatch" during strongSwan IKE** → your on-prem box is behind NAT but you didn't set `leftid=@<public-ip>` and `leftsourceip` properly.
