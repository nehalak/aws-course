# 03 — Hybrid Networking

## Concept
Direct Connect = dedicated line to AWS. Site-to-Site VPN = IPsec over internet. Cloud WAN = SD-WAN overlay across regions.

## Exercises
1. **Site-to-Site VPN to "on-prem"**: simulate on-prem as a second VPC with strongSwan on EC2. Establish VPN.
2. **BGP routing**: configure BGP ASNs both sides; observe routes learned.
3. **Transit Gateway + VPN**: attach VPN to TGW; route multiple VPCs through.
4. **Cloud WAN**: create global network + core network with 2 regions. Attach VPCs via core-network policy.
5. **DX simulation**: read DX docs; architect a redundant DX with VPN backup; submit as `decision.md`.
6. **Private NAT + overlapping CIDRs**: use Private NAT GW to resolve CIDR overlap cases.

## Verification
- On-prem EC2 can ping AWS EC2 across VPN.
- BGP advertisements visible in TGW route table.

## Gotchas
- DX is weeks-to-provision + ~$250/mo per connection (real, not this exercise).
- VPN throughput capped ~1.25 Gbps per tunnel.
- Cloud WAN pricing is complex.

## Cleanup
```bash
cdk destroy
```
