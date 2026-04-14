# 04 — S3 Advanced

## Concept
Lifecycle, replication (CRR/SRR), Object Lambda, Access Points, Multi-region Access Points, S3 Select.

## Exercises
1. **Lifecycle rules**: STANDARD → IA 30d → GLACIER 90d → delete 365d. Upload a file; inspect transitions next day.
2. **Cross-region replication (CRR)**: bucket A us-east-1 → B eu-west-1. Upload to A; see in B within seconds.
3. **Replication Time Control (RTC)**: 15-min SLA, extra cost. Enable & test.
4. **Object Lambda**: transform objects on GET — e.g., redact PII from CSV on-the-fly.
5. **Access Points**: separate access policies per team for same bucket.
6. **S3 Select**: query CSV/JSON/Parquet in-place with SQL — no extract needed.
7. **Multi-Region Access Point**: single endpoint routes to closest replica.

## Verification
- CRR destination has same object within 15 min.
- Object Lambda returns transformed content while original untouched.

## Gotchas
- Replication doesn't copy existing objects (need Batch Replication).
- Object Lambda = Lambda cost per GET.
- Multi-region AP requires replication already configured.

## Cleanup
```bash
cdk destroy
```
