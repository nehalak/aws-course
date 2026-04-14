# 04 — PrivateLink

## Concept
Expose a service (via NLB/GWLB) to other VPCs/accounts without peering. Traffic stays on AWS backbone.

## Exercises
1. **Provider side**: NLB in front of a Fargate service. Create VPC Endpoint Service pointing at NLB.
2. **Consumer side**: different VPC creates Interface VPC Endpoint targeting your service name.
3. **Whitelist consumer accounts** in endpoint service permissions.
4. **Test**: from consumer VPC EC2, `curl <endpoint-dns>` → reaches provider service privately.
5. **Cross-account**: repeat but provider in account A, consumer in account B.
6. **PrivateLink for SaaS**: architect exposing your SaaS to 50 customers without per-customer VPCs.

## Verification
- `traceroute` from consumer doesn't leave AWS.
- Service works without internet or peering.

## Gotchas
- Endpoint service requires NLB or GWLB.
- DNS must resolve in consumer — enable private DNS or use endpoint-specific DNS.
- Cross-account: consumers must accept; provider must whitelist.

## Cleanup
```bash
cdk destroy
```
