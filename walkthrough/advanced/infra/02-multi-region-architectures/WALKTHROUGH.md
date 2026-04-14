# Walkthrough — 02 Multi-Region Architectures

## About this service
"Multi-region" isn't a single AWS product — it's a **pattern** assembled from Route 53 health checks, global data services (DynamoDB Global Tables, Aurora Global Database, S3 CRR), and regional compute stacks deployed in parallel. The two archetypes are **active-active** (both regions serve live traffic, requires conflict-free or last-writer-wins data) and **active-passive** (one region serves, the other stands by — simpler, cheaper, slower to fail over).

**Why it matters:** a single AWS region has had multi-hour outages (us-east-1 Dec 2021, Jun 2023). For anything revenue-generating or life-critical, single-region is a business-continuity risk. Multi-region also lets you serve global users with low latency.

**When to use:** regulatory data-residency requirements, RTO/RPO targets under a few hours, global user base, or high-revenue apps where hours of downtime are a PR event.
**When NOT to use:** internal tools, MVPs, anything where the cost and operational complexity (2× everything, cross-region data-transfer bills, replication lag quirks) outweighs the reliability gain. Single-region with multi-AZ is already 99.99% for most apps.

## Estimated cost
Running two full stacks in us-east-1 + eu-west-1:
- **ALB × 2: ~$32/month** ($16.20 each idle)
- **Route 53 hosted zone + health checks: $0.50/zone + $0.50/HC × 2 = $1.50/month**
- **DynamoDB Global Tables (5 WCU, 5 RCU each region): ~$3/month** + cross-region replication writes $1.875/million
- **Aurora Global Database: primary writer db.r6g.large ~$170/month + secondary reader ~$170/month + cross-region replication traffic $0.02/GB**
- **S3 CRR: $0.023/GB storage × 2 + $0.02/GB replication transfer**
- **Cross-region data transfer: $0.02/GB (us-east-1 → eu-west-1)**
- Total for this lesson (Aurora is the killer): **~$380/month**. Replace Aurora Global with a DynamoDB-only exercise to drop to **~$40/month**.

---

## Step 1: Scaffold a region-parameterized CDK app
> **Why:** Multi-region CDK is about deploying the *same* stack definition twice with different `env` values. If you hardcode region names inside the stack, you'll hit bugs like the SSM parameter being resolved in the wrong region.

```bash
mkdir multi-region && cd multi-region
cdk init app --language=typescript
npm install aws-cdk-lib
```

`bin/app.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { RegionalStack } from '../lib/regional-stack';
import { GlobalDnsStack } from '../lib/global-dns-stack';

const app = new cdk.App();
const account = process.env.CDK_DEFAULT_ACCOUNT!;

const primary = new RegionalStack(app, 'Regional-USE1', {
  env: { account, region: 'us-east-1' },
  regionLabel: 'primary',
});
const secondary = new RegionalStack(app, 'Regional-EUW1', {
  env: { account, region: 'eu-west-1' },
  regionLabel: 'secondary',
});

new GlobalDnsStack(app, 'GlobalDns', {
  env: { account, region: 'us-east-1' }, // Route 53 is global but managed from any region
  primaryAlbDns: primary.albDns,
  secondaryAlbDns: secondary.albDns,
  crossRegionReferences: true,
});
```

## Step 2: Regional stack (ALB + DynamoDB Global Table member)
> **Why:** Each region owns its own compute and a *replica* of the global table. Identical stacks in both regions keep fail-over simple — you don't discover config drift at 3am.

`lib/regional-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecsp from 'aws-cdk-lib/aws-ecs-patterns';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export interface RegionalProps extends cdk.StackProps { regionLabel: string; }

export class RegionalStack extends cdk.Stack {
  public readonly albDns: string;

  constructor(scope: Construct, id: string, props: RegionalProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    const service = new ecsp.ApplicationLoadBalancedFargateService(this, 'Svc', {
      cluster,
      cpu: 256,
      desiredCount: 2,
      taskImageOptions: {
        image: ecs.ContainerImage.fromRegistry('public.ecr.aws/nginx/nginx:latest'),
        containerPort: 80,
        environment: { REGION_LABEL: props.regionLabel },
      },
      publicLoadBalancer: true,
      healthCheckGracePeriod: cdk.Duration.seconds(60),
    });
    service.targetGroup.configureHealthCheck({
      path: '/',
      healthyHttpCodes: '200-399',
      interval: cdk.Duration.seconds(15),
    });

    this.albDns = service.loadBalancer.loadBalancerDnsName;
    new cdk.CfnOutput(this, 'AlbDns', { value: this.albDns });
  }
}
```

Deploy both regions:
```bash
cdk deploy Regional-USE1 Regional-EUW1
```

Expected output per region:
```
Regional-USE1.AlbDns = Regio-Svc-XXXX-123.us-east-1.elb.amazonaws.com
Regional-EUW1.AlbDns = Regio-Svc-YYYY-456.eu-west-1.elb.amazonaws.com
```

## Step 3: DynamoDB Global Table
> **Why:** Global Tables give you multi-master async replication with **typical <1 second** cross-region lag and **last-writer-wins** conflict resolution. Requires streams enabled and identical schema in every region. Define it in *one* stack — CDK creates the replicas via the `replicas` property.

Add to a dedicated stack deployed once in `us-east-1`:

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const table = new dynamodb.TableV2(this, 'Orders', {
  tableName: 'orders',
  partitionKey: { name: 'orderId', type: dynamodb.AttributeType.STRING },
  billing: dynamodb.Billing.onDemand(),
  replicas: [
    { region: 'eu-west-1' },
  ],
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

Verify replication:
```bash
aws dynamodb put-item --region us-east-1 \
  --table-name orders \
  --item '{"orderId":{"S":"abc"},"total":{"N":"42"}}'

# Within ~1 second
aws dynamodb get-item --region eu-west-1 \
  --table-name orders --key '{"orderId":{"S":"abc"}}'
# → returns the item
```

## Step 4: Route 53 failover routing
> **Why:** DNS is your traffic switch. Failover routing sends all traffic to the primary until its health check goes unhealthy, then flips to the secondary. You can't use weighted or latency routing for active-passive — those would always send some traffic to the standby.

`lib/global-dns-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as route53 from 'aws-cdk-lib/aws-route53';

export interface GlobalDnsProps extends cdk.StackProps {
  primaryAlbDns: string;
  secondaryAlbDns: string;
}

export class GlobalDnsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: GlobalDnsProps) {
    super(scope, id, props);

    const zone = route53.HostedZone.fromLookup(this, 'Zone', { domainName: 'example.com' });

    // Health check on the primary ALB
    const primaryHc = new route53.CfnHealthCheck(this, 'PrimaryHC', {
      healthCheckConfig: {
        type: 'HTTPS',
        fullyQualifiedDomainName: props.primaryAlbDns,
        port: 443,
        resourcePath: '/',
        requestInterval: 30,
        failureThreshold: 3,
      },
    });

    // Primary record
    new route53.CfnRecordSet(this, 'Primary', {
      hostedZoneId: zone.hostedZoneId,
      name: 'app.example.com',
      type: 'CNAME',
      ttl: '60',
      resourceRecords: [props.primaryAlbDns],
      setIdentifier: 'primary-use1',
      failover: 'PRIMARY',
      healthCheckId: primaryHc.attrHealthCheckId,
    });

    // Secondary record (no health check — assumed healthy when primary fails)
    new route53.CfnRecordSet(this, 'Secondary', {
      hostedZoneId: zone.hostedZoneId,
      name: 'app.example.com',
      type: 'CNAME',
      ttl: '60',
      resourceRecords: [props.secondaryAlbDns],
      setIdentifier: 'secondary-euw1',
      failover: 'SECONDARY',
    });
  }
}
```

## Step 5: Chaos test — fail the primary
> **Why:** Paper DR plans fail in production. Until you've actually watched traffic flip, you don't know your real RTO. Simulate by breaking the health-check path.

From the primary region:
```bash
# Scale the primary service to 0 tasks — ALB targets go unhealthy
aws ecs update-service --region us-east-1 \
  --cluster RegionalUSE1Cluster --service Svc --desired-count 0
```

Watch Route 53 flip:
```bash
while true; do
  dig +short app.example.com @8.8.8.8
  sleep 5
done
```

Expected timeline:
- **t+0s:** primary ALB CNAME returned
- **t+30-90s:** health check fails 3× (interval 30s × failure threshold 3)
- **t+90-150s:** DNS returns secondary ALB CNAME (plus client-side TTL of 60s)
- **Effective RTO: ~2-3 minutes**

Restore:
```bash
aws ecs update-service --region us-east-1 \
  --cluster RegionalUSE1Cluster --service Svc --desired-count 2
```

## Step 6: Aurora Global Database (optional, expensive)
> **Why:** Relational DR is harder than key-value DR. Aurora Global gives you <1s replication lag globally with a single-click **failover** that promotes the secondary writer. RPO is ~1s, RTO ~1 min once triggered.

```typescript
import * as rds from 'aws-cdk-lib/aws-rds';

// Primary region
const globalCluster = new rds.CfnGlobalCluster(this, 'GlobalCluster', {
  globalClusterIdentifier: 'learn-global',
  engine: 'aurora-postgresql',
  engineVersion: '15.4',
});

const primary = new rds.DatabaseCluster(this, 'PrimaryCluster', {
  engine: rds.DatabaseClusterEngine.auroraPostgres({ version: rds.AuroraPostgresEngineVersion.VER_15_4 }),
  writer: rds.ClusterInstance.provisioned('w', { instanceType: new ec2.InstanceType('r6g.large') }),
  vpc,
});

const cfnCluster = primary.node.defaultChild as rds.CfnDBCluster;
cfnCluster.globalClusterIdentifier = globalCluster.ref;
```

In the secondary region, add a reader cluster referencing the same `globalClusterIdentifier`. Use **cross-region references** or SSM parameter sharing.

Promote the secondary (manual, one-way):
```bash
aws rds remove-from-global-cluster \
  --global-cluster-identifier learn-global \
  --db-cluster-identifier arn:aws:rds:eu-west-1:111111111111:cluster:secondary
```

## Step 7: S3 Cross-Region Replication + Multi-Region Access Point
> **Why:** For object storage, CRR gives eventual-consistency copies and MRAP gives a single global endpoint that routes each request to the nearest bucket.

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

const usBucket = new s3.Bucket(this, 'UsBucket', {
  versioned: true,  // required for CRR
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
});

const replicationRole = new iam.Role(this, 'ReplRole', {
  assumedBy: new iam.ServicePrincipal('s3.amazonaws.com'),
});

(usBucket.node.defaultChild as s3.CfnBucket).replicationConfiguration = {
  role: replicationRole.roleArn,
  rules: [{
    status: 'Enabled',
    destination: { bucket: 'arn:aws:s3:::my-eu-bucket' },
    priority: 1,
    filter: { prefix: '' },
    deleteMarkerReplication: { status: 'Enabled' },
  }],
};
```

## Step 8: Write the DR plan
> **Why:** When us-east-1 goes down, you will be panicking. A written plan with explicit RTO/RPO numbers, health-check thresholds, and the exact manual promotion commands is the difference between a 10-minute recovery and a 2-hour one.

Save `dr-plan.md` covering:
- **RTO target:** 5 min (DynamoDB) / 15 min (Aurora promote)
- **RPO target:** <1s for both (async replication lag)
- **Failover trigger:** automatic for Route 53; manual for Aurora Global
- **Runbook commands:** exact CLI invocations
- **Fail-back procedure:** after primary recovers, catch-up replication + Route 53 flip back
- **Game-day schedule:** quarterly chaos test

## Cleanup
> **Why:** This is one of the most expensive lessons in the course — Aurora Global alone is $340/month. Destroy everything.

```bash
cdk destroy GlobalDns
cdk destroy Regional-EUW1
cdk destroy Regional-USE1
```

If Aurora Global was deployed, also:
```bash
aws rds delete-global-cluster --global-cluster-identifier learn-global
```

Delete the DynamoDB replicas before the primary (CDK `TableV2` handles this).

## Common Errors
- **`ValidationException: Cannot update GlobalTableVersion`** → you can't change a Global Table from v1 to v2 in place. Use `TableV2` from the start.
- **Route 53 health check reports unhealthy even though ALB is fine** → your security group denies inbound from Route 53's health-checker IP ranges. Allow `0.0.0.0/0` on port 443, or add the published HC IPs.
- **Aurora secondary promotion fails: `InvalidGlobalClusterStateFault`** → you must remove the secondary from the global cluster *before* promoting. This is irreversible without a restart.
- **DynamoDB Global Table replication stuck** → check the target region has the DynamoDB service quota headroom and that the table has streams enabled with `NEW_AND_OLD_IMAGES`.
- **Cross-region CDK reference fails: `Cannot use resource in cross-environment fashion`** → set `crossRegionReferences: true` on both stacks.
- **DNS failover takes much longer than expected** → clients cache DNS beyond TTL (browsers especially). Your true RTO includes client-side caching; some browsers pin for minutes.
