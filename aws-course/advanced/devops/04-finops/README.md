# 04 — FinOps

## Concept
Cost Explorer + Budgets + Compute Optimizer + Savings Plans + tagging = cost visibility & control.

## Exercises
1. **Tagging strategy**: require tags `CostCenter`, `Environment`, `Owner`, `Application`. Enforce via SCP.
2. **Cost Explorer reports**: top 10 services by cost last 30 days. Group by tag `CostCenter`.
3. **Anomaly detection**: enable Cost Anomaly Detection; monitor by service + tag.
4. **Budgets**: $100/mo overall + per-app budget via tag; 80% alert.
5. **Compute Optimizer**: enable; review recommendations. Resize 1 instance per rec; measure savings.
6. **Savings Plans**: analyze Compute SP vs EC2 SP vs Reserved Instances for your running Fargate + Lambda. Simulate with SP recommendations tool.
7. **S3 Intelligent-Tiering rollout**: apply to all buckets via lifecycle.
8. **Right-sizing Lambda**: use Compute Optimizer for Lambda recommendations.

## Verification
- Untagged resources = 0 after enforcement.
- Savings Plan utilization > 95% after commitment.

## Gotchas
- Budget email lag: can be 24h.
- SP commitments are 1- or 3-year — not reversible.
- Reserved capacity for RDS/ElastiCache separate from SP.

## Cleanup
Don't buy SPs for learning — review only.
