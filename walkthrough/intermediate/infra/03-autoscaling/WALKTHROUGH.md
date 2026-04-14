# Walkthrough — 03 Auto Scaling Groups

## About this service
**Auto Scaling Group (ASG)** keeps N healthy EC2 instances running. It replaces dead instances, spreads across AZs, launches from a **launch template** (AMI + instance type + user data), and scales on **policies** — target tracking (simplest, most common), step scaling (custom ladders), or scheduled (known traffic patterns).

**Why it matters:** ASGs are how stateless EC2 workloads stay alive without ops babysitting. They're also the layer where spot interruptions, mixed instance types, and zero-downtime deploys happen.

**When to use ASG:** EC2-based apps behind a load balancer, batch workers scaling on queue depth, Spot fleets.
**When NOT to use:** anything serverless (Lambda/Fargate has its own scaling), single-box tools, or stateful workloads where rotation breaks data (use statefulsets/EKS or RDS for those).

## Estimated cost
- **ASG itself: free** — you pay for the EC2
- **2× t3.micro (on-demand, us-east-1): ~$15/month**
- **Same but Spot: ~$4.50/month** (~70% off)
- **CloudWatch alarms for scaling: $0.10/alarm/month** (first 10 free)
- **ALB in front: ~$16.40/month**
- Total for this lesson: **~$35/month** at min=2. Destroy promptly.

---

## Step 1: Scaffold + ASG behind ALB
> **Why:** You always operate an ASG with an ALB in real systems — the ALB health checks feed the ASG's "replace unhealthy" logic. Setup is identical to Lesson 02 but focused on scaling behavior.

```bash
mkdir asg && cd asg
cdk init app --language=typescript
npm install aws-cdk-lib
```

`lib/asg-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as autoscaling from 'aws-cdk-lib/aws-autoscaling';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';

export class AsgStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

    const userData = ec2.UserData.custom(`#!/bin/bash
dnf install -y nginx stress-ng
echo "$(hostname)" > /usr/share/nginx/html/index.html
echo ok > /usr/share/nginx/html/health
systemctl enable --now nginx`);

    const asg = new autoscaling.AutoScalingGroup(this, 'WebAsg', {
      vpc,
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
      machineImage: ec2.MachineImage.latestAmazonLinux2023(),
      minCapacity: 2, maxCapacity: 6, desiredCapacity: 2,
      userData,
      healthCheck: autoscaling.HealthCheck.elb({ grace: cdk.Duration.minutes(2) }),
      ssmSessionPermissions: true,
    });

    const alb = new elbv2.ApplicationLoadBalancer(this, 'Alb', { vpc, internetFacing: true });
    const tg = new elbv2.ApplicationTargetGroup(this, 'Tg', {
      vpc, port: 80, protocol: elbv2.ApplicationProtocol.HTTP,
      targetType: elbv2.TargetType.INSTANCE,
      healthCheck: { path: '/health' },
    });
    asg.attachToApplicationTargetGroup(tg);
    alb.addListener('Http', { port: 80, defaultTargetGroups: [tg] });

    // --- Step 2: target tracking on CPU ---
    asg.scaleOnCpuUtilization('CpuScale', {
      targetUtilizationPercent: 50,
      cooldown: cdk.Duration.seconds(60),
    });

    // --- Step 3: step scaling alternative (commented target first) ---
    // asg.scaleOnMetric('StepCpu', {
    //   metric: new cloudwatch.Metric({
    //     namespace: 'AWS/EC2', metricName: 'CPUUtilization',
    //     dimensionsMap: { AutoScalingGroupName: asg.autoScalingGroupName },
    //     statistic: 'Average', period: cdk.Duration.minutes(1),
    //   }),
    //   scalingSteps: [
    //     { upper: 40, change: -1 },
    //     { lower: 60, upper: 80, change: +1 },
    //     { lower: 80, change: +3 },
    //   ],
    //   adjustmentType: autoscaling.AdjustmentType.CHANGE_IN_CAPACITY,
    // });

    // --- Step 4: scheduled actions ---
    asg.scaleOnSchedule('BusinessHours', {
      schedule: autoscaling.Schedule.cron({ minute: '0', hour: '13', weekDay: 'MON-FRI' }), // 9am ET
      minCapacity: 5,
    });
    asg.scaleOnSchedule('OffHours', {
      schedule: autoscaling.Schedule.cron({ minute: '0', hour: '1' }),
      minCapacity: 1,
    });

    new cdk.CfnOutput(this, 'AlbDns', { value: alb.loadBalancerDnsName });
    new cdk.CfnOutput(this, 'AsgName', { value: asg.autoScalingGroupName });
  }
}
```

Deploy:
```bash
cdk deploy
```

Expect: 2 instances launch, ALB shows them healthy after ~90s.

## Step 2: Target tracking — induce CPU load
> **Why:** Target tracking is the "set it and forget it" option — AWS manages the alarms and step math for you. You tell it a target metric, it maintains it. The only way to feel this is to push the metric past target and watch scale-out happen.

Shell into one instance via SSM and burn CPU:
```bash
INSTANCE_ID=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names <asg-name> \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)

aws ssm start-session --target $INSTANCE_ID
# inside:
stress-ng --cpu 2 --timeout 600s &
```

Watch activity:
```bash
watch -n 5 "aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name <asg-name> \
  --max-items 3 --query 'Activities[].[StartTime,StatusCode,Cause]' --output text"
```

Expected (after 3–5 min):
```
2026-04-14T...  Successful  At 2026-... a monitor alarm ... in state ALARM triggered policy
                            CpuScale... changing the desired capacity from 2 to 3.
```

## Step 3: Step scaling (advanced)
> **Why:** Step scaling lets you define custom aggression: "bump by 1 if CPU 60–80, bump by 3 if >80." Useful when target-tracking oscillates or when you have asymmetric costs (slow to scale out = dropped traffic).

Swap the commented block in the stack above. Redeploy. Push CPU >80% to see the +3 jump in a single step.

## Step 4: Scheduled scaling
> **Why:** Known patterns (business hours, batch jobs) are cheaper with schedule than reactive. Reactive scaling always lags — schedules pre-warm.

Already configured. Inspect:
```bash
aws autoscaling describe-scheduled-actions --auto-scaling-group-name <asg-name> \
  --query 'ScheduledUpdateGroupActions[].[ScheduledActionName,Recurrence,MinSize]' --output table
```

## Step 5: Lifecycle hooks
> **Why:** Hooks give you a window to finish work before a doomed instance dies — drain connections, flush logs, upload state. Without them, ASG yanks the instance mid-request.

Add to the stack:

```typescript
import * as sns from 'aws-cdk-lib/aws-sns';
import * as snsSubs from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';

const drainFn = new lambda.Function(this, 'DrainFn', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    const { AutoScalingClient, CompleteLifecycleActionCommand } = require('@aws-sdk/client-auto-scaling');
    exports.handler = async (e) => {
      console.log('Draining', JSON.stringify(e));
      // ... actual drain logic here (deregister from TG, wait for in-flight, etc.) ...
      const asg = new AutoScalingClient({});
      for (const r of e.Records) {
        const m = JSON.parse(r.Sns.Message);
        await asg.send(new CompleteLifecycleActionCommand({
          LifecycleHookName: m.LifecycleHookName,
          AutoScalingGroupName: m.AutoScalingGroupName,
          LifecycleActionToken: m.LifecycleActionToken,
          LifecycleActionResult: 'CONTINUE',
        }));
      }
    };
  `),
});

const topic = new sns.Topic(this, 'DrainTopic');
topic.addSubscription(new snsSubs.LambdaSubscription(drainFn));

asg.addLifecycleHook('OnTerminate', {
  lifecycleTransition: autoscaling.LifecycleTransition.INSTANCE_TERMINATING,
  notificationTarget: /* role + topic via L1 if needed */ undefined as any,
  heartbeatTimeout: cdk.Duration.minutes(5),
  defaultResult: autoscaling.DefaultResult.CONTINUE,
});
```

Trigger a terminate and watch the hook delay the kill by up to 5 minutes while the Lambda drains.

## Step 6: Instance refresh
> **Why:** When you bake a new AMI, you want to rotate instances without downtime. Instance refresh does this with configurable min-healthy percentage.

Update the launch template's AMI (change `machineImage` or userData), deploy, then:

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name <asg-name> \
  --preferences MinHealthyPercentage=50,InstanceWarmup=120
```

Check:
```bash
aws autoscaling describe-instance-refreshes --auto-scaling-group-name <asg-name> \
  --query 'InstanceRefreshes[0].[Status,PercentageComplete]'
```

Expected progression:
```
["Pending", 0]
["InProgress", 25]
["InProgress", 50]
["Successful", 100]
```

## Cleanup
> **Why:** Idle ASG + ALB is ~$30/mo even without traffic.

```bash
cdk destroy
```

## Common Errors
- **Instances launch then terminate repeatedly** → ELB health check failing (wrong path, SG blocking, app not ready before grace period). Raise `grace` or fix the app.
- **Target tracking doesn't scale out** → metric never exceeds target. Check dimension names; default `CPUUtilization` is per-instance average.
- **`LaunchTemplateNotFound` on refresh** → CDK replaced the LT; wait for stack update to finish before `start-instance-refresh`.
- **Spot interruption drops capacity** → enable **Capacity Rebalancing** on the ASG.
- **Scheduled action fires at wrong hour** → cron in schedules uses **UTC**. Translate your local time.
