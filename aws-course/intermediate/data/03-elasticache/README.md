# 03 — ElastiCache

## Concept
Managed Redis / Memcached / Valkey. Caching, sessions, pub/sub, leaderboards.

## Exercises
1. **Redis cluster mode disabled**: 1 primary + 1 replica. Connect from EC2 via `redis-cli`.
2. **Lazy-load cache pattern**: Lambda → check Redis → miss → query RDS → SET in Redis.
3. **Write-through pattern**: app writes to both RDS and Redis.
4. **Cluster mode enabled**: 3 shards × 2 replicas. Test resharding.
5. **Pub/Sub**: 2 clients SUBSCRIBE, third PUBLISH. Observe.
6. **Valkey migration**: create Valkey cluster (open-source Redis fork); compare.
7. **Session store**: store JWT refresh tokens with TTL = expiry.

## Verification
- Cache hit rate > 90% after warm-up in your lazy-load test.
- Failover transparent to client using Redis cluster client.

## Gotchas
- Redis cluster mode needs cluster-aware client.
- TLS in-transit must be enabled at create time.
- Memory overhead ~25% for cluster bookkeeping.

## Cleanup
```bash
cdk destroy
```
