# Walkthrough — 01 RDS Basics

## About this service
**RDS (Relational Database Service)** runs managed relational DBs: Postgres, MySQL, MariaDB, Oracle, SQL Server. AWS handles patching, backups, Multi-AZ failover, monitoring. You still own schema, queries, performance tuning.

**Why it matters:** databases are the #1 thing people get wrong on cloud (self-managed). RDS removes most operational risk. Aurora (AWS's own Postgres/MySQL-compatible engine) goes further — separate storage and compute.

**When to use RDS:** any relational workload where you don't want to be a DBA. Apps using SQL.
**When NOT to use RDS:** NoSQL needs (DynamoDB), analytics at scale (Redshift), TB+ datasets where self-managed on EC2 is cheaper, apps needing DB features RDS doesn't support (custom extensions not on the allowlist).

## Estimated cost
- **`db.t3.micro` Postgres: ~$12/month** on-demand + $0.115/GB/mo storage (gp3)
- **Multi-AZ: 2x** (runs standby instance)
- **Backups: free up to DB size** (additional charged $0.095/GB/mo)
- **Snapshots: $0.095/GB/mo** retained beyond DB
- **IOPS on gp3 above 3000 baseline:** extra
- Total for this lesson: **~$14/month** running. Destroy after!

---

## Step 1: Stack
> **Why:** This config is close to production minimum: private subnet, generated password, IAM auth enabled. `deletionProtection: false` is ONLY for learning — production should be `true`.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';

export class RdsStack extends cdk.Stack {
  constructor(scope: any, id: string, props: cdk.StackProps & { vpc: ec2.IVpc }) {
    super(scope, id, props);
    const { vpc } = props;

    const db = new rds.DatabaseInstance(this, 'Pg', {
      engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_16_2 }),
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      allocatedStorage: 20,
      storageType: rds.StorageType.GP3,
      credentials: rds.Credentials.fromGeneratedSecret('postgres'),
      databaseName: 'appdb',
      multiAz: false,
      publiclyAccessible: false,
      iamAuthentication: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      deletionProtection: false,
      backupRetention: cdk.Duration.days(1),
    });

    new cdk.CfnOutput(this, 'Endpoint', { value: db.instanceEndpoint.hostname });
    new cdk.CfnOutput(this, 'SecretArn', { value: db.secret!.secretArn });
  }
}
```

## Step 2: Bastion
> **Why:** RDS in private subnet = inaccessible from your laptop. Bastion EC2 in public subnet = jump box. SSH to it, then psql from there. Production uses Session Manager + RDS Proxy instead.

```typescript
const bastionSg = new ec2.SecurityGroup(this, 'BastionSg', { vpc });
bastionSg.addIngressRule(ec2.Peer.ipv4('YOUR.IP/32'), ec2.Port.tcp(22));
db.connections.allowDefaultPortFrom(bastionSg);
```

Deploy takes ~10 min for RDS.

## Step 3: Connect
> **Why:** Fetching the password from Secrets Manager (instead of typing it) is the pattern you'll use in apps. Tests the full chain: RDS creates password → Secrets Manager stores → you retrieve → psql uses.

```bash
SECRET=$(aws secretsmanager get-secret-value --secret-id <arn> --query SecretString --output text)
PASS=$(echo $SECRET | jq -r .password)
HOST=$(echo $SECRET | jq -r .host)
PGPASSWORD=$PASS psql -h $HOST -U postgres -d appdb
```

```sql
CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO users (name) VALUES ('alice'), ('bob');
SELECT * FROM users;
```

## Step 4: Snapshot & restore
> **Why:** Snapshots = DR insurance. Practicing restore NOW builds confidence you can recover. Teams that never test backups often find they can't restore when disaster strikes.

```bash
aws rds create-db-snapshot --db-instance-identifier $DB_ID --db-snapshot-identifier learn-snap
aws rds wait db-snapshot-completed --db-snapshot-identifier learn-snap
```

Drop table, then:
```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier restored-pg \
  --db-snapshot-identifier learn-snap \
  --db-instance-class db.t3.micro
```

## Step 5: Multi-AZ failover
> **Why:** Multi-AZ = sync standby in a different AZ. On failover, DNS flips to standby. Downtime 30-60s (not 0). Critical to test if HA matters.

Change `multiAz: true`, redeploy, then:
```bash
aws rds reboot-db-instance --db-instance-identifier $DB_ID --force-failover
```

## Step 6: IAM DB auth
> **Why:** Passwordless connections for apps. Tokens valid 15 min. Eliminates password rotation, auditability via CloudTrail.

```sql
CREATE USER iamuser;
GRANT rds_iam TO iamuser;
```

```bash
TOKEN=$(aws rds generate-db-auth-token --hostname $HOST --port 5432 --username iamuser)
PGPASSWORD=$TOKEN psql "host=$HOST user=iamuser dbname=appdb sslmode=require"
```

## Cleanup
```bash
aws rds delete-db-instance --db-instance-identifier restored-pg --skip-final-snapshot
aws rds delete-db-snapshot --db-snapshot-identifier learn-snap
cdk destroy    # ~10 min
```

## Common Errors
- **`psql: could not connect`** — bastion SG allows your IP? DB SG allows bastion SG? In private subnet?
- **`password authentication failed`** — rotated password; re-fetch.
- **Snapshot restore different endpoint** — yes, always; update app config.
