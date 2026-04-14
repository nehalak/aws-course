# 03 — Auto Scaling Groups

## Concept
ASG maintains N healthy instances. Scaling policies react to metrics (CPU, SQS depth, custom).

## Exercises
1. **ASG with launch template**: CDK ASG, min=2 max=6 desired=2, behind ALB. Instances run stress tool on boot.
2. **Target tracking**: scale on average CPU 50%. `stress --cpu 4` on one instance → ASG scales out.
3. **Step scaling**: define `+1 if 60-80%`, `+3 if >80%`. Force high CPU; observe.
4. **Scheduled scaling**: min=5 every weekday 9am, min=1 nights.
5. **Lifecycle hooks**: add hook on terminate → Lambda drains connections before kill.
6. **Instance refresh**: change launch template AMI, trigger refresh with 50% healthy percentage.

## Verification
- Console shows scaling activity log.
- Lifecycle hook delays termination by the grace period.

## Gotchas
- ASG health check type: EC2 vs ELB (ELB catches app failures, not just OS).
- Replacement during refresh = downtime if min < desired.
- Spot interruptions: use Capacity Rebalancing.

## Cleanup
```bash
cdk destroy
```
