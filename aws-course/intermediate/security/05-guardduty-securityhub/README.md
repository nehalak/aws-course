# 05 — GuardDuty & Security Hub

## Concept
GuardDuty = threat detection (ML on CloudTrail/VPC/DNS). Security Hub = aggregator + CIS/PCI compliance checks.

## Exercises
1. **Enable GuardDuty** in all regions via Organizations (or single region if standalone).
2. **Generate sample findings**: GuardDuty console → "Generate sample findings". Observe 40+ finding types.
3. **Enable Security Hub** with AWS Foundational + CIS v1.4.0 standards.
4. **Fix 3 failing controls**: e.g., enable S3 public access block account-wide, enable default EBS encryption, rotate IAM access keys.
5. **Auto-remediation**: EventBridge rule on GuardDuty finding `UnauthorizedAccess:EC2/SSHBruteForce` → Lambda that updates SG to block source IP.
6. **Suppression rules**: suppress recurring finding for a test instance.

## Verification
- Sample findings appear in both GD and SH.
- Lambda fires when synthetic finding matches pattern.

## Gotchas
- GuardDuty cost scales with CloudTrail + VPC Flow Logs volume.
- Security Hub controls cost $0.0010 per check.
- Disable when experimenting to avoid bills.

## Cleanup
Disable both services in console (keeps findings 90 days).
