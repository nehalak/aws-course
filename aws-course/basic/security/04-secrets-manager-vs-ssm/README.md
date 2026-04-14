# 04 — Secrets Manager vs SSM Parameter Store

## Concept
Both store secrets. Secrets Manager = rotation, cross-region replication, $0.40/secret/mo. SSM Parameter Store = free (standard), no rotation, max 4KB standard.

## Exercises
1. **SSM Parameter**: CDK a `StringParameter` with name `/myapp/dev/api-url`. Read via CLI: `aws ssm get-parameter --name /myapp/dev/api-url`.
2. **Secure string**: create `/myapp/dev/db-password` as `SecureString` (KMS-encrypted). Read with `--with-decryption`.
3. **Secrets Manager**: CDK a secret with a generated password (32 chars, no `/@" `). Retrieve via CLI.
4. **Rotate a secret**: attach a Lambda rotator (Secrets Manager template for RDS) — we'll wire to RDS in `data/01`.
5. **Compare costs**: pretend you have 50 app configs + 10 DB passwords. Which goes where and why? Write `decision.md`.
6. **Reference in CDK**: use `secretsmanager.Secret.fromSecretNameV2` in a Lambda env var (as a reference, not resolved at synth).

## Verification
- `aws secretsmanager get-secret-value` returns the JSON.
- CloudFormation template shows `{{resolve:secretsmanager:...}}` not the literal value.

## Gotchas
- Never pass secret values to `environment` plaintext.
- SSM `StringParameter` vs `SecureString` — mixing up types = config drift.
- Secret recovery window: 7–30 days before permanent delete.

## Cleanup
```bash
cdk destroy
aws secretsmanager delete-secret --secret-id <id> --force-delete-without-recovery
```
