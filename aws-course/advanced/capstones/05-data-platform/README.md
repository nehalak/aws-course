# Capstone 05 — Data Platform

## Goal
End-to-end modern data platform: ingest, lake, warehouse, BI.

## Requirements
- Kinesis Firehose ingesting app events → S3 raw (Parquet, partitioned)
- DMS replicating RDS → S3 CDC
- Glue Catalog + Lake Formation governance
- Glue ETL: raw → curated (dedup, enrich, quality checks)
- dbt models in Redshift Serverless (or use Athena + Iceberg)
- Data quality: Glue Data Quality rules + AWS Deequ
- QuickSight dashboards for business users
- Observability: lineage via OpenLineage → DataZone
- Backfill runbook

## Deliverables
- CDK + dbt repo
- `data-contracts.md` per source
- QuickSight dashboard with 5+ viz
- Freshness SLAs + alarms

## Verification
- End-to-end data age < 15 min on live stream.
- DQ rules flag synthetic bad data.

## Gotchas
- Redshift Serverless warm-up = first query slow.
- Iceberg + Athena latest version for DML.

## Cleanup
```bash
cdk destroy
# empty S3 + pause Redshift
```
