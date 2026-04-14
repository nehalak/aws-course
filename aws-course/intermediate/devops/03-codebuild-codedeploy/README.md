# 03 — CodeBuild & CodeDeploy

## Concept
CodeBuild = managed build. `buildspec.yml`. CodeDeploy = in-place or Blue/Green for EC2/ECS/Lambda.

## Exercises
1. **CodeBuild project**: `buildspec.yml` runs tests + builds Docker image + pushes to ECR.
2. **Local builds**: `codebuild-local` Docker tool to test buildspec locally.
3. **CodeDeploy EC2 in-place**: deploy a new app version to ASG with one-at-a-time.
4. **CodeDeploy Lambda canary**: shift 10% traffic for 5 min then 100%. Include pre-/post-traffic hooks.
5. **CodeDeploy ECS Blue/Green**: traffic shift with automatic rollback on CloudWatch alarm.
6. **Artifact management**: producer build → S3 artifact → deployer consumes.

## Verification
- Canary deploy rolls back on forced 500s.
- In-place deploy keeps capacity during rolling update.

## Gotchas
- `appspec.yml` format differs per compute type.
- Blue/Green for ECS requires 2 TGs + 2 listeners.

## Cleanup
```bash
cdk destroy
```
