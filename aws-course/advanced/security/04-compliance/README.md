# 04 — Compliance

## Concept
Artifact = compliance report downloads. Audit Manager = evidence collection. FedRAMP/HIPAA/PCI = AWS + your shared responsibility.

## Exercises
1. **Artifact**: download SOC 2 report. Skim responsibilities matrix.
2. **Audit Manager**: enable; add framework (PCI DSS). Run an assessment; see auto-collected evidence.
3. **HIPAA architecture**: design (paper) a HIPAA-eligible architecture: encrypt at rest (KMS) + in transit (TLS), dedicated tenancy, CloudTrail + Config, no non-eligible services.
4. **PCI DSS**: list the "AWS scope reduction" patterns (tokenization, Stripe/outsourced vaulting).
5. **GDPR data residency**: tag PII, enforce replication only to EU regions via SCP.
6. **AWS Config conformance pack** for PCI; remediate 3 failing controls.

## Verification
- Audit Manager assessment report PDF generated.
- Conformance pack compliance > 80%.

## Gotchas
- "HIPAA eligible" ≠ HIPAA compliant — you still own architecture.
- BAA required with AWS before storing PHI.

## Cleanup
Disable Audit Manager to stop costs.
