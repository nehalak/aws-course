# 01 — CDK Patterns

## Concept
L1 = raw CFN. L2 = typed construct with defaults. L3 = opinionated patterns. Aspects = cross-cutting enforcement. Custom resources = CFN for non-CFN things.

## Exercises
1. **Build an L3 construct**: `SecureBucket` wrapping S3 with your org's defaults (encryption, versioning, lifecycle, block public). Publish as npm module locally; consume in another project.
2. **Aspect**: enforce all buckets have `RemovalPolicy.RETAIN`. Add CDK aspect that visits tree and asserts.
3. **cdk-nag**: add AWS Solutions NagPack. Fix all errors in one of your basic/ stacks.
4. **Custom resource**: on deploy, run a Lambda that seeds a DynamoDB table with initial data.
5. **Cross-stack references**: StackA exports VPC → StackB imports via `Fn.importValue` or CDK references.
6. **Environment-agnostic vs env-aware**: deploy same app to 2 regions using `env: {region}`.

## Verification
- `cdk synth` shows your L3 expands properly.
- cdk-nag fails build on violations.

## Gotchas
- Cross-stack refs create tight coupling; prefer SSM params for loose.
- Custom resources are Lambdas — can fail deploy if buggy.

## Cleanup
```bash
cdk destroy
```
