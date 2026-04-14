# Walkthrough — 05 AWS Batch

## About this service
**AWS Batch** is a managed batch scheduler. You submit jobs; Batch picks compute (Fargate, EC2 on-demand, or EC2 Spot), provisions it, runs your container, and scales down to zero. Three pieces: **compute environment** (what compute pool), **job queue** (prioritized FIFO), **job definition** (container + resources). Add-ons: **array jobs** (one submit → N parallel children), **job dependencies** (DAG), **multi-node parallel** (MPI).

**Why it matters:** Long-running, CPU/GPU-heavy, embarrassingly parallel workloads (genomics, simulation, ML training, video transcoding) don't fit Lambda's 15-min cap. ECS services assume always-on. Batch gives you scale-to-zero with Spot discounts.

**When to use:** thousands of parallel jobs, >15-min runs, GPU training, HPC simulations, nightly processing queues.
**When NOT to use:** real-time request/response (use ECS/Lambda), workflow orchestration across services (use Step Functions), single long-running daemon (use ECS service).

## Estimated cost
- **Batch service: free** — you only pay for underlying compute
- **Fargate on-demand: $0.04048/vCPU-hr + $0.004445/GB-hr**
- **EC2 Spot: typically 70% off on-demand** (e.g., c5.large at $0.026/hr vs $0.085 on-demand)
- **ECR storage: $0.10/GB-month**
- **CloudWatch Logs: $0.50/GB ingested**
- Example: 1000 Fargate jobs × 1 vCPU × 2GB × 5 min = 83 vCPU-hr + 166 GB-hr = ~$4
- Total for this lesson: **~$5/month** (lab volumes)

---

## Step 1: Project setup
> **Why:** Batch L2 constructs are in `aws-cdk-lib/aws-batch` (GA since v2.96). Fargate for simple jobs, EC2 Spot for max savings on long CPU work.

```bash
mkdir batch-intro && cd batch-intro
cdk init app --language=typescript
npm install aws-cdk-lib constructs
```

## Step 2: VPC and Fargate compute environment
> **Why:** Batch Fargate needs a VPC with outbound internet (or VPC endpoints) to pull the image and write logs. `maxvCpus` caps spend — without it, Batch will happily spin up thousands of tasks.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as batch from 'aws-cdk-lib/aws-batch';
import * as ecs from 'aws-cdk-lib/aws-ecs';

export class BatchStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

    const fargateCE = new batch.FargateComputeEnvironment(this, 'FargateCE', {
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      maxvCpus: 32,
      spot: false,
    });

    const queue = new batch.JobQueue(this, 'JobQueue', {
      priority: 1,
    });
    queue.addComputeEnvironment(fargateCE, 1);
  }
}
```

## Step 3: Job definition (hello world)
> **Why:** Job definition = container image + resources + env. `command` can reference parameter substitution (`Ref::foo`). Logs default to CloudWatch — no extra config.

```typescript
import * as path from 'path';

const helloJobDef = new batch.EcsJobDefinition(this, 'HelloJob', {
  container: new batch.EcsFargateContainerDefinition(this, 'HelloContainer', {
    image: ecs.ContainerImage.fromRegistry('public.ecr.aws/docker/library/python:3.12-slim'),
    cpu: 0.25,
    memory: cdk.Size.mebibytes(512),
    command: ['python', '-c', 'print("hello batch")'],
  }),
});
```

Submit:
```bash
aws batch submit-job --job-name hello --job-queue <queueName> --job-definition <defName>
aws batch describe-jobs --jobs <jobId> --query 'jobs[0].status'
# "SUBMITTED" → "RUNNABLE" → "STARTING" → "RUNNING" → "SUCCEEDED"
aws logs tail /aws/batch/job --since 10m
# hello batch
```

## Step 4: Submit 100 jobs and watch the queue drain
> **Why:** Batch's value shines at scale. 100 submits in a loop; Batch spins up N Fargate tasks in parallel (capped by `maxvCpus` ÷ job vCPU) and drains the queue.

```bash
for i in $(seq 1 100); do
  aws batch submit-job --job-name hello-$i --job-queue <queueName> --job-definition <defName> >/dev/null &
done
wait

aws batch list-jobs --job-queue <queueName> --job-status RUNNING --query 'length(jobSummaryList)'
# 32   (because maxvCpus=32 / 0.25 vCPU per job allows up to 128 but Fargate scaling paces it)

# 2-3 min later:
aws batch list-jobs --job-queue <queueName> --job-status SUCCEEDED --query 'length(jobSummaryList)'
# 100
```

## Step 5: EC2 Spot compute environment (70% cheaper)
> **Why:** For CPU-bound batch processing, EC2 Spot saves massively. Cold start is worse (~5 min first job to provision EC2) but cost is dominant.

```typescript
const spotCE = new batch.ManagedEc2EcsComputeEnvironment(this, 'SpotCE', {
  vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  instanceTypes: [
    ec2.InstanceType.of(ec2.InstanceClass.C5, ec2.InstanceSize.LARGE),
    ec2.InstanceType.of(ec2.InstanceClass.C5, ec2.InstanceSize.XLARGE),
    ec2.InstanceType.of(ec2.InstanceClass.C5A, ec2.InstanceSize.LARGE),
  ],
  useOptimalInstanceClasses: false,
  spot: true,
  spotBidPercentage: 80,
  minvCpus: 0,
  maxvCpus: 64,
  allocationStrategy: batch.AllocationStrategy.SPOT_CAPACITY_OPTIMIZED,
});

const spotQueue = new batch.JobQueue(this, 'SpotQueue', { priority: 1 });
spotQueue.addComputeEnvironment(spotCE, 1);

const primeJobDef = new batch.EcsJobDefinition(this, 'PrimeJob', {
  container: new batch.EcsEc2ContainerDefinition(this, 'PrimeC', {
    image: ecs.ContainerImage.fromRegistry('public.ecr.aws/docker/library/python:3.12-slim'),
    cpu: 1,
    memory: cdk.Size.mebibytes(1024),
    command: ['python', '-c',
      'n=2_000_000; s=[True]*n;\nfor i in range(2,int(n**0.5)+1):\n  if s[i]:\n    for j in range(i*i,n,i): s[j]=False\nprint(sum(s))'],
  }),
  retryAttempts: 3,  // Spot reclaim → retry
});
```

Submit 1000 jobs → look at spend over 1 hour vs the same on-demand run.

## Step 6: Array jobs
> **Why:** `--array-properties size=50` spawns 50 child jobs from a single submit, each with `AWS_BATCH_JOB_ARRAY_INDEX` set 0..49. Perfect for "process file N of 50".

```bash
aws batch submit-job \
  --job-name array-demo \
  --job-queue <queueName> \
  --job-definition <defName> \
  --array-properties size=50

# Status shows parent + 50 children:
aws batch describe-jobs --jobs <parentId> --query 'jobs[0].arrayProperties'
# { "size": 50, "statusSummary": { "SUCCEEDED": 50 } }
```

Container references the index:
```bash
command: ['sh','-c','echo "I am child $AWS_BATCH_JOB_ARRAY_INDEX"']
```

## Step 7: Job dependencies
> **Why:** Build a DAG: job B only runs if job A succeeds. Batch enforces this server-side without a workflow engine.

```bash
A=$(aws batch submit-job --job-name A --job-queue <q> --job-definition <def> --query 'jobId' --output text)
B=$(aws batch submit-job --job-name B --job-queue <q> --job-definition <def> \
  --depends-on jobId=$A --query 'jobId' --output text)

aws batch describe-jobs --jobs $B --query 'jobs[0].status'
# "PENDING"  (waiting for A)
# → after A succeeds → "RUNNABLE" → ... → "SUCCEEDED"
```

For arrays, use `type=SEQUENTIAL` (child N depends on N-1) or `N_TO_N` (fan-in).

## Cleanup
```bash
# Disable CEs before delete or stack teardown hangs
aws batch update-compute-environment --compute-environment <name> --state DISABLED
cdk destroy
```

## Common Errors
- **Job stuck `RUNNABLE` forever** → compute environment can't scale (hit `maxvCpus`, no subnet capacity, or Spot market unavailable at bid). Check CE status + events.
- **`CannotPullContainerError`** → Fargate in private subnet with no NAT / no VPC endpoints for ECR + S3 + Logs.
- **Fargate job rejected: vCPU/memory combo invalid** → Fargate has fixed pairs (0.25/512, 0.25/1024, 0.5/1024, ...). 0.25 vCPU + 2048 MB is invalid.
- **Spot reclaim = job failure** → set `retryAttempts: 3`. Batch retries on a different host.
- **Too many job definition revisions** → every deploy creates a new `:N`. Deregister old ones periodically.
- **Array job "failed: 1 of 50"** — children are independent; one failing doesn't stop the rest. Check each child's logs.
