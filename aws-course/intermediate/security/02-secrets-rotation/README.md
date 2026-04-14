# 02 — Secrets Rotation

## Concept
Secrets Manager auto-rotates creds via Lambda rotator. Built-in templates for RDS/Redshift/DocDB.

## Exercises
1. **RDS rotation**: attach Secrets Manager-generated password to RDS Postgres (from basic/data/01). Enable 30-day rotation.
2. **Force rotation**: `aws secretsmanager rotate-secret`. Observe new password; old stays valid briefly (AWSPREVIOUS).
3. **Custom rotator**: write a Lambda that rotates a third-party API key. Implement the 4-step state machine (createSecret → setSecret → testSecret → finishSecret).
4. **Staging labels**: understand `AWSCURRENT` vs `AWSPENDING` vs `AWSPREVIOUS`. Tag-based retrieval.
5. **Cross-account secret**: share a secret via resource policy to another account.

## Verification
- RDS still accessible after rotation.
- CloudWatch Logs show rotator Lambda success.

## Gotchas
- App must handle new password transparently — pull from Secrets Manager, don't cache forever.
- Rotation inside a private subnet needs VPC endpoint for Secrets Manager.

## Cleanup
```bash
cdk destroy
```
