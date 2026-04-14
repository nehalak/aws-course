# 01 — CDK Mastery

## Concept
Custom constructs lib, jsii (multi-language), aspects for policy, cdk-nag, unit testing with assertions.

## Exercises
1. **Construct library**: scaffold `projen` + jsii project publishing `@yourorg/aws-secure-patterns`. Include `SecureBucket`, `HardenedVpc`, `AuditedLambda`.
2. **Publish locally** with `npm pack`; consume in a separate CDK app.
3. **Aspects for policy**: aspect that errors if any resource missing `CostCenter` tag.
4. **cdk-nag**: apply AWS Solutions + HIPAA packs; fix or suppress every finding.
5. **Unit tests**: `@aws-cdk/assertions` — assert bucket is encrypted, S3 blockPublicAccess true, IAM `Action` list.
6. **Snapshot test**: `Template.fromStack(stack).toJSON()` snapshot diffing.
7. **Integration tests**: `@aws-cdk/integ-tests` actually deploys + asserts.

## Verification
- `npm test` passes assertions.
- cdk-nag CI gate blocks PR on violations.

## Gotchas
- jsii locks you to a subset of TS features.
- Aspects run during synth — heavy ones slow you down.

## Cleanup
```bash
cdk destroy
```
