# 02 — Policy as Code

## Concept
SCPs (Organizations), cfn-guard (CFN static check), OPA/Rego (K8s + Terraform), cdk-nag (runtime synth).

## Exercises
1. **cfn-guard rules**: write a rule `all S3 buckets must have encryption`. Run against synth'd templates in CI.
2. **OPA Gatekeeper** on EKS: block pods without resource limits; require image from your ECR registry only.
3. **Kyverno** as alternative — same rule set; compare UX.
4. **SCP write-up**: design SCPs for Dev/Prod OUs — what to allow/deny. Test in sandbox account.
5. **IAM Access Analyzer policy generation**: let AAA generate least-privilege from CloudTrail history.
6. **cdk-nag aspect**: already in lesson 01; here, integrate into pipeline as blocker.

## Verification
- cfn-guard fails build on policy violation.
- Gatekeeper denies non-conformant pods.

## Gotchas
- SCP deny has no resource scoping — blanket.
- OPA Rego syntax has learning curve.

## Cleanup
```bash
cdk destroy
```
