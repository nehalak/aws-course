# Walkthrough — 02 SGs & NACLs

## About these services
**Security Groups (SGs)** are stateful instance-level firewalls. Attached to ENIs (instances, ALBs, RDS, Lambda). Default: deny all; you add allow rules. Stateful = return traffic auto-allowed.

**Network ACLs (NACLs)** are stateless subnet-level firewalls. Ordered rules (lower number evaluated first). Can have `Deny` rules (SGs can't).

**Why these matter:** together they implement defense in depth. SGs do 99% of the work. NACLs catch things SGs can't (bulk IP blocks, compliance requirements for subnet-level isolation).

**When to use SGs:** always. Every resource has one.
**When to use NACLs:** subnet-wide blocks (block a bad IP from a whole subnet), compliance saying "deny SSH at subnet boundary", layered defense. Most projects leave the default NACL (allow all) and rely on SGs.

## Estimated cost
- SGs and NACLs: **100% free**
- VPC Flow Logs to CloudWatch (Step 6): **~$0.50/GB ingested**
- Total: **<$0.10/month** for this lesson

---

## Step 1: SG experiments
> **Why:** Feeling the default-deny behavior + seeing how adding rules immediately changes reality is faster than reading docs. This is foundational muscle memory.

CDK:
```typescript
const sg = new ec2.SecurityGroup(this, 'WebSg', { vpc });
sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(80), 'HTTP');
```

Try SSH:
```bash
ssh -i key.pem ec2-user@<ip>      # hangs, then timeout (SG drop)
```

Add SSH:
```typescript
sg.addIngressRule(ec2.Peer.ipv4('YOUR.IP.0.0/32'), ec2.Port.tcp(22), 'SSH');
```

## Step 2: Statefulness proof
> **Why:** Understanding statefulness is what separates SG beginners from experts. No outbound rule needed for responses to inbound requests — and vice versa.

From inside instance:
```bash
curl -v https://google.com     # works, no inbound 443 rule needed
```

## Step 3: SG-references-SG
> **Why:** The correct pattern. Instead of `allow 5432 from 10.0.0.0/24` (leaks if you re-IP), you write `allow 5432 from sg-web`. IP changes don't matter.

```typescript
const webSg = new ec2.SecurityGroup(this, 'Web', { vpc });
webSg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(80));

const dbSg = new ec2.SecurityGroup(this, 'Db', { vpc });
dbSg.addIngressRule(webSg, ec2.Port.tcp(5432), 'From web tier only');
```

## Step 4: Custom NACL
> **Why:** NACL rule numbering is ordered; lower = evaluated first. You'll use this pattern rarely but when you do it's exact — blocking one bad IP at the subnet level is fast.

```typescript
const nacl = new ec2.NetworkAcl(this, 'PrivNacl', { vpc });
nacl.addEntry('AllowInbound', {
  ruleNumber: 100,
  cidr: ec2.AclCidr.anyIpv4(),
  traffic: ec2.AclTraffic.allTraffic(),
  direction: ec2.TrafficDirection.INGRESS,
  ruleAction: ec2.Action.ALLOW,
});
nacl.addEntry('DenyBadIp', {
  ruleNumber: 90,                         // lower number = evaluated first
  cidr: ec2.AclCidr.ipv4('198.51.100.5/32'),
  traffic: ec2.AclTraffic.tcpPort(22),
  direction: ec2.TrafficDirection.INGRESS,
  ruleAction: ec2.Action.DENY,
});
```

## Step 5: Ephemeral port break
> **Why:** NACL is stateless — you must allow both directions. Breaking ephemeral outbound explicitly shows why SGs (stateful) are generally easier.

Remove the default outbound ephemeral rule. `curl` hangs. Restore the rule.

## Step 6: Verify via Flow Logs
> **Why:** Flow Logs record accept/reject decisions. Essential for debugging "why can't X reach Y?" in complex networks.

```typescript
new ec2.FlowLog(this, 'Fl', {
  resourceType: ec2.FlowLogResourceType.fromVpc(vpc),
  destination: ec2.FlowLogDestination.toCloudWatchLogs(),
});
```

Insights query:
```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| sort @timestamp desc
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **NACL rule order** — rule 100 evaluated before rule 200. Put denies low, allows high.
- **SG referencing SG across VPCs** — not allowed; must be same VPC or peered with prefix list.
- **"No route to host"** — usually route table, not SG.
