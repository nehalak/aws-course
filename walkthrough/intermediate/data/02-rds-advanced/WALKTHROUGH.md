# Walkthrough — 02 RDS Advanced & Aurora

## About this service
**Aurora** is AWS's own MySQL/Postgres-compatible engine. Unlike classic RDS, Aurora *separates storage from compute*: a single 6-way replicated storage layer backs one writer and up to 15 readers. Readers add minutes, not hours. Failover is ~30 seconds. Aurora Serverless v2 scales compute in 0.5 ACU increments (half-second granularity).

**Why it matters:** Aurora is usually 3-5x the throughput of RDS Postgres for the same compute cost, with faster failover and cheaper replicas. It is the default relational choice for new AWS workloads that need scale.

**When to use:** SaaS/multi-tenant apps, read-heavy workloads, apps needing <1 min failover, variable workloads (Serverless v2).
**When NOT to use:** small toy projects (t3.micro RDS is cheaper), engines Aurora doesn't support (Oracle/SQL Server — use RDS), sustained low-throughput (standard RDS cheaper idle).

## Estimated cost
- **Aurora Postgres `db.t4g.medium`: ~$49/month** writer + $49/month per reader (us-east-1)
- **Storage: $0.10/GB/month** (pay for peak, doesn't shrink)
- **I/O: $0.20 per million requests** (or Aurora I/O-Optimized for I/O-heavy)
- **Aurora Serverless v2: $0.12/ACU-hour** (0.5 ACU min = $43/month idle)
- **RDS Proxy: $0.015/vCPU-hour** of target instance (~$11/month for db.t4g.medium)
- **Performance Insights: free** for 7-day retention
- Total for this lesson: **~$150/month** if left running. Destroy after!

---

## Step 1: Aurora cluster with writer + 2 readers
> **Why:** This is production-shape: writer in one AZ, readers in others. The reader endpoint round-robins across readers. Apps should split: writes to writer endpoint, reads to reader endpoint. `iamAuthentication` + Secrets Manager = no hardcoded passwords.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';

export class AuroraStack extends cdk.Stack {
  public readonly cluster: rds.DatabaseCluster;

  constructor(scope: any, id: string, props: cdk.StackProps & { vpc: ec2.IVpc }) {
    super(scope, id, props);
    const { vpc } = props;

    this.cluster = new rds.DatabaseCluster(this, 'AuroraPg', {
      engine: rds.DatabaseClusterEngine.auroraPostgres({
        version: rds.AuroraPostgresEngineVersion.VER_16_2,
      }),
      writer: rds.ClusterInstance.provisioned('Writer', {
        instanceType: ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.MEDIUM),
        enablePerformanceInsights: true,
      }),
      readers: [
        rds.ClusterInstance.provisioned('Reader1', {
          instanceType: ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.MEDIUM),
        }),
        rds.ClusterInstance.provisioned('Reader2', {
          instanceType: ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.MEDIUM),
        }),
      ],
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      credentials: rds.Credentials.fromGeneratedSecret('postgres'),
      defaultDatabaseName: 'appdb',
      iamAuthentication: true,
      storageEncrypted: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    new cdk.CfnOutput(this, 'WriterEp', { value: this.cluster.clusterEndpoint.hostname });
    new cdk.CfnOutput(this, 'ReaderEp', { value: this.cluster.clusterReadEndpoint.hostname });
    new cdk.CfnOutput(this, 'SecretArn', { value: this.cluster.secret!.secretArn });
  }
}
```

## Step 2: Failover test
> **Why:** Aurora failover is DNS-driven: on writer failure, a reader is promoted and writer DNS flips. Apps with connection pooling may hold stale connections — proxy or retry logic recovers them. Testing failover NOW is the only way to know your app tolerates it.

```bash
aws rds failover-db-cluster --db-cluster-identifier $CLUSTER_ID
# Watch: writer endpoint now points to a former reader in ~30s
dig +short $WRITER_ENDPOINT
```

Expected: different IP within ~30 seconds; active connections dropped; app reconnects.

## Step 3: Aurora Serverless v2
> **Why:** Serverless v2 scales in 0.5 ACU steps (1 ACU ≈ 2 GiB RAM + proportional CPU) within seconds. Great for spiky workloads. Min 0.5 ACU still costs ~$43/month — NOT truly scale-to-zero like Serverless v1.

```typescript
const serverlessCluster = new rds.DatabaseCluster(this, 'AuroraSv2', {
  engine: rds.DatabaseClusterEngine.auroraPostgres({
    version: rds.AuroraPostgresEngineVersion.VER_16_2,
  }),
  writer: rds.ClusterInstance.serverlessV2('Writer'),
  readers: [rds.ClusterInstance.serverlessV2('Reader1', { scaleWithWriter: true })],
  serverlessV2MinCapacity: 0.5,
  serverlessV2MaxCapacity: 8,
  vpc: props.vpc,
  credentials: rds.Credentials.fromGeneratedSecret('postgres'),
  defaultDatabaseName: 'appdb',
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

Load-test with pgbench from a bastion; in CloudWatch watch `ServerlessDatabaseCapacity` scale 0.5 → 8 ACU.

## Step 4: RDS Proxy
> **Why:** Postgres connections cost ~10 MB RAM each. 1000 Lambda concurrent invocations × direct connections = DB OOM. RDS Proxy pools and multiplexes connections. Apps connect to proxy; proxy maintains pool to DB.

```typescript
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';

const proxy = new rds.DatabaseProxy(this, 'Proxy', {
  proxyTarget: rds.ProxyTarget.fromCluster(this.cluster),
  secrets: [this.cluster.secret!],
  vpc: props.vpc,
  iamAuth: true,
  requireTLS: true,
});

new cdk.CfnOutput(this, 'ProxyEndpoint', { value: proxy.endpoint });
```

Point your Lambda env var `DB_HOST` at the proxy endpoint. Simulate 1000 concurrent Lambdas; Aurora's `numbackends` stays flat (~20) while proxy handles thousands of client connections.

## Step 5: Performance Insights
> **Why:** PI is automatic, always-on profiling by wait event. "Show me the top SQL burning CPU" becomes a single dashboard view. Free tier = 7 days retention. Essential for diagnosing slow queries without installing pg_stat_statements manually.

Run a slow query:
```sql
SELECT pg_sleep(10), generate_series(1,1000000);
```

Console → RDS → Performance Insights → your cluster. Top-SQL view shows the offending query with average active sessions waiting on CPU.

## Step 6: Cross-region read replica
> **Why:** DR and geographic reads. Aurora Global Database replicates to secondary region with <1 sec lag. Promote for DR (RTO ~1 min) or serve reads locally from the secondary.

```typescript
const globalCluster = new rds.CfnGlobalCluster(this, 'Global', {
  globalClusterIdentifier: 'app-global',
  sourceDbClusterIdentifier: this.cluster.clusterArn,
});
// Then in a stack deployed to eu-west-1:
// new rds.DatabaseCluster(..., { globalClusterIdentifier: 'app-global', ... })
```

## Step 7: Blue/Green deployment
> **Why:** Blue/Green clones your cluster, lets you upgrade/modify the green copy (schema changes, major version upgrade), then atomic DNS cutover. Rollback = flip back. The only safe way to do major version upgrades on prod.

```bash
aws rds create-blue-green-deployment \
  --blue-green-deployment-name upg-pg16 \
  --source $CLUSTER_ARN \
  --target-engine-version 16.3
```

Wait for `AVAILABLE`, test green, then:
```bash
aws rds switchover-blue-green-deployment --blue-green-deployment-identifier <id>
```

## Cleanup
```bash
aws rds delete-blue-green-deployment --blue-green-deployment-identifier <id> --delete-target
cdk destroy    # ~15 min
```

## Common Errors
- **Aurora storage bill keeps growing** — Aurora storage never shrinks; only snapshot + restore into new cluster reclaims. Plan accordingly.
- **Proxy: `IAM authentication failed`** — proxy requires `iamAuth: true` AND the IAM policy `rds-db:connect` attached to caller.
- **Serverless v2 idle bill high** — min 0.5 ACU ≈ $43/month floor. For true scale-to-zero, use Aurora DSQL or pause manually.
- **Blue/Green switchover timeout** — replication lag too high; wait for it to catch up or increase timeout.
- **Failover cancels in-flight transactions** — expected; apps need retry logic.
