# Walkthrough — 05 Route 53 Basics

## About this service
**Route 53** is AWS's DNS service. It hosts **zones** (like `example.com`), answers DNS queries globally, and offers features DNS providers don't: **health checks**, **failover routing**, **weighted records**, **geolocation routing**, **ALIAS records** (CNAME-like but work at apex domain).

**Why it matters:** DNS is how anything gets found. Route 53 integrates with AWS (ACM cert validation, ALB endpoints, CloudFront, failover to DR region). If you run anything on AWS with a real domain, you'll touch Route 53.

**When to use Route 53:** apps on AWS needing DNS + routing smarts; cross-region failover; multi-region traffic routing.
**When NOT to use Route 53:** if you're happy with Cloudflare/other DNS — they work fine; you'll just do more manual work integrating with AWS services. For simple single-region apps, external DNS is often cheaper.

## Estimated cost
- **Hosted zone: $0.50/month** (per zone)
- **Queries: $0.40 per million** (negligible for most sites)
- **Health checks: $0.50/month** each (HTTP/HTTPS basic)
- **Domain registration**: varies by TLD, ~$12/yr for `.com`
- Total for this lesson: **~$1/month** if you keep the zone

---

> If you don't own a domain, use `nip.io` (maps `10.0.0.1.nip.io` → `10.0.0.1`). You lose the hosted-zone experience but skip the $12/year.

## Step 1: Create hosted zone
> **Why:** A hosted zone is where your DNS records live. Creating it gives you 4 AWS nameservers you point your registrar at — that delegates the domain to AWS.

```bash
aws route53 create-hosted-zone --name example.com --caller-reference $(date +%s)
```

Copy the 4 NS records. Go to your registrar, replace default NS with these.

Verify propagation (5–60 min):
```bash
dig NS example.com
```

## Step 2: CDK record
> **Why:** Most records you create programmatically — app deploys provision DNS names. CDK's `ARecord` + `HostedZone.fromLookup` is the pattern.

```typescript
import * as route53 from 'aws-cdk-lib/aws-route53';
import * as targets from 'aws-cdk-lib/aws-route53-targets';

const zone = route53.HostedZone.fromLookup(this, 'Zone', { domainName: 'example.com' });

new route53.ARecord(this, 'TestA', {
  zone,
  recordName: 'test',                              // test.example.com
  target: route53.RecordTarget.fromIpAddresses('1.2.3.4'),
  ttl: cdk.Duration.minutes(5),
});
```

```bash
dig test.example.com +short
# 1.2.3.4
```

## Step 3: ALIAS for apex
> **Why:** DNS spec forbids CNAME at the zone apex (`example.com`). ALIAS is AWS's extension — it works like CNAME but is an A record internally. Required to point your root domain at an ALB or CloudFront.

```typescript
new route53.ARecord(this, 'ApexA', {
  zone,
  target: route53.RecordTarget.fromAlias(new targets.LoadBalancerTarget(alb)),
});
```

## Step 4: Health check + failover
> **Why:** DNS-based failover is the cheapest cross-region DR pattern. When primary is down, Route 53 serves secondary's IP.

```typescript
const hc = new route53.CfnHealthCheck(this, 'Hc', {
  healthCheckConfig: {
    type: 'HTTP',
    resourcePath: '/health',
    fullyQualifiedDomainName: 'test.example.com',
    requestInterval: 30,
    failureThreshold: 3,
  },
});
```

Shut down your backend — watch the HC go UNHEALTHY in the Route 53 console (~90 sec).

## Step 5: Private hosted zone
> **Why:** Same DNS semantics but resolvable only inside your VPC. Standard way to give AWS resources stable internal names (`db.internal.local` instead of memorizing endpoints).

```typescript
const privateZone = new route53.PrivateHostedZone(this, 'Private', {
  zoneName: 'internal.local',
  vpc,
});

new route53.CnameRecord(this, 'Db', {
  zone: privateZone,
  recordName: 'db',
  domainName: 'my-rds.abc.us-east-1.rds.amazonaws.com',
});
```

## Step 6: Weighted records
> **Why:** Weighted routing = DNS-level load balancing + gradual rollouts. "Send 10% of traffic to the new version" is one `weight` change.

```typescript
new route53.ARecord(this, 'W70', {
  zone, recordName: 'app',
  target: route53.RecordTarget.fromIpAddresses('1.1.1.1'),
  setIdentifier: 'primary', weight: 70,
});
new route53.ARecord(this, 'W30', {
  zone, recordName: 'app',
  target: route53.RecordTarget.fromIpAddresses('2.2.2.2'),
  setIdentifier: 'secondary', weight: 30,
});
```

```bash
for i in $(seq 1 20); do dig app.example.com +short; done | sort | uniq -c
# ~14x 1.1.1.1, ~6x 2.2.2.2
```

## Cleanup
```bash
cdk destroy
aws route53 delete-hosted-zone --id <zone-id>   # must be empty first
```

## Common Errors
- **`InvalidChangeBatch`** — conflicting record types (ALIAS + CNAME at same name).
- **`HostedZoneNotEmpty`** — remove all records except SOA/NS before delete.
- **DNS not resolving** — check registrar NS, then TTL, then cache.
