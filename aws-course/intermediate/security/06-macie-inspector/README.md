# 06 — Macie & Inspector

## Concept
Macie = discovers PII/PHI in S3. Inspector = vulnerability scans for EC2, ECR images, Lambda.

## Exercises
1. **Macie**: enable. Create classification job on a bucket. Upload a fake CSV with SSNs/emails. Observe findings.
2. **Custom data identifier**: regex for employee ID format. Test it.
3. **Inspector**: enable for EC2 + ECR + Lambda. Launch a vulnerable old Ubuntu AMI; watch findings populate.
4. **ECR scan**: push an image based on old node:14 base. Inspector lists CVEs.
5. **Export findings** to Security Hub.
6. **Remediate**: update image base; push; watch findings close.

## Verification
- Macie report lists PII matches.
- CVE count drops after image rebuild.

## Gotchas
- Macie pricing: $1/GB for sensitive data discovery. Sample first.
- Inspector for EC2 needs SSM agent.

## Cleanup
Disable services.
