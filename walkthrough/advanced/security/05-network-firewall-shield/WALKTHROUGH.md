# Walkthrough — 05 Network Firewall & Shield

## About this service
**AWS Network Firewall (NFW)** is a managed, stateful IDS/IPS at the VPC perimeter, backed by Suricata rule syntax — you get deep packet inspection, domain filtering, TLS SNI inspection, and protocol anomaly detection. **AWS Shield Standard** (free) auto-mitigates common layer-3/4 DDoS attacks against public endpoints. **Shield Advanced** ($3,000/month, 1-year commit) adds cost protection, the DDoS Response Team (DRT), and layer-7 protection for ALB/CloudFront. **WAF** sits at layer 7 and handles rate-limiting, bot control, and application-layer rules (cheaper than Shield Advanced for most needs).

**Why it matters:** Security groups and NACLs are stateless/stateful *allow-lists* with no content awareness. NFW sees inside traffic: it blocks `evil.com` even if DNS resolves it to a permitted IP. Shield/WAF keep you up under volumetric attacks — without them, a small DDoS can rack up $10k NAT Gateway bills overnight.
**When to use:** NFW for centralized egress filtering in multi-VPC landing zones, or as a compliance boundary (PCI "DMZ" equivalent). Shield Advanced only if you've experienced repeat DDoS or have business-critical public endpoints and budget. WAF on every public ALB/CloudFront.
**When NOT to use:** NFW for a single-VPC dev account — SG+NACL+Route53 Resolver DNS Firewall is much cheaper. Shield Advanced for low-traffic sites — WAF rate-limit rules cover 95% of actual attacks seen by small sites.

## Estimated cost
- AWS Network Firewall: **$0.395/hr per firewall endpoint** (one per AZ) + **$0.065/GB** processed. 2-AZ setup = **~$577/month** baseline + traffic
- Shield Standard: **free**
- Shield Advanced: **$3,000/month, 1-year commit** + data transfer
- AWS WAF: **$5/web ACL/month** + **$1/rule/month** + **$0.60/million requests**
- Route53 Resolver DNS Firewall: **$0.60/million queries**
- Total for this lesson (NFW 2-AZ + WAF for 1 ALB, ~5 GB traffic): **~$590/month** — destroy after the lab

---

## Step 1: Deploy Network Firewall with a Suricata rule group
> **Why:** NFW sits in its own subnet; your route tables send egress traffic through it. Suricata rules let you block by domain (TLS SNI), URI, file hash, protocol anomaly — way more than SG allow-lists. Start with a simple domain-deny + DNS-log to prove the wiring.

`lib/nfw-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as nfw from 'aws-cdk-lib/aws-networkfirewall';
import * as logs from 'aws-cdk-lib/aws-logs';

export class NfwStack extends cdk.Stack {
  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', {
      maxAzs: 2,
      subnetConfiguration: [
        { name: 'public',   subnetType: ec2.SubnetType.PUBLIC,           cidrMask: 24 },
        { name: 'firewall', subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 28 },
        { name: 'app',      subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
      ],
    });

    // Stateful Suricata rule group: deny known-bad domains, log all DNS
    const suricataRules = `
#drop-malicious
drop tls any any -> any any (msg:"Block malicious domain"; \
  tls.sni; content:"evil.com"; endswith; sid:1000001; rev:1;)
drop tls any any -> any any (msg:"Block crypto mining pool"; \
  tls.sni; content:"xmrpool.eu"; endswith; sid:1000002; rev:1;)

#alert-log-all-dns
alert dns any any -> any any (msg:"DNS query"; sid:1000003; rev:1;)
`;

    const ruleGroup = new nfw.CfnRuleGroup(this, 'BadDomainsRG', {
      capacity: 100,
      ruleGroupName: 'bad-domains',
      type: 'STATEFUL',
      ruleGroup: {
        rulesSource: { rulesString: suricataRules },
        statefulRuleOptions: { ruleOrder: 'STRICT_ORDER' },
      },
    });

    // Stateless rule group: drop non-RFC1918 ingress to the firewall subnets
    const statelessRg = new nfw.CfnRuleGroup(this, 'StatelessRG', {
      capacity: 100,
      ruleGroupName: 'drop-public-ingress',
      type: 'STATELESS',
      ruleGroup: {
        rulesSource: {
          statelessRulesAndCustomActions: {
            statelessRules: [{
              priority: 10,
              ruleDefinition: {
                actions: ['aws:drop'],
                matchAttributes: {
                  sources: [{ addressDefinition: '0.0.0.0/0' }],
                  destinations: [{ addressDefinition: '10.0.0.0/8' }],
                  protocols: [6],  // TCP
                  destinationPorts: [{ fromPort: 22, toPort: 22 }],
                },
              },
            }],
          },
        },
      },
    });

    const policy = new nfw.CfnFirewallPolicy(this, 'Policy', {
      firewallPolicyName: 'corp-policy',
      firewallPolicy: {
        statelessDefaultActions: ['aws:forward_to_sfe'],
        statelessFragmentDefaultActions: ['aws:forward_to_sfe'],
        statefulEngineOptions: { ruleOrder: 'STRICT_ORDER' },
        statefulDefaultActions: ['aws:drop_established', 'aws:alert_established'],
        statefulRuleGroupReferences: [{ resourceArn: ruleGroup.attrRuleGroupArn, priority: 100 }],
        statelessRuleGroupReferences: [{ resourceArn: statelessRg.attrRuleGroupArn, priority: 10 }],
      },
    });

    const firewallSubnets = vpc.selectSubnets({ subnetGroupName: 'firewall' }).subnetIds;
    const firewall = new nfw.CfnFirewall(this, 'Firewall', {
      firewallName: 'corp-fw',
      firewallPolicyArn: policy.attrFirewallPolicyArn,
      vpcId: vpc.vpcId,
      subnetMappings: firewallSubnets.map(id => ({ subnetId: id })),
    });

    // Flow + alert logs to CloudWatch
    const logGroup = new logs.LogGroup(this, 'NFWLogs', {
      retention: logs.RetentionDays.ONE_MONTH,
    });
    new nfw.CfnLoggingConfiguration(this, 'Logging', {
      firewallArn: firewall.attrFirewallArn,
      loggingConfiguration: {
        logDestinationConfigs: [
          { logType: 'ALERT', logDestinationType: 'CloudWatchLogs',
            logDestination: { logGroup: logGroup.logGroupName } },
          { logType: 'FLOW', logDestinationType: 'CloudWatchLogs',
            logDestination: { logGroup: logGroup.logGroupName } },
        ],
      },
    });
  }
}
```

Add route-table wiring so app-subnet egress flows through the firewall endpoint (NFW in-line routing — see NFW docs for the exact RT layout; it requires edge association on the IGW side as well).

Test:
```bash
# From an EC2 in the app subnet:
curl -k https://evil.com     # → connection reset by NFW
curl https://amazonaws.com   # → works

# Check logs
aws logs tail /aws/networkfirewall/... --follow
```

## Step 2: Firewall Manager for org-wide rule sharing
> **Why:** NFW per account gets expensive and inconsistent — security team wants the same baseline (block known-bad domains, log DNS) in every account. **AWS Firewall Manager** is the fleet control plane: you define a policy once in the security account, and FMS applies it to every in-scope account via rule-group sharing.

```bash
# Enable FMS in the Organization security account
aws fms associate-admin-account --admin-account 111122223333

aws fms put-policy --policy '{
  "PolicyName":"baseline-nfw-policy",
  "ResourceType":"AWS::EC2::VPC",
  "SecurityServicePolicyData":{
    "Type":"NETWORK_FIREWALL",
    "ManagedServiceData":"{...firewall policy JSON with bad-domains RG...}"
  },
  "IncludeMap":{"ORG_UNIT":["ou-eng-xxx"]},
  "RemediationEnabled":true
}'
```

Now every VPC in OU `ou-eng-xxx` automatically gets NFW with the baseline rule group. Drift (someone deletes the firewall) is auto-remediated.

## Step 3: Simulate a DDoS and watch CloudWatch
> **Why:** You cannot validate your protection without generating load. `hey` (or `vegeta`, `k6`) hammers your ALB; you watch `RequestCount`, 5xx rates, and target CPU. This tells you the threshold at which your app breaks, which drives your WAF rate-limit value.

Deploy a basic ALB + ECS Fargate "hello world" target, then:
```bash
# From a separate EC2 (or your laptop, but NAT charges apply on the target side)
go install github.com/rakyll/hey@latest

# 500 concurrent, 2 minutes
hey -z 2m -c 500 https://your-alb.elb.amazonaws.com/

# Watch in CloudWatch:
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/your-alb/xxx \
  --start-time $(date -u -d '5 min ago' +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) \
  --period 60 --statistics Sum
```

Expect a 10–100x RequestCount spike; 5xx rate on your ECS target will climb once concurrency exceeds task capacity.

## Step 4: WAF rate-limit rule (cheap DDoS mitigation)
> **Why:** WAF rate-limit is the 80/20 of DDoS mitigation for small sites — $5/ACL/month + $1/rule, vs Shield Advanced's $3k/month. It tracks requests per source IP over 5 min; above threshold, return 429. Stops most bot floods dead.

```typescript
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';

const webAcl = new wafv2.CfnWebACL(this, 'WebAcl', {
  scope: 'REGIONAL',
  defaultAction: { allow: {} },
  visibilityConfig: { cloudWatchMetricsEnabled: true, metricName: 'webAcl', sampledRequestsEnabled: true },
  rules: [
    {
      name: 'RateLimitPerIP',
      priority: 10,
      action: { block: { customResponse: { responseCode: 429 } } },
      statement: {
        rateBasedStatement: { limit: 1000, aggregateKeyType: 'IP' }, // 1000 req / 5 min / IP
      },
      visibilityConfig: { cloudWatchMetricsEnabled: true, metricName: 'rateLimit', sampledRequestsEnabled: true },
    },
    {
      name: 'AWSCommonRuleSet',
      priority: 20,
      overrideAction: { none: {} },
      statement: { managedRuleGroupStatement: { vendorName: 'AWS', name: 'AWSManagedRulesCommonRuleSet' } },
      visibilityConfig: { cloudWatchMetricsEnabled: true, metricName: 'crs', sampledRequestsEnabled: true },
    },
    {
      name: 'AWSKnownBadInputs',
      priority: 30,
      overrideAction: { none: {} },
      statement: { managedRuleGroupStatement: { vendorName: 'AWS', name: 'AWSManagedRulesKnownBadInputsRuleSet' } },
      visibilityConfig: { cloudWatchMetricsEnabled: true, metricName: 'kbi', sampledRequestsEnabled: true },
    },
  ],
});

new wafv2.CfnWebACLAssociation(this, 'Assoc', {
  resourceArn: alb.loadBalancerArn,
  webAclArn: webAcl.attrArn,
});
```

Re-run `hey` from Step 3 at `-c 500` → within 5 minutes, requests start coming back as 429. Confirm in WAF "Sampled requests" view.

## Step 5: Route53 Resolver DNS Firewall (companion to NFW)
> **Why:** NFW blocks on TLS SNI *after* DNS resolution. DNS Firewall blocks at DNS resolution time — before the connection ever starts. Combined: DNS Firewall as first-line filter (cheap), NFW as deep-inspection second line.

```typescript
import * as r53r from 'aws-cdk-lib/aws-route53resolver';

const domainList = new r53r.CfnFirewallDomainList(this, 'BadDomains', {
  name: 'malicious-domains',
  domains: ['evil.com', 'xmrpool.eu', 'coinhive.com'],
});

const rg = new r53r.CfnFirewallRuleGroup(this, 'DnsFwRG', {
  name: 'block-bad-dns',
  firewallRules: [{
    firewallDomainListId: domainList.attrId,
    priority: 100,
    action: 'BLOCK',
    blockResponse: 'NXDOMAIN',
  }],
});

new r53r.CfnFirewallRuleGroupAssociation(this, 'DnsFwAssoc', {
  firewallRuleGroupId: rg.attrId,
  vpcId: vpc.vpcId,
  priority: 101,
  name: 'attach-to-vpc',
});
```

Test:
```bash
# From an EC2 in the VPC:
dig evil.com   # → NXDOMAIN (blocked)
dig amazonaws.com # → resolves
```

## Step 6: Shield Advanced — paper tour, don't enable
> **Why:** Shield Advanced is a $36k/year 1-year commit. You should know what it gives you before your CFO signs it; you should not enable it on a lab.

**What Shield Advanced adds over Shield Standard:**
- **Cost protection**: if a DDoS spikes your CloudFront/Route53/ALB bill, AWS refunds the overage. This alone is the main reason large orgs buy it — without it, a 1 Tbps attack can result in a $100k surprise bill.
- **DDoS Response Team (DRT) access**: 24/7 engineers who can deploy WAF rules in your account during an attack.
- **Real-time attack metrics** + layer-7 protection for ALB/CloudFront (health-based detection).
- **Global Threat Environment dashboard** — see attacks happening across AWS (abstracted) to contextualize.
- **Protection groups**: group resources and set protection rules across them.

**When it's worth it:** public SaaS with more than a few million in ARR, media sites during event windows (elections, finals), gaming/crypto (DDoS-as-extortion is real).

**When it's not:** anything protected by CloudFront + WAF rate-limit is 95% of the way there; Shield Standard is already auto-mitigating layer 3/4.

Console tour path: **WAF & Shield → Shield → Protected resources → "Subscribe to Shield Advanced"** — read the panels, do NOT subscribe.

## Cleanup
```bash
# NFW bills per hour per endpoint — destroy ASAP
cdk destroy NfwStack

# WAF ACL
aws wafv2 disassociate-web-acl --resource-arn <alb-arn>
aws wafv2 delete-web-acl --id <id> --scope REGIONAL --lock-token <token>

# DO NOT subscribe to Shield Advanced in a lab. If you did, you're locked in for 12 months.
```

## Common Errors
- **NFW routes not set correctly → traffic bypasses firewall** → the classic NFW gotcha. Traffic must go: IGW → firewall subnet route table (to firewall endpoint) → firewall endpoint → public subnet. Use VPC route table diagrams to verify.
- **Suricata rule syntax errors silently drop the whole rule group** → check `aws network-firewall describe-rule-group` output for the compiled rules; bad rules show as "capacity" under-usage.
- **Strict order vs default order for stateful rules** → Strict order is evaluation-order-sensitive (first match wins). Default order applies pass → drop → alert categories. Most people want strict.
- **WAF rate-limit doesn't block** → the metric is per-IP over 5 min window; you need `-c` high enough that a single IP exceeds the threshold. Also, limit is eventually consistent — first ~30s after limit cross might not block.
- **NFW and IDS in same VPC double-charge** → if you also run a third-party IDS on a Gateway Load Balancer, you pay for both. Pick one for production.
- **Shield Advanced auto-subscribed extra resources** → SA "Automatic application layer DDoS mitigation" adds WAF rules which themselves cost money. Check the WAF sub-bill.
- **DNS Firewall doesn't catch everything** → clients that use DoH (DNS-over-HTTPS to 8.8.8.8) bypass VPC DNS entirely. Block 1.1.1.1/8.8.8.8 egress on port 443 in SGs/NFW to force resolution through VPC DNS.
