# 01 — RDS Basics

## Concept
Managed relational databases. Postgres/MySQL/MariaDB/Oracle/SQL Server. Handles patching, backups, failover.

## Exercises
1. **CDK RDS Postgres** `db.t3.micro`, 20GB gp3, in private subnets of the VPC. Password from Secrets Manager (auto-generated).
2. **Connect from bastion**: launch a small EC2 in public subnet, SG allows 5432 to RDS. `psql -h <endpoint> -U postgres`. Create a table, insert rows.
3. **Snapshot & restore**: take a manual snapshot. Delete a table. Restore snapshot to a new instance. Verify table returns.
4. **Multi-AZ**: toggle Multi-AZ on, force failover (`reboot-db-instance --force-failover`). Observe ~30-60 sec downtime.
5. **Parameter group**: create custom parameter group, change `log_statement=all`. Enable RDS log export to CloudWatch.
6. **IAM auth**: enable IAM DB auth, create DB user mapped to IAM. Connect using token: `aws rds generate-db-auth-token`.

## Verification
- `psql \dt` shows your table.
- Failover causes brief connection drop, endpoint stays same.

## Gotchas
- `db.t3.micro` = free tier 750hrs, but storage + IOPS cost extra.
- Public RDS = bad idea. Keep in private subnets.
- Snapshots are kept after delete only if final snapshot created.

## Cleanup
```bash
cdk destroy  # takes ~10 min
# delete any manual snapshots
```
