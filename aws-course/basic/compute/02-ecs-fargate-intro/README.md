# 02 — ECS Fargate Intro

## Concept
Run containers without managing EC2. Task definition = container spec. Service = desired count + placement + LB.

## Exercises
1. **Build & push a container**:
   ```bash
   docker build -t hello .
   # CDK will create ECR repo or use DockerImageAsset
   ```
   Use CDK `DockerImageAsset` pointing to a local `Dockerfile` that serves "hello" on port 80.
2. **CDK Fargate service** using `ApplicationLoadBalancedFargateService` (L3 construct). Desired count 2.
3. **Curl the ALB DNS** → rotate between tasks (check hostname in response).
4. **Scale out**: update desired count to 5, redeploy. Watch tasks launch.
5. **Exec into task**: `aws ecs execute-command ...` — requires `enableExecuteCommand: true`.
6. **Fail a container**: make it exit 1 on boot. Observe ECS restart loop.

## Verification
- `curl <alb-dns>` returns 200.
- `ecs list-tasks` shows N running tasks.

## Gotchas
- ALB = $16/mo idle. Destroy when done.
- Fargate Spot is 70% cheaper but preemptible.
- Task role ≠ execution role (pull image, write logs) vs (app permissions).

## Cleanup
```bash
cdk destroy
```
