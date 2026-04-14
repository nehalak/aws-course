# 05 — Route 53 Basics

## Concept
DNS. Hosted zones (public/private), record types (A, AAAA, CNAME, ALIAS, MX, TXT), TTL, health checks.

## Exercises
1. **Buy or use a domain** (or use a free subdomain like `.sslip.io`). If you own one, create a hosted zone in Route 53 and update registrar NS records.
2. **CDK records**: create an A record for `test.<domain>` pointing to an EC2 public IP.
3. **ALIAS vs CNAME**: create an ALIAS record pointing to an ALB (from a future lesson) at the apex (`domain.com`). Note you can't CNAME an apex.
4. **Health check**: create one that pings `/health` on your EC2. Shut down nginx → observe alarm.
5. **Private hosted zone**: create one for `internal.local`. Associate with VPC. Create `db.internal.local` → RDS endpoint (lesson `data/01`).
6. **Weighted routing**: two A records for `test.<domain>` with weights 70/30. Dig many times, count ratios.

## Verification
- `dig test.<domain>` resolves.
- Health check status changes when backend stops.

## Gotchas
- Hosted zone = $0.50/mo. Delete when done.
- DNS propagation isn't instant — TTL matters.
- NS records at registrar must match exactly.

## Cleanup
```bash
cdk destroy
# manually delete hosted zone
```
