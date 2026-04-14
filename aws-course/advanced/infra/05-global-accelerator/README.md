# 05 — Global Accelerator

## Concept
Static anycast IPs at AWS edge. Routes TCP/UDP to nearest healthy endpoint. Deterministic failover vs CloudFront (cacheable).

## Exercises
1. **GA with 2 regional endpoints**: ALB us-east-1 + ALB eu-west-1. Traffic dials 50/50.
2. **Test routing**: `dig` the anycast IP from different regions. Same IP, different edge.
3. **Failover**: break us-east-1 health check; observe traffic shifts entirely to eu.
4. **Client affinity**: set `SOURCE_IP` so repeat clients hit same endpoint.
5. **Custom routing**: map specific clients to specific endpoints (gaming scenarios).
6. **Compare to CloudFront**: write `ga-vs-cf.md` — TCP vs HTTP, cacheable vs pass-through, pricing.

## Verification
- Same IP resolves everywhere; traffic hits nearest region.
- Failover within seconds (not DNS TTL-bound).

## Gotchas
- GA = $18/mo fixed + data transfer.
- No caching — not a CDN replacement.

## Cleanup
```bash
cdk destroy
```
