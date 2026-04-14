# Walkthrough — 02 Multi-AZ with Load Balancers

## About this service
**ELB (Elastic Load Balancing)** has three flavors:
- **ALB (Application Load Balancer)** — Layer 7. Understands HTTP(S), routes by path/host/header, does WebSockets, terminates TLS. Default choice for web apps and APIs.
- **NLB (Network Load Balancer)** — Layer 4. TCP/UDP pass-through, static IP per AZ, preserves client IP without X-Forwarded-For. Use for gaming, databases, or anything that isn't HTTP.
- **GWLB (Gateway Load Balancer)** — for inline network appliances (firewalls, IDS). Rarely relevant outside netsec.

**Why it matters:** a load balancer is how multi-AZ stops being theater. Without it, you have two servers in two AZs and no way to actually spread traffic, fail over, or do zero-downtime deploys.

**When to use ALB:** HTTP/HTTPS, gRPC, WebSockets, path-based routing, AWS Cognito/OIDC integration.
**When to use NLB:** non-HTTP (TCP/UDP), need static IPs, need to preserve source IP natively, ultra-low latency (µs not ms).
**When NOT to use either:** tiny one-box apps — use Route 53 + single ASG with health check. For public-facing static sites, use CloudFront + S3.

## Estimated cost
- **ALB: ~$16.40/month** + ~$0.008/LCU-hour (usually $1–$5/mo on light traffic)
- **NLB: ~$16.40/month** + $0.006/NLCU-hour
- **Target Group / health checks: free**
- **Cross-zone traffic on NLB: $0.01/GB** (ALB cross-zone is free)
- Total for this lesson: **~$40/month** (ALB + NLB + 2 EC2 t3.micro + small Fargate task). Destroy after.

---

## Step 1: Scaffold
> **Why:** Fresh CDK project keeps the LB stack and associated VPC isolated.

```bash
mkdir multi-az && cd multi-az
cdk init app --language=typescript
npm install aws-cdk-lib
```

## Step 2: ALB with two target groups (path-based routing)
> **Why:** The "single ALB, many services" pattern is the workhorse of most AWS apps. You learn listener rules, target groups, and health checks all at once.

`lib/multi-az-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as autoscaling from 'aws-cdk-lib/aws-autoscaling';

export class MultiAzStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

    // --- ALB in public subnets ---
    const alb = new elbv2.ApplicationLoadBalancer(this, 'Alb', {
      vpc, internetFacing: true,
    });

    // --- Target Group A: Fargate service ---
    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });
    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', { cpu: 256, memoryLimitMiB: 512 });
    taskDef.addContainer('app', {
      image: ecs.ContainerImage.fromRegistry('hashicorp/http-echo'),
      command: ['-text=hello-from-fargate', '-listen=:8080'],
      portMappings: [{ containerPort: 8080 }],
    });
    const fargateSvc = new ecs.FargateService(this, 'Svc', {
      cluster, taskDefinition: taskDef, desiredCount: 2,
    });

    const tgA = new elbv2.ApplicationTargetGroup(this, 'TgA', {
      vpc, port: 8080, protocol: elbv2.ApplicationProtocol.HTTP,
      targetType: elbv2.TargetType.IP,
      healthCheck: { path: '/health', healthyHttpCodes: '200-299', interval: cdk.Duration.seconds(15) },
    });
    fargateSvc.attachToApplicationTargetGroup(tgA);

    // --- Target Group B: EC2 ASG ---
    const asg = new autoscaling.AutoScalingGroup(this, 'WebAsg', {
      vpc,
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
      machineImage: ec2.MachineImage.latestAmazonLinux2023(),
      minCapacity: 2, maxCapacity: 4,
      userData: ec2.UserData.custom(`#!/bin/bash
dnf install -y nginx
echo 'hello-from-ec2' > /usr/share/nginx/html/index.html
echo 'ok' > /usr/share/nginx/html/health
systemctl enable --now nginx`),
    });

    const tgB = new elbv2.ApplicationTargetGroup(this, 'TgB', {
      vpc, port: 80, protocol: elbv2.ApplicationProtocol.HTTP,
      targetType: elbv2.TargetType.INSTANCE,
      healthCheck: { path: '/health' },
    });
    asg.attachToApplicationTargetGroup(tgB);

    // --- Listener + rules ---
    const listener = alb.addListener('Http', {
      port: 80,
      defaultAction: elbv2.ListenerAction.forward([tgB]),
    });
    listener.addAction('ApiPath', {
      priority: 10,
      conditions: [elbv2.ListenerCondition.pathPatterns(['/api/*'])],
      action: elbv2.ListenerAction.forward([tgA]),
    });

    // --- Step 3: stickiness on TG-B ---
    const cfnTgB = tgB.node.defaultChild as elbv2.CfnTargetGroup;
    cfnTgB.targetGroupAttributes = [
      { key: 'stickiness.enabled', value: 'true' },
      { key: 'stickiness.type', value: 'lb_cookie' },
      { key: 'stickiness.lb_cookie.duration_seconds', value: '3600' },
    ];

    new cdk.CfnOutput(this, 'AlbDns', { value: alb.loadBalancerDnsName });
  }
}
```

```bash
cdk deploy
```

Test:
```bash
curl http://<alb-dns>/          # hello-from-ec2
curl http://<alb-dns>/api/foo   # hello-from-fargate (echoes /api/foo)
```

## Step 3: Health checks and failure injection
> **Why:** Health checks are how the LB knows which targets are alive. The only way to trust them is to break something and watch the TG mark it unhealthy.

Kill nginx on one EC2:
```bash
aws ssm send-command --instance-ids i-xxxx --document-name AWS-RunShellScript \
  --parameters 'commands=["systemctl stop nginx"]'
```

Watch:
```bash
watch -n 2 "aws elbv2 describe-target-health --target-group-arn <tg-b-arn> \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Reason]' --output table"
```

Expected progression:
```
| i-aaaa | healthy    | None                     |
| i-bbbb | healthy    | None                     |
...after 2 failed checks...
| i-aaaa | unhealthy  | Target.FailedHealthChecks|
| i-bbbb | healthy    | None                     |
```

Create alarm on `UnHealthyHostCount`:
```typescript
new cloudwatch.Alarm(this, 'UnhealthyAlarm', {
  metric: tgB.metrics.unhealthyHostCount(),
  threshold: 1, evaluationPeriods: 2,
});
```

## Step 4: Sticky sessions
> **Why:** Stateful apps (old-school sessions, WebSocket rooms) break if requests ping-pong between targets. Cookie stickiness pins a client to a target.

Already enabled in Step 2. Verify:
```bash
curl -v http://<alb-dns>/ 2>&1 | grep -i set-cookie
# Set-Cookie: AWSALB=...; Expires=...; Path=/
curl -b "AWSALB=<cookie>" http://<alb-dns>/  # same backend every time
```

## Step 5: NLB for TCP pass-through
> **Why:** ALB can't do non-HTTP. If you need to front Postgres or a game server, NLB is the answer. Also the only ELB with a stable static IP per AZ.

Add to the stack:

```typescript
const nlb = new elbv2.NetworkLoadBalancer(this, 'Nlb', {
  vpc, internetFacing: false,
});
const nlbTg = new elbv2.NetworkTargetGroup(this, 'NlbTg', {
  vpc, port: 5432, protocol: elbv2.Protocol.TCP,
  targetType: elbv2.TargetType.INSTANCE,
  healthCheck: { port: '5432', protocol: elbv2.Protocol.TCP },
});
nlb.addListener('Pg', { port: 5432, defaultTargetGroups: [nlbTg] });
new cdk.CfnOutput(this, 'NlbDns', { value: nlb.loadBalancerDnsName });
```

NLB endpoint preserves source IP — no X-Forwarded-For needed. Cross-zone balancing is **disabled by default** (unlike ALB) because cross-zone traffic bills at $0.01/GB.

## Step 6: WebSocket echo on ALB
> **Why:** ALB speaks HTTP/1.1 with Upgrade, so WebSockets "just work" — but health checks and idle timeout (60s default) bite. Bump the timeout.

Fargate container for echo (replace in task def):
```typescript
taskDef.addContainer('ws', {
  image: ecs.ContainerImage.fromRegistry('jmalloc/echo-server'),
  portMappings: [{ containerPort: 8080 }],
});
alb.setAttribute('idle_timeout.timeout_seconds', '3600');
```

Test:
```bash
npm i -g wscat
wscat -c ws://<alb-dns>/api/.ws
> hello
< hello
```

## Step 7: HTTP→HTTPS redirect
> **Why:** Shipping with :80 open and no redirect gets flagged in every security review. One listener rule fixes it.

```typescript
alb.addListener('Redirect', {
  port: 80,
  defaultAction: elbv2.ListenerAction.redirect({
    protocol: 'HTTPS', port: '443', permanent: true,
  }),
});
alb.addListener('Https', {
  port: 443,
  certificates: [elbv2.ListenerCertificate.fromArn('arn:aws:acm:...')],
  defaultAction: elbv2.ListenerAction.forward([tgB]),
});
```

`curl -vI http://<alb>/` should return `301 → https://...`.

## Cleanup
> **Why:** Two LBs + Fargate + ASG + NAT = ~$40/mo.

```bash
cdk destroy
```

## Common Errors
- **502 Bad Gateway** from ALB → target returned non-HTTP or died mid-request. Check target logs.
- **All targets unhealthy** → health check path returns 302/404. Must be 200–299. Use `healthyHttpCodes` to broaden.
- **WebSocket disconnects at 60s** → ALB idle timeout. Bump via `idle_timeout.timeout_seconds`.
- **NLB "no healthy targets" but TCP check passes on the host** → security group not allowing NLB's subnet CIDR (NLB uses source IP, not its own).
- **"CertificateNotFound"** on HTTPS listener → cert must be in same region as the ALB and ISSUED status.
