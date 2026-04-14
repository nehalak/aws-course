# 03 — KMS Basics

## Concept
Managed encryption keys. CMK (Customer Master Key) never leaves KMS. Envelope encryption: KMS encrypts a data key, data key encrypts data.

## Exercises
1. **CDK a CMK** with rotation enabled and a key policy granting your user `kms:*`.
2. **Encrypt a string** via CLI:
   ```bash
   aws kms encrypt --key-id <id> --plaintext "hello" --query CiphertextBlob --output text | base64 -d > enc.bin
   aws kms decrypt --ciphertext-blob fileb://enc.bin --query Plaintext --output text | base64 -d
   ```
3. **S3 with SSE-KMS**: create a bucket using your CMK. Upload object. Deny yourself `kms:Decrypt` — observe `GetObject` fails despite S3 permissions.
4. **Key policy vs IAM**: KMS requires BOTH to allow. Remove key policy → even admin can't use. (Be careful — only root can fix.)
5. **Grants**: create a grant allowing a Lambda role to decrypt. Revoke grant.
6. **Aliases**: create alias `alias/app-data`. Use in CDK as `kms.Alias.fromAliasName`.

## Verification
- Encrypted ciphertext ≠ plaintext.
- S3 GetObject fails when KMS decrypt denied.

## Gotchas
- Delete a CMK = 7-30 day waiting period (can cancel).
- Key policy lockout = contact AWS support.
- KMS costs: $1/key/mo + $0.03/10k requests.

## Cleanup
```bash
cdk destroy  # schedules key deletion in 7 days by default
```
