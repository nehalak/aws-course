# 02 — Encryption Deep

## Concept
KMS key policies, grants, XKS (external key store), CloudHSM (dedicated), Nitro Enclaves (confidential compute).

## Exercises
1. **Key policy patterns**: write policy that (a) restricts to a VPC endpoint via `kms:ViaService` (b) requires MFA via `aws:MultiFactorAuthPresent`.
2. **Grants**: programmatically create grant for a Lambda role, decrypt only; revoke; verify.
3. **Multi-Region keys**: create primary in us-east-1; replicate to eu-west-1; encrypt in one, decrypt in other.
4. **CloudHSM cluster**: provision (expensive — skim docs + allocate 2h). Initialize. Integrate with KMS custom key store.
5. **XKS**: review architecture (external HSM over HTTPS). Don't deploy unless you have one.
6. **Nitro Enclave**: launch M5.metal or supported instance; run sample enclave app processing secret in isolation.

## Verification
- Multi-region key encrypts/decrypts across regions.
- Nitro Enclave cannot access network; only vsock.

## Gotchas
- CloudHSM = ~$1.45/hr per HSM × 2 for HA = $$$.
- XKS requires your HSM to meet 99.999% SLA.

## Cleanup
Delete CloudHSM cluster immediately.
