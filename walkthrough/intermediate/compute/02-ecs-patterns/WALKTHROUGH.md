# Walkthrough — 02 ECS Patterns

## About this service
**Amazon ECS** orchestrates Docker containers on Fargate or EC2. Beyond "run one service behind an ALB", production ECS means **service discovery** (Cloud Map for service-to-service DNS), **capacity providers** (mix on-demand + Spot for 70% savings), **Blue/Green deployments** (safe rollout via CodeDeploy), **scheduled tasks** (EventBridge-triggered), and **autoscaling on custom metrics** (SQS queue depth).

**Why it matters:** Vanilla ECS is easy. Getting to zero-downtime deploys, fault-tolerant service-to-service comms, and 70%-off compute is what separates demo apps from production.

**When to use:** microservices that need internal DNS (Cloud Map), fault-tolerant batch workers (Spot), regulated rollouts (Blue/Green), nightly jobs (Scheduled task), queue-driven workers (SQS scaling).
**When NOT to use:** single-service apps (App Runner is simpler), workloads that need pod-level k8s ecosystem (use EKS), per-request billing preferred (Lambda).

## Estimated cost
- **Fargate on-demand: $0.04048/vCPU-hr + $0.004445/GB-hr** → 2 tasks × 0.5 vCPU × 1GB × 730h = ~$33/month
- **Fargate Spot: ~70% off** → same as above = ~$10/month
- **ALB: $16/month** (base) + $0.008/LCU-hr
- **Cloud Map: $0.50/hosted zone/month** + $0.40 per 1M queries
- **CodeDeploy for ECS: free**
- **NAT Gateway: $32/month** + $0.045/GB processed
- Total for this lesson: **~$90/month** (mostly NAT + ALB + Fargate)

---

## Step 1: Project setup and VPC
> **Why:** ECS on Fargate needs a VPC with at least 2 AZ subnets for HA. One NAT gateway = cheap; two = HA but doubles cost. Use one for labs.

```bash
mkdir ecs-patterns && cd ecs-patterns
cdk init app --language=typescript
npm install aws-cdk-lib constructs
```

```typescript
// lib/vpc.ts
import * as ec2 from 'aws-cdk-lib/aws-ec2';
export const makeVpc = (scope: any) => new ec2.Vpc(scope, 'Vpc', {
  maxAzs: 2,
  natGateways: 1,
});
```

## Step 2: Cluster + Cloud Map namespace
> **Why:** A Cloud Map **private DNS namespace** (`svc.local`) lets services find each other by name without hardcoded IPs or ALBs. `A` records get registered/deregistered automatically as tasks come and go.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as cloudmap from 'aws-cdk-lib/aws-servicediscovery';

export class EcsPatternsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc,
      defaultCloudMapNamespace: { name: 'svc.local', type: cloudmap.NamespaceType.DNS_PRIVATE },
      enableFargateCapacityProviders: true,
    });
  }
}
```

## Step 3: Service A and B with service discovery
> **Why:** Service A calls Service B by DNS name `b.svc.local`. Tasks re-register automatically on restart, so even after a deploy the name still resolves.

```typescript
const taskA = new ecs.FargateTaskDefinition(this, 'TaskA', { cpu: 256, memoryLimitMiB: 512 });
taskA.addContainer('c', {
  image: ecs.ContainerImage.fromRegistry('nginx:latest'),
  portMappings: [{ containerPort: 80 }],
  logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'a', logRetention: 7 }),
});

const serviceA = new ecs.FargateService(this, 'SvcA', {
  cluster, taskDefinition: taskA, desiredCount: 2,
  cloudMapOptions: { name: 'a', dnsTtl: cdk.Duration.seconds(10) },
});

// Repeat for B, then let A reach B:
serviceA.connections.allowTo(serviceB.connections, ec2.Port.tcp(80));

// From a shell inside task A:
// curl http://b.svc.local/    → hits service B round-robin
```

## Step 4: Capacity provider mix (Fargate + Spot)
> **Why:** `base: 1, weight: 1` on FARGATE keeps one always-on safety task; weight 4 on SPOT means 4/5 new tasks land on Spot. If Spot gets reclaimed, the service self-heals. You get ~70% discount on those tasks.

```typescript
const spotService = new ecs.FargateService(this, 'SpotSvc', {
  cluster,
  taskDefinition: taskA,
  desiredCount: 5,
  capacityProviderStrategies: [
    { capacityProvider: 'FARGATE', weight: 1, base: 1 },
    { capacityProvider: 'FARGATE_SPOT', weight: 4 },
  ],
});
```

Observe:
```bash
aws ecs list-tasks --cluster <c> --service-name SpotSvc --query 'taskArns' | head
aws ecs describe-tasks --cluster <c> --tasks <arn> --query 'tasks[].capacityProviderName'
# "FARGATE" "FARGATE_SPOT" "FARGATE_SPOT" "FARGATE_SPOT" "FARGATE_SPOT"
```

## Step 5: Blue/Green deployment via CodeDeploy
> **Why:** Rolling deploys can break if a new version has bugs — you've already shifted all traffic. Blue/Green keeps the old version live on a target group, deploys the new one to a second target group, then you **shift traffic gradually** with auto-rollback on alarms.

```typescript
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';

const alb = new elbv2.ApplicationLoadBalancer(this, 'Alb', { vpc, internetFacing: true });
const prodListener = alb.addListener('Prod', { port: 80 });
const testListener = alb.addListener('Test', { port: 8080 });

const tgBlue = new elbv2.ApplicationTargetGroup(this, 'Blue', {
  vpc, port: 80, protocol: elbv2.ApplicationProtocol.HTTP,
  targetType: elbv2.TargetType.IP,
});
const tgGreen = new elbv2.ApplicationTargetGroup(this, 'Green', {
  vpc, port: 80, protocol: elbv2.ApplicationProtocol.HTTP,
  targetType: elbv2.TargetType.IP,
});
prodListener.addTargetGroups('p', { targetGroups: [tgBlue] });
testListener.addTargetGroups('t', { targetGroups: [tgGreen] });

const bgService = new ecs.FargateService(this, 'BgSvc', {
  cluster, taskDefinition: taskA, desiredCount: 2,
  deploymentController: { type: ecs.DeploymentControllerType.CODE_DEPLOY },
});
bgService.attachToApplicationTargetGroup(tgBlue);

new codedeploy.EcsDeploymentGroup(this, 'DG', {
  service: new codedeploy.EcsApplication(this, 'App', {}) as any,
  blueGreenDeploymentConfig: {
    blueTargetGroup: tgBlue,
    greenTargetGroup: tgGreen,
    listener: prodListener,
    testListener,
  },
  deploymentConfig: codedeploy.EcsDeploymentConfig.CANARY_10PERCENT_5MINUTES,
});
```

Deploy v2: CodeDeploy shifts 10% → wait 5 min → 100%. Console shows both target groups live during cutover.

## Step 6: Scheduled Fargate task
> **Why:** Cron-style jobs (DB migrations, nightly reports) should run as one-shot tasks, not long-running services. EventBridge triggers RunTask on a schedule.

```typescript
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

new events.Rule(this, 'NightlyMigrate', {
  schedule: events.Schedule.cron({ minute: '0', hour: '3' }),
  targets: [new targets.EcsTask({
    cluster,
    taskDefinition: taskA,
    subnetSelection: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  })],
});
```

## Step 7: Autoscale on SQS queue depth
> **Why:** Worker services should scale on **backlog**, not CPU. 1000 messages and 2 workers? Scale up. Empty queue for 5 min? Scale down.

```typescript
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as cw from 'aws-cdk-lib/aws-cloudwatch';
import * as appscaling from 'aws-cdk-lib/aws-applicationautoscaling';

const queue = new sqs.Queue(this, 'Jobs');
queue.grantConsumeMessages(taskA.taskRole);

const scaling = serviceA.autoScaleTaskCount({ minCapacity: 1, maxCapacity: 20 });
scaling.scaleOnMetric('QueueDepth', {
  metric: queue.metricApproximateNumberOfMessagesVisible({ period: cdk.Duration.minutes(1) }),
  scalingSteps: [
    { upper: 10, change: -1 },
    { lower: 100, change: +2 },
    { lower: 1000, change: +5 },
  ],
  adjustmentType: appscaling.AdjustmentType.CHANGE_IN_CAPACITY,
});
```

## Cleanup
```bash
cdk destroy
# Manually delete ENIs if any are orphaned in the VPC
```

## Common Errors
- **Task fails with "ResourceInitializationError: unable to pull image"** → Fargate in private subnet needs NAT (or VPC endpoints for ECR + S3 + Logs).
- **`curl: Could not resolve host: b.svc.local`** → both services must be in the same cluster with the same namespace; DNS only works inside the VPC.
- **Blue/Green stuck** → check CodeDeploy logs; often the new task fails health check on test listener.
- **Spot tasks cycling** → Spot reclaim rate high in your AZ; add FARGATE base capacity.
- **`Cloud Map DNS returns stale IP`** → lower `dnsTtl` to 10s; default 60s caches too aggressively.
