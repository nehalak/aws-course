# Walkthrough — 03 ElastiCache

## About this service
**ElastiCache** is managed Redis / Memcached / Valkey (the open-source Redis fork AWS now pushes). It's an in-memory data store serving sub-millisecond reads. Used for caching DB results, session storage, rate limiting, pub/sub, leaderboards, and real-time analytics.

**Why it matters:** adding a cache is often the cheapest way to 10x an app's throughput. A RDS query at 5 ms becomes a Redis `GET` at 0.3 ms. Sessions in Redis survive app restarts and scale across horizontally-scaled servers.

**When to use:** read-heavy workloads, session storage, leaderboards (sorted sets), pub/sub, rate limiting, queue/stream patterns.
**When NOT to use:** durable primary storage (Redis is in-memory — data loss on full failure possible without careful config), complex queries (use a database), tiny workloads where DynamoDB on-demand is cheaper.

## Estimated cost
- **`cache.t4g.micro` Valkey: ~$11/month** per node (us-east-1)
- **Cluster mode disabled (1 primary + 1 replica): ~$22/month**
- **Cluster mode enabled (3 shards × 2 nodes = 6 nodes): ~$66/month**
- **Data transfer: free within AZ**, $0.01/GB cross-AZ
- **Backups: first backup free**, $0.085/GB/month beyond
- Total for this lesson: **~$25/month** if left running. Destroy after!

---

## Step 1: Valkey replication group (cluster mode disabled)
> **Why:** Start simple — 1 primary + 1 replica in Multi-AZ. Automatic failover ~30 sec. Apps use the *primary endpoint* for writes, *reader endpoint* for reads. Valkey is the open-source Redis fork; AWS recommends it over Redis OSS as of 2024 (cheaper, same API).

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as cache from 'aws-cdk-lib/aws-elasticache';

export class ElastiCacheStack extends cdk.Stack {
  constructor(scope: any, id: string, props: cdk.StackProps & { vpc: ec2.IVpc }) {
    super(scope, id, props);
    const { vpc } = props;

    const sg = new ec2.SecurityGroup(this, 'CacheSg', { vpc, allowAllOutbound: true });

    const subnetGroup = new cache.CfnSubnetGroup(this, 'SubnetGroup', {
      description: 'cache subnets',
      subnetIds: vpc.privateSubnets.map(s => s.subnetId),
    });

    const rg = new cache.CfnReplicationGroup(this, 'Valkey', {
      replicationGroupDescription: 'app cache',
      engine: 'valkey',
      engineVersion: '7.2',
      cacheNodeType: 'cache.t4g.micro',
      numCacheClusters: 2,
      automaticFailoverEnabled: true,
      multiAzEnabled: true,
      cacheSubnetGroupName: subnetGroup.ref,
      securityGroupIds: [sg.securityGroupId],
      transitEncryptionEnabled: true,
      atRestEncryptionEnabled: true,
      port: 6379,
    });
    rg.addDependency(subnetGroup);

    new cdk.CfnOutput(this, 'PrimaryEp', { value: rg.attrPrimaryEndPointAddress });
    new cdk.CfnOutput(this, 'ReaderEp', { value: rg.attrReaderEndPointAddress });
  }
}
```

## Step 2: Connect from a bastion
> **Why:** ElastiCache has no public endpoint. From an EC2 bastion in the same VPC (with SG allowing port 6379 → cache SG): `redis-cli` with TLS flag because transit encryption is on.

```bash
sudo yum install -y redis6
redis-cli -h $PRIMARY_EP --tls -p 6379
```

```
127.0.0.1:6379> SET user:1 '{"name":"alice"}'
OK
127.0.0.1:6379> GET user:1
"{\"name\":\"alice\"}"
127.0.0.1:6379> EXPIRE user:1 60
(integer) 1
```

## Step 3: Lazy-load cache pattern
> **Why:** The most common cache pattern. App checks Redis → miss → query source DB → `SET` in Redis with TTL → return. Next call is a hit. Simple, resilient, and cache failures degrade (not break) the app.

```python
import redis, psycopg
r = redis.Redis(host=EP, ssl=True, decode_responses=True)
def get_user(uid):
    cached = r.get(f"user:{uid}")
    if cached: return json.loads(cached)
    with psycopg.connect(PG_DSN) as conn:
        row = conn.execute("SELECT name FROM users WHERE id=%s", (uid,)).fetchone()
    r.setex(f"user:{uid}", 300, json.dumps(row))   # 5-min TTL
    return row
```

Expected: first call ~8 ms (DB), second ~0.5 ms (cache).

## Step 4: Write-through pattern
> **Why:** On every write, update BOTH the DB and cache. Reads never miss. Cost: double-write latency; risk: if one succeeds and the other fails, they diverge. Use when staleness is unacceptable.

```python
def update_user(uid, name):
    with psycopg.connect(PG_DSN) as conn:
        conn.execute("UPDATE users SET name=%s WHERE id=%s", (name, uid))
    r.setex(f"user:{uid}", 300, json.dumps({"name": name}))
```

## Step 5: Cluster mode enabled
> **Why:** Cluster mode shards the keyspace across multiple primaries — horizontal scaling beyond one node's RAM. Keys are hashed to slots, slots live on shards. Requires a *cluster-aware client* (`redis-py-cluster`, Lettuce cluster mode). Single endpoint = the cluster configuration endpoint.

```typescript
const shardedRg = new cache.CfnReplicationGroup(this, 'ValkeyCluster', {
  replicationGroupDescription: 'sharded',
  engine: 'valkey',
  cacheNodeType: 'cache.t4g.micro',
  numNodeGroups: 3,           // 3 shards
  replicasPerNodeGroup: 2,    // 2 replicas per shard
  automaticFailoverEnabled: true,
  multiAzEnabled: true,
  cacheSubnetGroupName: subnetGroup.ref,
  securityGroupIds: [sg.securityGroupId],
  transitEncryptionEnabled: true,
});
```

Test: `redis-cli --cluster info $CONFIG_EP:6379` shows 3 masters, 6 slaves.

## Step 6: Pub/Sub
> **Why:** Redis pub/sub is at-most-once, fire-and-forget messaging. Great for real-time fanout (chat, live dashboards). NOT a queue — if no subscriber online, message lost. Cluster-mode note: pub/sub is cluster-wide in Valkey 7+.

Terminal 1 and 2:
```
redis-cli -h $EP --tls SUBSCRIBE chat
```

Terminal 3:
```
redis-cli -h $EP --tls PUBLISH chat "hello"
(integer) 2     # 2 subscribers received
```

## Step 7: Session store with TTL
> **Why:** JWT refresh tokens need server-side revocation. Store them in Redis with TTL = token expiry. `DEL` on logout revokes instantly; expired tokens auto-evicted — no cron job.

```python
r.setex(f"refresh:{user_id}", 7*24*3600, refresh_token)
# On logout
r.delete(f"refresh:{user_id}")
# On every API call
if r.get(f"refresh:{user_id}") != presented_token: raise Unauthorized()
```

## Cleanup
```bash
cdk destroy    # ~10 min
```

## Common Errors
- **`Connection refused`** — security group doesn't allow port 6379 from bastion SG.
- **`Error: SSL connection error`** — `transitEncryptionEnabled: true` means client MUST use TLS (`--tls` flag / `ssl=True`).
- **`MOVED 1234 10.0.2.5:6379`** — using a non-cluster client against cluster mode. Switch to `redis-py-cluster` or Lettuce cluster.
- **`READONLY You can't write against a read only replica`** — writing to the reader endpoint. Use primary endpoint.
- **Pub/Sub messages missing** — subscriber disconnected momentarily. Redis pub/sub has no replay; use Streams (`XADD`/`XREAD`) for durability.
