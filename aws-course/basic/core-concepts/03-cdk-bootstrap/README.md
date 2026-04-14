# 03 — CDK Bootstrap

## Concept
CDK needs a staging bucket + IAM roles per account/region. `cdk bootstrap` creates them. `cdk synth` → CloudFormation YAML. `cdk deploy` → CFN stack.

## Exercises
1. **Bootstrap two regions**:
   ```bash
   cdk bootstrap aws://<ACCOUNT>/us-east-1
   cdk bootstrap aws://<ACCOUNT>/eu-west-1
   ```
2. **Inspect the bootstrap stack** in CloudFormation console — find the `CDKToolkit` stack. Identify: staging bucket, ECR repo, 5 IAM roles (file-publish, image-publish, lookup, deploy, cfn-exec). Document their purpose.
3. **First app**:
   ```bash
   mkdir hello-cdk && cd hello-cdk
   cdk init app --language=typescript
   ```
   Add one S3 bucket in `lib/hello-cdk-stack.ts`. Run:
   ```bash
   cdk synth      # inspect the CFN YAML
   cdk diff
   cdk deploy
   ```
4. **Context & env**: deploy the same stack to both regions using `env: { account, region }`.
5. **`cdk.context.json`** — do a `Vpc.fromLookup()` and observe the cache file appear. Commit it.

## Verification
- Stack appears in both regions' CloudFormation.
- `aws s3 ls` shows the bucket.

## Gotchas
- Bootstrap per region. Forgetting = "need to bootstrap" errors.
- Don't delete the `CDKToolkit` stack — it orphans state.

## Cleanup
```bash
cdk destroy
```
