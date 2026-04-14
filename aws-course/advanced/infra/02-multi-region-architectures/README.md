# 02 — Multi-Region Architectures

## Concept
Active-active vs active-passive. Data replication, DNS failover, eventual consistency trade-offs.

## Exercises
1. **Route 53 failover**: primary us-east-1, secondary eu-west-1 each with ALB. Health check on primary; stop it; DNS flips within TTL.
2. **DynamoDB Global Tables**: 2-region; write to each; observe replication lag (<1 sec typical).
3. **Aurora Global Database**: primary writer us-east-1; reader replica eu-west-1. Promote DR region.
4. **S3 CRR + Multi-Region Access Point**: clients hit MRAP, routed to nearest replica.
5. **Latency-based Route 53**: routes users to closest healthy region.
6. **Chaos test**: simulate region outage by breaking health check; measure RTO.
7. **RPO/RTO doc**: write `dr-plan.md` with your metrics.

## Verification
- Global table writes converge both regions.
- Failover within expected RTO.

## Gotchas
- DynamoDB Global Tables require streams + same schema.
- Aurora Global promote is manual + one-way without restart.
- Cross-region data transfer = $$$.

## Cleanup
```bash
cdk destroy  # in both regions
```
