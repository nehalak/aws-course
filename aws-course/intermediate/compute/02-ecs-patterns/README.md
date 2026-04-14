# 02 — ECS Patterns

## Concept
Service discovery (Cloud Map), capacity providers (Fargate + Spot), Blue/Green via CodeDeploy, task scheduling.

## Exercises
1. **Service Discovery**: two Fargate services in a Cloud Map namespace `svc.local`. Service A resolves `b.svc.local` → B.
2. **Capacity provider mix**: `FARGATE` weight 1 base 1, `FARGATE_SPOT` weight 4. Observe task distribution.
3. **Blue/Green deploy**: `DeploymentController.CODE_DEPLOY`. Deploy v2; test traffic on green listener; shift 10%/50%/100%.
4. **Scheduled task**: EventBridge rule runs a Fargate task nightly to run a migration.
5. **EC2 launch type**: same service but on ECS-EC2 with ASG capacity provider. Observe bin-packing.
6. **Task scaling**: step scaling on SQS queue depth.

## Verification
- Blue/Green rollback works by not shifting 100%.
- Spot interruption handled gracefully with `stopTimeout`.

## Gotchas
- Cloud Map DNS TTL can cache stale IPs.
- Fargate Spot interruption warning = 2 min.
- Blue/Green requires 2 target groups + 2 listeners.

## Cleanup
```bash
cdk destroy
```
