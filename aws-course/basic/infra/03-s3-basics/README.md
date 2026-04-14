# 03 — S3 Basics

## Concept
Object storage. 99.999999999% durability. Prefix-based (no real folders). Storage classes trade access speed vs cost.

## Exercises
1. **CDK bucket** with:
   - Versioning ON
   - Block public access ON
   - SSE-S3 encryption
   - Lifecycle: move to `INTELLIGENT_TIERING` after 30 days, delete old versions after 90.
2. **Upload & version**:
   ```bash
   echo "v1" > file.txt && aws s3 cp file.txt s3://<bucket>/
   echo "v2" > file.txt && aws s3 cp file.txt s3://<bucket>/
   aws s3api list-object-versions --bucket <bucket>
   ```
3. **Restore a prior version** via `aws s3api copy-object`.
4. **Storage classes**: upload one 1GB file each to STANDARD, IA, GLACIER. Compare costs in Cost Explorer next day.
5. **Presigned URL**: generate one with `aws s3 presign` valid for 5 min. Share → observe access.
6. **Static website**: enable website hosting, upload `index.html`, access via website endpoint (NOT recommended for prod — use CloudFront).

## Verification
- `aws s3 ls` shows 2 versions.
- Presigned URL works before expiry, 403 after.

## Gotchas
- Bucket names are **globally** unique.
- Public buckets = #1 cause of data leaks. Keep Block Public Access ON.
- `DELETE` on versioned bucket creates a delete marker — doesn't free storage.

## Cleanup
```bash
aws s3 rm s3://<bucket> --recursive
aws s3api delete-objects ...  # delete all versions
cdk destroy
```
