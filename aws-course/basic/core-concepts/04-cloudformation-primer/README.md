# 04 — CloudFormation Primer

## Concept
CDK compiles to CFN. Understanding CFN = understanding failure modes (ROLLBACK_FAILED, UPDATE_FAILED, drift).

## Exercises
1. **Raw CFN**: write `bucket.yaml` that creates one S3 bucket. Deploy:
   ```bash
   aws cloudformation deploy --stack-name learn-cfn --template-file bucket.yaml
   ```
2. **Update & observe**: add a tag, redeploy. Watch events in console.
3. **Force a failure**: set `BucketName` to an already-taken name. Observe ROLLBACK. Read the events.
4. **Drift**: manually add a tag in S3 console. Run drift detection — see the drift.
5. **Delete stack** and recreate with same `BucketName` — understand DeletionPolicy. Set `DeletionPolicy: Retain` and test.
6. **CDK parity**: recreate the same bucket in CDK. Run `cdk synth` and diff the output YAML against your hand-written template.

## Verification
- Drift report shows the manual tag.
- Retained bucket survives `delete-stack`.

## Gotchas
- `ROLLBACK_COMPLETE` stack cannot be updated — must delete + recreate.
- Physical IDs: some resources (S3 buckets) can't change name without replace.

## Cleanup
```bash
aws s3 rb s3://<bucket> --force
aws cloudformation delete-stack --stack-name learn-cfn
```
