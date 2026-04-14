# Walkthrough — 02 ECS Fargate Intro

## About this service
**ECS (Elastic Container Service)** is AWS's container orchestrator. **Fargate** is a launch type where AWS runs your containers — you don't manage EC2 hosts. Define a **task definition** (containers, CPU, memory, env) and a **service** (desired count, LB, placement) and AWS keeps it running.

**Why it matters:** the sweet spot between Lambda (too constrained for long-running HTTP servers) and EC2 (too much ops). Most modern AWS-native apps run on Fargate.

**When to use Fargate:** long-running HTTP services, workers, web apps, scheduled jobs >15min. Good default for containerized production workloads.
**When NOT to use Fargate:** massive scale where EC2 is cheaper (>50% utilization sustained), GPU workloads (ECS-EC2 instead), Kubernetes ecosystem needs (EKS), serverless event handling (Lambda).

## Estimated cost
- **Fargate: $0.04048/vCPU-hr + $0.004445/GB-hr** (us-east-1 on-demand)
- **Fargate Spot: ~70% cheaper** with preemption
- Example: 2 tasks × (0.25 vCPU + 0.5 GB) × 24/7 = ~$10/month
- **ALB: $16/month base + data processing** ← this is often the biggest cost
- Total for this lesson: **~$25/month** if left running. Destroy after!

---

## Step 1: Dockerfile + app
> **Why:** Fargate runs Docker images. You need an image in a registry. The `public.ecr.aws` base avoids Docker Hub rate limits.

`app/Dockerfile`:
```dockerfile
FROM public.ecr.aws/docker/library/node:20-alpine
WORKDIR /app
COPY server.js .
EXPOSE 80
CMD ["node", "server.js"]
```

`app/server.js`:
```javascript
const http = require('http');
const os = require('os');
http.createServer((req, res) => {
  if (req.url === '/health') return res.end('ok');
  res.end(`hello from ${os.hostname()}\n`);
}).listen(80);
```

## Step 2: Stack
> **Why:** `ApplicationLoadBalancedFargateService` (L3) creates ~15 resources (VPC networking, ALB, TGs, listeners, ECS service, task def, ECR push, IAM roles). This is where CDK shines.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecsp from 'aws-cdk-lib/aws-ecs-patterns';
import { DockerImageAsset } from 'aws-cdk-lib/aws-ecr-assets';
import * as path from 'path';

export class FargateStack extends cdk.Stack {
  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });
    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    const image = new DockerImageAsset(this, 'Img', {
      directory: path.join(__dirname, '../app'),
    });

    const svc = new ecsp.ApplicationLoadBalancedFargateService(this, 'Svc', {
      cluster,
      desiredCount: 2,
      taskImageOptions: {
        image: ecs.ContainerImage.fromDockerImageAsset(image),
        containerPort: 80,
      },
      publicLoadBalancer: true,
      cpu: 256,
      memoryLimitMiB: 512,
      enableExecuteCommand: true,
    });

    svc.targetGroup.configureHealthCheck({ path: '/health' });

    new cdk.CfnOutput(this, 'AlbDns', { value: svc.loadBalancer.loadBalancerDnsName });
  }
}
```

First deploy: 5-8 min (Docker build, ECR push, service stabilize).

## Step 3: Test
> **Why:** Seeing different hostnames across curls proves load balancing — traffic really is spread across 2 containers on 2 different ENIs in 2 different AZs.

```bash
ALB=$(aws cloudformation describe-stacks --stack-name FargateStack \
  --query 'Stacks[0].Outputs[?OutputKey==`AlbDns`].OutputValue' --output text)

for i in {1..10}; do curl http://$ALB/; done
# mix of "hello from ip-10-x-x-x"
```

## Step 4: Scale out
> **Why:** Change one number, redeploy, 3 more tasks appear. In production this is auto-scaling on metrics — same mechanism.

Edit `desiredCount: 5`, redeploy:
```bash
aws ecs list-tasks --cluster <cluster> --service-name <service>
# 5 task ARNs
```

## Step 5: Exec into a task
> **Why:** `execute-command` is your `kubectl exec` for Fargate. Invaluable for debugging. Must be enabled at service creation (`enableExecuteCommand: true`).

```bash
TASK=$(aws ecs list-tasks --cluster <cluster> --service-name <svc> --query 'taskArns[0]' --output text)
aws ecs execute-command \
  --cluster <cluster> --task $TASK --container web \
  --interactive --command "/bin/sh"
```

## Step 6: Force failure
> **Why:** Seeing ECS's self-healing (replacing failed tasks in a restart loop) builds trust in the platform. When real bugs cause crashes, you know what to expect in the console.

Edit `server.js` to `process.exit(1)`. Redeploy. Service events:
```
service Svc has stopped 10 running tasks: Essential container exited
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **"unable to pull image"** (Fargate in private subnet with no NAT) — need NAT GW or VPC endpoints.
- **Health check fails, tasks cycle** — path `/health` wrong, or container port mismatch.
- **`execute-command` fails** — `enableExecuteCommand: true` not set + task role missing `ssmmessages:*`.
