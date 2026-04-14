# 03 — Disaster Recovery

## Concept
RPO (max data loss) + RTO (max downtime). Strategies: Backup/Restore → Pilot Light → Warm Standby → Active/Active. AWS Backup centralizes. FIS = chaos engineering.

## Exercises
1. **AWS Backup plan**: backup RDS + EBS + DynamoDB + EFS daily, 30-day retention, cross-region copy.
2. **Restore test**: restore RDS from backup to alt region; measure RTO.
3. **Pilot Light**: Aurora Global secondary + scaled-down stacks in DR region. Promote; measure.
4. **Warm Standby**: full stack scaled-down in DR; scale up on failover.
5. **FIS experiments**:
   - Stop 50% of ASG instances; verify ALB health.
   - Inject network latency to RDS.
   - API throttling on DynamoDB.
6. **Chaos day**: induce AZ failure (pause AZ subnet's NAT); validate auto-recovery.
7. **Runbook doc**: write `dr-runbook.md` — step-by-step failover procedure.

## Verification
- Restore drill completes under RTO.
- FIS experiments trigger expected alarms.

## Gotchas
- Backup vault `Recovery Point` prices = storage + duration.
- Cross-region copy doubles storage cost.
- FIS has blast-radius controls — use them.

## Cleanup
```bash
cdk destroy
# delete Backup vaults after recovery points aged out
```
