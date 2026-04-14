# 02 — Analytics Stack

## Concept
Athena (serverless SQL over S3), Redshift (columnar DW), EMR (Spark/Hadoop), QuickSight (BI).

## Exercises
1. **Athena federated query**: query both S3 data + RDS via Athena federation connector.
2. **Redshift Serverless**: create workgroup; `COPY` from S3 Parquet; run analytical queries.
3. **Redshift RA3 + managed storage**: deploy 2-node cluster; compare to Serverless for sustained workloads.
4. **Materialized views** in Redshift; auto-refresh.
5. **EMR Serverless**: submit Spark job reading S3 Parquet → aggregation → S3 output.
6. **QuickSight**: connect to Athena; build a dashboard with 3 viz + filter controls.
7. **Redshift Spectrum**: query S3 directly from Redshift as external schema.

## Verification
- QuickSight dashboard renders on live data.
- Spark job completes and writes results.

## Gotchas
- Redshift Serverless min 8 RPU = expensive for toy work.
- Athena cost = data scanned; always use columnar + partitions.
- QuickSight per-user pricing.

## Cleanup
```bash
cdk destroy
# pause/delete Redshift, QuickSight
```
