# 01 — Multi-Account Landing Zone

## Concept
Organizations + Control Tower + AFT (Account Factory for Terraform). Accounts for isolation, billing, blast radius.

## Exercises
1. **Set up Organizations**: create OUs — `Security`, `Workloads/Dev`, `Workloads/Prod`, `Sandbox`.
2. **Enable Control Tower**: review mandatory guardrails. Note log archive + audit accounts.
3. **Create 3 accounts** via Account Factory for learning (dev, staging, prod).
4. **SCPs**: deny root user actions, deny leaving org, deny specific regions, require MFA for destructive actions. Attach per-OU.
5. **Delegated admin**: delegate Security Hub, GuardDuty to the audit account.
6. **CDK cross-account deploy**: bootstrap all 3 accounts with `--trust <tooling-account>`. Pipeline deploys to all 3.
7. **Central logging**: aggregate CloudTrail + Config + VPC Flow Logs to log-archive account bucket.

## Verification
- SCP blocks `us-west-2` if policy says so.
- Central bucket receives logs from all accounts.

## Gotchas
- Control Tower setup is 1-way — undo is painful.
- Closing accounts has 90-day suspension.
- Free tier applies per account.

## Cleanup
Close non-essential accounts. Keep management + security.
