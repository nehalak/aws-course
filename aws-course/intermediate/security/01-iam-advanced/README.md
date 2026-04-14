# 01 — IAM Advanced

## Concept
Permission boundaries cap max perms. SCPs cap orgs/OUs. ABAC = tag-based perms. Condition keys = fine-grained control.

## Exercises
1. **Permission boundary**: create boundary allowing only `s3:*` + `logs:*`. Create a user with `AdministratorAccess` identity policy but boundary attached. Verify DynamoDB calls fail.
2. **ABAC**: tag users with `Team=alpha`. Policy grants `s3:*` on buckets where `${aws:ResourceTag/Team} = ${aws:PrincipalTag/Team}`.
3. **Condition keys**: deny all S3 unless `aws:SourceIp` is office CIDR. Then allow with `aws:MultiFactorAuthPresent`.
4. **SCP**: in Organizations, create SCP denying `ec2:*` in all regions except `us-east-1`. Attach to an OU.
5. **Role chaining**: user assumes RoleA which assumes RoleB. Observe session tags/duration limits.
6. **Access Analyzer**: enable; create a bucket shared cross-account; observe finding.

## Verification
- Boundary user fails DynamoDB despite `Admin` policy.
- ABAC user accesses only `Team=alpha` buckets.

## Gotchas
- Boundary limits max, doesn't grant.
- SCP deny overrides identity allow at org level.
- Role chain max 1 hop explicit, session 1hr when chained.

## Cleanup
```bash
cdk destroy
# detach SCP and delete
```
