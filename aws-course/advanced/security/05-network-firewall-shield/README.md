# 05 — Network Firewall & Shield

## Concept
Network Firewall = stateful Suricata-based IDS/IPS at VPC. Shield Standard (free DDoS) + Advanced ($3k/mo).

## Exercises
1. **Network Firewall**: deploy with Suricata rules: deny traffic to malicious-domains.com, log all DNS.
2. **Stateless rules**: drop non-RFC1918 ingress on specific subnets.
3. **Rule groups sharing** (Firewall Manager): central mgmt across Organization.
4. **Simulate DDoS**: `hey` tool hammering ALB. Watch CloudWatch `RequestCount` spike.
5. **WAF rate-limit rule** (alternative cheaper DDoS mitigation).
6. **Shield Advanced tour** in console: review what DRT (DDoS response team) access gets you. Don't enable unless in real production.

## Verification
- Network Firewall logs blocked connections.
- Rate-limit returns 429 under hammer.

## Gotchas
- NFW = $$$ per endpoint + GB. Expensive.
- Shield Advanced = 1-year commit.

## Cleanup
```bash
cdk destroy
```
