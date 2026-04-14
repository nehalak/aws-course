# 03 — EC2 Deep

## Concept
Nitro system, placement groups, spot strategies, dedicated hosts, burstable CPU credits.

## Exercises
1. **Nitro**: launch a Nitro instance (m5+) and a non-Nitro (m4). Compare boot time + EBS throughput.
2. **Placement group cluster**: 3 instances in cluster PG; iperf3 between them vs across AZs. Observe 10 Gbps vs 1 Gbps.
3. **Spread PG**: 7 instances max across hardware for HA.
4. **Spot fleet**: request 10 spot across 5 instance types with `capacity-optimized`. Simulate interruption with `aws ec2 send-spot-instance-interruption`.
5. **Spot interruption handler**: metadata endpoint `/spot/instance-action` — polling script drains connection.
6. **Dedicated host**: create one for BYOL / compliance (don't keep running).
7. **Burstable credits**: on `t3.small`, exhaust CPU credits; observe throttling; switch to `t3.unlimited`.

## Verification
- Cluster PG iperf shows 10 Gbps.
- Interruption handler runs before termination.

## Gotchas
- Dedicated host = per-host billing, expensive.
- `t3.unlimited` can exceed burst = surprise bill.
- Spot Fleet deprecated name now Spot Requests + EC2 Fleet.

## Cleanup
```bash
cdk destroy
```
