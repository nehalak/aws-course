# Walkthrough — Capstone 04 Global Web App

## About this capstone
You are building an active-active web application that serves users in North America, Europe, and Asia-Pacific with <200ms p95 latency and survives the total loss of a region. CloudFront fronts three regional ALBs; Fargate services run in `us-east-1`, `eu-west-1`, and `ap-southeast-1`; Aurora Global Database replicates writes from the US to the other two; DynamoDB Global Tables handle session state with local writes everywhere; WAF and ACM and Route 53 wire it all together. This capstone synthesizes everything about multi-region architecture, DNS-based failover, data consistency trade-offs, and observability across regions.

**Why it matters:** Anything with a global user base (consumer social, B2B SaaS with EU customers, gaming) hits the same constraints. Regional outages happen — the 2021 `us-east-1` outage took down half of Western consumer internet for 8 hours. Companies that had actually tested their DR survived. Companies with "we have multi-region" PowerPoint did not.

**Prerequisites:**
- `intermediate/cloudfront` — origin groups, behaviors, cache policies.
- `intermediate/route53` — latency + failover routing.
- `intermediate/aurora` — Global Database, reader endpoints.
- `intermediate/dynamodb` — Global Tables v2 (2019.11.21).
- `intermediate/fargate` — services, ALB integration.
- `advanced/waf` — managed rule groups, custom rules.
- `advanced/observability` — cross-region CloudWatch, Synthetics.

## Estimated cost
- CloudFront: $0.085/GB first 10TB, $0.0075 per 10k HTTPS requests. Modest site ~$10–20/mo.
- Route 53 hosted zone: $0.50/mo + $0.40 per million queries + $0.75/mo per latency record.
- ALB × 3 regions: $16/mo × 3 = **$48/mo** + LCU.
- Fargate × 3 regions (2 tasks each, 0.5 vCPU / 1 GB): ~$36/task/mo × 6 = **$216/mo**.
- NAT × 3 regions: $32 × 3 = **$96/mo**.
- Aurora Global Database: writer cluster (db.r6g.large × 2 = $0.29/hr × 2) + 2 reader regions with headless secondaries ($0.29/hr × 2 minimum) = **~$840/mo idle**. This is the big one.
- DynamoDB Global Tables: $1.875 per million write requests × 3 region replication + storage.
- WAF: $5 + $1/rule + $0.60 per million requests.
- Cross-region data transfer: $0.02/GB intra-AZ same region → free; cross-region $0.02/GB.
- CloudWatch Synthetics: $0.0012 per canary run — 3 canaries every 5 min ≈ $3/mo.
- Total for this capstone: **~$1,200–1,500/month if left running**. **WARN: this is BY FAR the most expensive capstone — Aurora Global alone is ~$28/day.** Tear down within hours of finishing, or swap Aurora for a single-region RDS during the walkthrough to cut ~$800.

---

## Architecture

```
           +-------------------+
  User  -> |     CloudFront    | (WAF attached here - global)
           +---------+---------+
                     |
       +-------------+-------------+
       |             |             |
   Route 53 latency + failover records
       |             |             |
    us-east-1     eu-west-1     ap-southeast-1
       |             |             |
      ALB           ALB           ALB
       |             |             |
    Fargate svc   Fargate svc   Fargate svc
       |             |             |
       +------ Aurora Global ------+   (writer: us-east-1)
       |             |             |
       +---- DynamoDB Global Tables ---+  (local writes in each region)
       |             |             |
       +------ S3 MRAP + Cognito UP (us-east-1, failover replica scripted)
```

## Step 1: Multi-region CDK project layout
> **Why:** CDK apps instantiate stacks with `env={ account, region }`. Structure the repo so the "global" stack (CloudFront, Route 53, WAF, Cognito) is separate from the "regional" stack instantiated three times.

```
global-app/
├── bin/app.ts
├── lib/
│   ├── global-stack.ts         # CloudFront, WAF, Route 53, Cognito
│   ├── regional-network.ts     # VPC per region
│   ├── regional-compute.ts     # Fargate + ALB per region
│   ├── data-stack-us.ts        # Aurora Global writer + DynamoDB GT primary
│   ├── data-stack-eu.ts        # Aurora secondary + DynamoDB replica
│   ├── data-stack-ap.ts        # Aurora secondary + DynamoDB replica
│   └── observability-stack.ts  # Synthetics in 3 regions
├── services/app/{Dockerfile,src/}
└── cdk.json
```

```typescript
// bin/app.ts
const REGIONS = ['us-east-1', 'eu-west-1', 'ap-southeast-1'] as const;
const primary = 'us-east-1';

for (const r of REGIONS) {
  new RegionalNetworkStack(app, `Net-${r}`, { env: { account, region: r } });
  new RegionalComputeStack(app, `Compute-${r}`, { env: { account, region: r } });
}
new DataStackUs(app, 'DataUs', { env: { account, region: primary } });
new DataStackEu(app, 'DataEu', { env: { account, region: 'eu-west-1' } });
new DataStackAp(app, 'DataAp', { env: { account, region: 'ap-southeast-1' } });
new GlobalStack(app, 'Global', { env: { account, region: 'us-east-1' } }); // CloudFront lives here
```

## Step 2: Per-region VPC + Fargate + ALB
> **Why:** Fargate services must live in a region-local VPC. Three copies of the same construct — parameterize by region; keep the code identical so there is no per-region snowflake.

```typescript
// lib/regional-compute.ts (excerpt)
const td = new ecs.FargateTaskDefinition(this, 'Td', { cpu: 512, memoryLimitMiB: 1024 });
td.addContainer('app', {
  image: ecs.ContainerImage.fromAsset('services/app'),
  environment: {
    REGION: this.region,
    DDB_TABLE: 'sessions',     // Global Tables — same name everywhere
    AURORA_ENDPOINT: readerEndpointPerRegion[this.region],
  },
  logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'app' }),
});

const alb = new elbv2.ApplicationLoadBalancer(this, 'Alb', { vpc, internetFacing: true });
const svc = new ecs.FargateService(this, 'Svc', { cluster, taskDefinition: td, desiredCount: 2 });
alb.addListener('Https', { port: 443, certificates: [regionalCert], defaultAction: elbv2.ListenerAction.forward([tg]) });
```

Each region needs its own ACM certificate — ACM certs are regional (except the CloudFront cert which must be in us-east-1).

## Step 3: Aurora Global Database
> **Why:** Aurora Global is purpose-built for this: sub-second cross-region replication, single writer, readers in each region. The single writer is the catch — your EU app writes go over the Atlantic to the US. For read-heavy workloads that is fine; for write-heavy, consider DynamoDB Global Tables which have multi-region active-active writes.

```typescript
// lib/data-stack-us.ts (excerpt)
import * as rds from 'aws-cdk-lib/aws-rds';

const globalCluster = new rds.CfnGlobalCluster(this, 'GlobalCluster', {
  globalClusterIdentifier: 'app-global',
  engine: 'aurora-postgresql',
  engineVersion: '15.4',
});

const primary = new rds.DatabaseCluster(this, 'Primary', {
  engine: rds.DatabaseClusterEngine.auroraPostgres({ version: rds.AuroraPostgresEngineVersion.VER_15_4 }),
  writer: rds.ClusterInstance.provisioned('writer', { instanceType: new ec2.InstanceType('r6g.large') }),
  readers: [rds.ClusterInstance.provisioned('reader', { instanceType: new ec2.InstanceType('r6g.large') })],
  vpc, defaultDatabaseName: 'app',
});
(primary.node.defaultChild as rds.CfnDBCluster).globalClusterIdentifier = globalCluster.ref;
```

In `DataStackEu` and `DataStackAp`, create **secondary clusters** attached to the same global cluster — these are read-only until promoted.

## Step 4: DynamoDB Global Tables for session data
> **Why:** Sessions need local writes (sub-10ms) in every region. Global Tables give you active-active with last-writer-wins — perfect for sessions since the last update is what you want. Not perfect for counters.

```typescript
// lib/data-stack-us.ts
const sessions = new ddb.Table(this, 'Sessions', {
  tableName: 'sessions',
  partitionKey: { name: 'sid', type: ddb.AttributeType.STRING },
  billingMode: ddb.BillingMode.PAY_PER_REQUEST,
  replicationRegions: ['eu-west-1', 'ap-southeast-1'],
  replicationTimeout: cdk.Duration.hours(2),
  timeToLiveAttribute: 'ttl',
});
```

Note: replicationRegions creates the table as a 2019.11.21 Global Table. Storage + writes are billed in every replica region.

## Step 5: S3 Multi-Region Access Point for assets
> **Why:** User-uploaded images should land close to the user and be served everywhere. MRAP gives a single global endpoint that routes to the closest replicated bucket.

```typescript
// three buckets with CRR between them, then:
new s3.CfnMultiRegionAccessPoint(this, 'Mrap', {
  name: 'app-assets',
  regions: [{ bucket: usBucket.bucketName }, { bucket: euBucket.bucketName }, { bucket: apBucket.bucketName }],
});
```

## Step 6: Cognito — regional, with failover plan
> **Why:** Cognito User Pools are **regional**. There is no multi-region user pool. The pragmatic answers: (a) put the pool in one region, accept the login unavailability during regional failover, design the app to tolerate 5-10 min auth outage; (b) run two pools and use backup-and-restore of user exports; (c) use an external IdP (Auth0, Okta) with Cognito federation.

Pick (a) for this capstone; document the trade-off in `architecture.md`.

## Step 7: CloudFront + WAF + Route 53
> **Why:** CloudFront is the globally-edge layer; WAF attached at the CloudFront distribution protects all regions at once. Route 53 latency routing sends API traffic to the closest healthy regional ALB with automatic health-check-based failover.

```typescript
// lib/global-stack.ts (excerpt)
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';

const waf = new wafv2.CfnWebACL(this, 'Waf', {
  defaultAction: { allow: {} }, scope: 'CLOUDFRONT',
  visibilityConfig: { cloudWatchMetricsEnabled: true, metricName: 'global', sampledRequestsEnabled: true },
  rules: [
    { name: 'AWSManagedCommon', priority: 1, overrideAction: { none: {} },
      statement: { managedRuleGroupStatement: { vendorName: 'AWS', name: 'AWSManagedRulesCommonRuleSet' } },
      visibilityConfig: { cloudWatchMetricsEnabled: true, metricName: 'common', sampledRequestsEnabled: true } },
  ],
});

const dist = new cloudfront.Distribution(this, 'Dist', {
  webAclId: waf.attrArn,
  defaultBehavior: {
    origin: new origins.HttpOrigin('api.example.com'),  // latency-routed DNS
    allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL,
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
  },
  additionalBehaviors: {
    '/assets/*': { origin: new origins.HttpOrigin('app-assets.accesspoint.s3-global.amazonaws.com') },
  },
});

// Route 53: latency record per region, each with its own health check
const zone = route53.HostedZone.fromLookup(this, 'Zone', { domainName: 'example.com' });
for (const [region, alb] of Object.entries(regionalAlbs)) {
  new route53.CfnRecordSet(this, `Api-${region}`, {
    hostedZoneId: zone.hostedZoneId,
    name: 'api.example.com',
    type: 'A',
    aliasTarget: { dnsName: alb.dnsName, hostedZoneId: alb.albHostedZoneId, evaluateTargetHealth: true },
    region,                 // latency policy keyed on region
    setIdentifier: region,
    healthCheckId: healthChecks[region].ref,
  });
}
```

Set DNS TTL = 60s so failover is fast but cache-friendly.

## Step 8: Synthetics canaries from 3 locations
> **Why:** The only way to actually know your global p95 < 200ms is to measure from canaries in the same 3 regions (real user monitoring is better but this is the baseline).

```typescript
// lib/observability-stack.ts (instantiated per region)
new synthetics.Canary(this, 'Canary', {
  schedule: synthetics.Schedule.rate(cdk.Duration.minutes(5)),
  test: synthetics.Test.custom({
    code: synthetics.Code.fromInline(`
      const synthetics = require('Synthetics');
      exports.handler = async () => {
        const page = await synthetics.getPage();
        const resp = await page.goto('https://api.example.com/health', { waitUntil: 'domcontentloaded' });
        if (resp.status() !== 200) throw new Error('health not 200');
      };`),
    handler: 'index.handler',
  }),
  runtime: synthetics.Runtime.SYNTHETICS_NODEJS_PUPPETEER_6_2,
});
```

## Step 9: Deploy in order
> **Why:** CloudFront + Route 53 live in the global stack but reference regional ALB DNS names; if you deploy global before the regional stacks the alias targets do not exist.

```bash
# deploy all regional stacks in parallel
npx cdk deploy Net-us-east-1 Compute-us-east-1 Net-eu-west-1 Compute-eu-west-1 Net-ap-southeast-1 Compute-ap-southeast-1 DataUs
npx cdk deploy DataEu DataAp
npx cdk deploy Global
```

## Step 10: Synthetics validation
> **Why:** If the p95 numbers are not coming back, everything else is academic.

```bash
aws cloudwatch get-metric-statistics --namespace CloudWatchSynthetics \
  --metric-name Duration --dimensions Name=CanaryName,Value=canary-us-east-1 \
  --start-time $(date -u -d '1 hour ago' +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) --period 300 --statistics p95
```

Target: all three regions report p95 < 200ms for 1 hour.

## Step 11: Chaos test — induce us-east-1 outage
> **Why:** The whole capstone's reason for existing. You do not have multi-region until you have failed over a region and come back.

```bash
# simulate by health-checking a path that now 500s on the us-east-1 app
curl -X POST https://api.us-east-1.internal/admin/fail   # your app toggles unhealthy
# within 2-3 health-check intervals, Route 53 pulls the us-east-1 record out
# traffic shifts to eu-west-1/ap-southeast-1
# measure RTO
```

**Aurora promotion** is a **manual** operational step during a real regional DR (not automatic):

```bash
aws rds failover-global-cluster --global-cluster-identifier app-global \
  --target-db-cluster-identifier arn:aws:rds:eu-west-1:...:cluster:secondary
```

RTO target: < 5 min with TTL 60s and health-check interval 30s / 3 consecutive failures.

## Step 12: Cost analysis of cross-region transfer
> **Why:** The bill surprise in multi-region architectures is always cross-region data transfer. Measure it before it ships.

```bash
# Cost Explorer with group-by "Usage Type" filter DataTransfer-Regional-Bytes
aws ce get-cost-and-usage --time-period Start=2026-04-01,End=2026-04-14 \
  --granularity DAILY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE | jq '.ResultsByTime[].Groups[] | select(.Keys[0] | contains("DataTransfer"))'
```

## Verification / success criteria
- Synthetics p95 < 200ms from all three regions over a 24h window.
- Chaos test: induced us-east-1 outage → Route 53 health check fails → traffic shifts → app responds from eu-west-1/ap-southeast-1 within 5 min.
- Aurora failover: manual promotion of eu-west-1 secondary completes and new writes succeed; on restore, old primary rejoins as secondary.
- DynamoDB Global Tables: write in us-east-1 is readable in eu-west-1 within 1s (typical) as measured by a timestamp-compare canary.
- WAF blocks a deliberate SQLi attempt (`' OR 1=1 --`) with a managed-rule block visible in WAF logs.

## Cleanup
```bash
# deletion order matters — global stack references regional
npx cdk destroy Global
npx cdk destroy DataEu DataAp          # detach secondaries first!
# then dismantle Aurora global cluster via CLI (CDK can't always)
aws rds remove-from-global-cluster --global-cluster-identifier app-global --db-cluster-identifier <eu-secondary-arn>
aws rds remove-from-global-cluster --global-cluster-identifier app-global --db-cluster-identifier <ap-secondary-arn>
aws rds delete-global-cluster --global-cluster-identifier app-global
npx cdk destroy DataUs
npx cdk destroy Compute-us-east-1 Net-us-east-1 Compute-eu-west-1 Net-eu-west-1 Compute-ap-southeast-1 Net-ap-southeast-1
```

Double-check in each region: ALBs, NAT GWs, Aurora instances, EIPs all gone. The billing nightmare comes from orphaned NATs and Aurora instances.

## Common Errors
- **CloudFront certificate error** → ACM cert for CloudFront must be in `us-east-1` regardless of your distribution's origins.
- **Route 53 health check constantly flapping** → check-interval too short + endpoint's response time > timeout; widen timeout.
- **Aurora secondary "creating" for 45 min** → normal on first attach; do not repeatedly cancel/recreate.
- **DynamoDB Global Tables write conflict** → last-writer-wins based on wall clock — clock skew between regions causes surprises; store a logical version field and resolve in app.
- **Cognito login works only in us-east-1** → expected; this is the Cognito regional limitation you documented. Plan failover via external IdP if this matters.
- **Cross-region data transfer bill spike** → Fargate in eu-west-1 writing to Aurora writer in us-east-1 = every write is transatlantic; route write-heavy endpoints to US traffic only or shard by region.
- **Synthetics canary "endpoint not reachable"** → canary runs in its region; if your ALB is internal-only, the canary needs a VPC attachment.
- **CDK `ENOTFOUND` deploying Global stack** → HostedZone lookup requires the account/region to be set explicitly; use `env` not `region: Aws.REGION`.
- **WAF rule not blocking** → `overrideAction` on managed rule groups is `none` for "use vendor action" vs `count` for "log only" — the naming is confusing and costs lots of CTFs.
