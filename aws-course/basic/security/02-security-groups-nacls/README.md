# 02 — Security Groups & NACLs

## Concept
SG = stateful instance firewall (default deny, allow rules). NACL = stateless subnet ACL (ordered allow + deny). Defense in depth.

## Exercises
1. **SG experiments**: launch EC2 with SG allowing only port 80 from `0.0.0.0/0`. Try SSH → blocked. Add port 22 from your IP → works.
2. **Stateful test**: from EC2, `curl https://google.com` works (outbound) even without inbound 443 rule. That's statefulness.
3. **SG referencing SG**: create SG-A (web) and SG-B (db). SG-B inbound allows port 5432 from SG-A only. Launch 2 EC2s, verify only web-tagged one connects.
4. **NACL**: create a NACL on private subnet. Deny port 22 from specific CIDR. Observe rule number ordering matters.
5. **Ephemeral ports**: NACL outbound must allow ephemeral return traffic (1024-65535). Break it on purpose; see things fail.

## Verification
- SG-B rejects traffic from an EC2 outside SG-A.
- NACL deny rule shows in VPC Flow Logs as REJECT.

## Gotchas
- SGs can reference other SGs by ID — same VPC only.
- NACLs are per-subnet; SGs are per-ENI.
- Default NACL allows all; custom NACL denies all.

## Cleanup
```bash
cdk destroy
```
