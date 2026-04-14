# 03 — AWS Config

## Concept
Records resource configurations + evaluates against rules. Conformance packs = pre-built rule sets. Auto-remediation via SSM docs.

## Exercises
1. **Enable Config** in region. Select all resource types. Delivery to S3.
2. **Managed rules**: attach `s3-bucket-public-read-prohibited`, `encrypted-volumes`, `iam-password-policy`.
3. **Custom rule** (Lambda): flag EC2 instances without `Environment` tag.
4. **Conformance pack**: deploy `Operational-Best-Practices-for-PCI-DSS`. Review compliance %.
5. **Auto-remediation**: on `s3-bucket-public-read-prohibited` failure → SSM doc re-applies block-public-access.
6. **Timeline**: pick a resource in Config; view configuration history diffs.

## Verification
- Non-compliant resources show in dashboard.
- Remediation runs and flips resource to compliant.

## Gotchas
- Config is per-region — enable everywhere or you miss things.
- Costs: $0.003 per configuration item + rule evaluations.

## Cleanup
```bash
cdk destroy
# disable recorder to stop costs
```
