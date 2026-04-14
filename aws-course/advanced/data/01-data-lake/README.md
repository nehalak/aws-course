# 01 — Data Lake

## Concept
S3 as storage, Glue Catalog as metadata, Lake Formation for fine-grained permissions, Athena to query.

## Exercises
1. **Raw/Curated/Analytics zones**: three S3 buckets with lifecycle appropriate to each.
2. **Glue crawler**: point at `raw/` bucket CSV; creates Glue tables.
3. **Glue ETL job** (Python Shell or Spark): transform CSV → Parquet + partition by date → write to `curated/`.
4. **Athena**: query curated table. Observe partition pruning.
5. **Lake Formation permissions**: grant user `alice` SELECT only on non-PII columns; deny access to `ssn` column.
6. **LF-tags**: tag tables `PII=yes/no`; grant access by tag.
7. **Data quality**: Glue Data Quality rules on the table (null %, unique, range).

## Verification
- Athena respects LF column-level filtering.
- Parquet queries cost < 1% of CSV equivalent.

## Gotchas
- Lake Formation has its own permissions model — both IAM AND LF must allow.
- Glue crawlers cost per DPU-hour; schedule wisely.

## Cleanup
```bash
cdk destroy
# empty S3 buckets first
```
