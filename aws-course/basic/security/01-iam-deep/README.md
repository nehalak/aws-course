# 01 — IAM Deep

## Concept
Identity (user/role/group) + policy (allow/deny) + resource = access decision. Roles > users for workloads.

## Exercises
1. **Policy anatomy**: write an IAM policy JSON that allows `s3:GetObject` only on `arn:aws:s3:::my-bucket/reports/*` only when `aws:SourceIp` is your home IP.
2. **User, group, policy**: create IAM group `readers`, attach `AmazonS3ReadOnlyAccess`, add a user `alice`. Log in as alice, verify read works, write fails.
3. **Role for EC2**: CDK a role with `AmazonS3ReadOnlyAccess`, attach to an EC2. SSH in, `aws s3 ls` — works without credentials.
4. **AssumeRole**: create a role `DeployRole` trusted by your `cdk-dev` user. Assume it:
   ```bash
   aws sts assume-role --role-arn <arn> --role-session-name test
   ```
5. **Identity vs resource policy**: create an S3 bucket. Grant `alice` access via (a) identity policy, (b) bucket policy. Understand both work; understand `Deny` beats `Allow`.
6. **Policy simulator**: use the IAM Policy Simulator to test a denied action and understand why.

## Verification
- alice can `get-object` but not `put-object`.
- EC2 can list S3 via role.
- Assumed role creds differ from user creds (`sts get-caller-identity`).

## Gotchas
- Inline vs managed policies — prefer managed for reuse.
- `*` in resource is usually wrong. Scope tight.
- Role trust policy is separate from permission policy.

## Cleanup
```bash
cdk destroy
# delete alice, group, assumed role
```
