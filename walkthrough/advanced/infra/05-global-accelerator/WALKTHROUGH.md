# Walkthrough — 05 Global Accelerator

## About this service
**AWS Global Accelerator (GA)** gives you **two static anycast IPv4 addresses** (plus IPv6) advertised from AWS's global edge network. Clients connect to those IPs and AWS routes them over its backbone to the nearest *healthy* regional endpoint (ALB/NLB/EC2/EIP) you've registered. It's a layer-4 (TCP/UDP) service — no caching, no content manipulation — which makes it fundamentally different from CloudFront, even though they share the same edge footprint.

**Why it matters:** two things. **(1) Deterministic failover:** health-check-driven traffic shifting happens at the edge in **under 30 seconds**, bypassing DNS TTL entirely. Clients using DNS-based failover (Route 53) can cache records for minutes. **(2) Performance:** traffic enters AWS at the edge nearest the client, then rides the AWS backbone instead of the public internet — typically 10-60% lower latency and less jitter for TCP/UDP workloads.

**When to use:** global gaming/voice/video (UDP), financial TCP APIs, any regional failover where DNS caching is unacceptable, apps with static IP allowlist requirements from customer firewalls.
**When NOT to use:** HTTP-heavy public web where you'd benefit from caching (use CloudFront); purely single-region apps (GA has a fixed $18/month charge even if only one endpoint); cost-sensitive small traffic (Route 53 latency-based routing is free).

## Estimated cost
- **Fixed accelerator fee: $0.025/hour ≈ $18/month**
- **Premium data transfer (DT-Premium): $0.015/GB** (US/EU) on top of normal egress, billed for traffic that actually rides the AWS backbone
- Two ALBs (one us-east-1, one eu-west-1): **~$32/month**
- Route 53 hosted zone if aliasing a custom domain: **$0.50/month**
- Total for this lesson: **~$55/month** plus minor data transfer. Destroy the accelerator when done.
- Compare: CloudFront is pay-per-GB with zero fixed cost, but no fixed IPs and no UDP.

---

## Step 1: Scaffold + two regional ALB stacks
> **Why:** GA's superpower is cross-region failover, so you need at least two regions with an ALB each. Reusing the pattern from lesson 02 keeps this tight — the only new construct is the accelerator itself.

```bash
mkdir global-accel && cd global-accel
cdk init app --language=typescript
npm install aws-cdk-lib
```

`bin/app.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { RegionalAlbStack } from '../lib/regional-alb';
import { AcceleratorStack } from '../lib/accelerator';

const app = new cdk.App();
const account = process.env.CDK_DEFAULT_ACCOUNT!;

const us = new RegionalAlbStack(app, 'Alb-USE1', {
  env: { account, region: 'us-east-1' }, label: 'us-east-1',
});
const eu = new RegionalAlbStack(app, 'Alb-EUW1', {
  env: { account, region: 'eu-west-1' }, label: 'eu-west-1',
});

new AcceleratorStack(app, 'Ga', {
  // GA is a global service — its CloudFormation stack must be in us-west-2
  env: { account, region: 'us-west-2' },
  usAlbArn: us.albArn,
  euAlbArn: eu.albArn,
  crossRegionReferences: true,
});
```

## Step 2: Regional ALB stack with a region-identifying response
> **Why:** To *prove* which region you hit, each ALB serves a page that says its region name. Without that you can't tell whether GA routed you correctly versus Route 53 round-robin.

`lib/regional-alb.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecsp from 'aws-cdk-lib/aws-ecs-patterns';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';

export interface RegionalProps extends cdk.StackProps { label: string; }

export class RegionalAlbStack extends cdk.Stack {
  public readonly albArn: string;

  constructor(scope: Construct, id: string, props: RegionalProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });
    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    const svc = new ecsp.ApplicationLoadBalancedFargateService(this, 'Svc', {
      cluster,
      cpu: 256,
      desiredCount: 1,
      taskImageOptions: {
        image: ecs.ContainerImage.fromRegistry('hashicorp/http-echo'),
        containerPort: 5678,
        command: ['-text', `hello from ${props.label}`, '-listen', ':5678'],
      },
      publicLoadBalancer: true,
      listenerPort: 80,
    });
    svc.targetGroup.configureHealthCheck({
      path: '/',
      healthyHttpCodes: '200-399',
      interval: cdk.Duration.seconds(10),
      healthyThresholdCount: 2,
      unhealthyThresholdCount: 2,
    });

    this.albArn = svc.loadBalancer.loadBalancerArn;
    new cdk.CfnOutput(this, 'AlbDns', { value: svc.loadBalancer.loadBalancerDnsName });
    new cdk.CfnOutput(this, 'AlbArn', { value: svc.loadBalancer.loadBalancerArn });
  }
}
```

Deploy both regions first:
```bash
cdk deploy Alb-USE1 Alb-EUW1
```

## Step 3: Accelerator with two endpoint groups
> **Why:** A GA **accelerator** has **listeners** (port/protocol), each listener has **endpoint groups** (one per region), each endpoint group has **endpoints** (ALB/NLB/EC2/EIP). This hierarchy is how GA knows which regional resources are failover candidates for each other.

`lib/accelerator.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ga from 'aws-cdk-lib/aws-globalaccelerator';
import * as ga_endpoints from 'aws-cdk-lib/aws-globalaccelerator-endpoints';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';

export interface GaProps extends cdk.StackProps {
  usAlbArn: string;
  euAlbArn: string;
}

export class AcceleratorStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: GaProps) {
    super(scope, id, props);

    const accel = new ga.Accelerator(this, 'Accel', {
      acceleratorName: 'learn-ga',
      ipAddressType: ga.IpAddressType.IPV4,
    });

    const listener = accel.addListener('Http', {
      portRanges: [{ fromPort: 80, toPort: 80 }],
      clientAffinity: ga.ClientAffinity.NONE,   // flip to SOURCE_IP for sticky
    });

    const usAlb = elbv2.ApplicationLoadBalancer.fromLookup(this, 'UsAlb', {
      loadBalancerArn: props.usAlbArn,
    });
    const euAlb = elbv2.ApplicationLoadBalancer.fromLookup(this, 'EuAlb', {
      loadBalancerArn: props.euAlbArn,
    });

    listener.addEndpointGroup('UsGroup', {
      region: 'us-east-1',
      endpoints: [
        new ga_endpoints.ApplicationLoadBalancerEndpoint(usAlb, { weight: 128 }),
      ],
      trafficDialPercentage: 100,                // full traffic eligible
      healthCheckPath: '/',
      healthCheckInterval: cdk.Duration.seconds(10),
      healthyThresholdCount: 2,
      unhealthyThresholdCount: 2,
    });

    listener.addEndpointGroup('EuGroup', {
      region: 'eu-west-1',
      endpoints: [
        new ga_endpoints.ApplicationLoadBalancerEndpoint(euAlb, { weight: 128 }),
      ],
      trafficDialPercentage: 100,
      healthCheckPath: '/',
    });

    new cdk.CfnOutput(this, 'AnycastDns', { value: accel.dnsName });
    new cdk.CfnOutput(this, 'StaticIps', { value: cdk.Fn.join(',', accel.ipAddresses) });
  }
}
```

Deploy:
```bash
cdk deploy Ga
```

Expected output:
```
Ga.AnycastDns = a1b2c3d4e5f6a7b8.awsglobalaccelerator.com
Ga.StaticIps  = 75.2.x.x,99.83.y.y
```

## Step 4: Observe anycast routing
> **Why:** The magic of anycast is that the *same* IP serves from every edge, and BGP routes your packets to the nearest PoP. The test is to resolve from different geographies and confirm the IP is identical, yet the backend hit is different.

From a US machine:
```bash
dig +short a1b2c3d4e5f6a7b8.awsglobalaccelerator.com
# 75.2.x.x  99.83.y.y  <-- same two IPs everywhere in the world

curl a1b2c3d4e5f6a7b8.awsglobalaccelerator.com
# hello from us-east-1
```

From an EU machine (or an EC2 in eu-west-1):
```bash
dig +short a1b2c3d4e5f6a7b8.awsglobalaccelerator.com
# 75.2.x.x  99.83.y.y  <-- SAME IPs, proving anycast

curl a1b2c3d4e5f6a7b8.awsglobalaccelerator.com
# hello from eu-west-1  <-- but backend differs
```

## Step 5: Failover test
> **Why:** This is the reason to pay $18/month for GA over Route 53 latency routing. You want to see the sub-30-second flip with your own eyes.

Break the us-east-1 endpoint deliberately by changing its health check to hit a path that doesn't exist:
```bash
aws elbv2 modify-target-group \
  --region us-east-1 \
  --target-group-arn <us-target-group-arn> \
  --health-check-path /does-not-exist
```

From a US machine, run a loop:
```bash
while true; do
  curl -s a1b2c3d4e5f6a7b8.awsglobalaccelerator.com
  sleep 2
done
```

Expected timeline:
- **t+0s to t+~20s:** `hello from us-east-1`
- **t+~20-30s:** GA marks us-east-1 unhealthy (2 failed checks × 10s interval)
- **t+~30s onward:** `hello from eu-west-1` — no DNS change, same IP
- Compare with Route 53 failover (lesson 02): there you waited 90-150s *plus* client DNS TTL.

Restore:
```bash
aws elbv2 modify-target-group \
  --region us-east-1 \
  --target-group-arn <us-target-group-arn> \
  --health-check-path /
```

## Step 6: Client affinity (sticky sessions)
> **Why:** Default `NONE` lets GA spread repeat requests across endpoint groups. For stateful apps (shopping carts on instance-local sessions, WebSockets) you want **SOURCE_IP** affinity so the same client lands on the same region for the duration of their flow.

Flip in the listener:
```typescript
clientAffinity: ga.ClientAffinity.SOURCE_IP,
```

Redeploy and verify from a single client:
```bash
for i in {1..20}; do curl -s a1b2c3d4e5f6a7b8.awsglobalaccelerator.com; done
# Should return the SAME region 20 times in a row — same 5-tuple → same endpoint
```

## Step 7: Traffic dial for gradual cutover / blue-green
> **Why:** `trafficDialPercentage` on an endpoint group is a regional percentage knob independent of health. Setting it to 0 drains the group without marking it unhealthy — perfect for maintenance windows or blue-green regional migrations.

```bash
aws globalaccelerator update-endpoint-group \
  --region us-west-2 \
  --endpoint-group-arn <us-group-arn> \
  --traffic-dial-percentage 0
```

All traffic now rides eu-west-1 even though us-east-1 is healthy. Useful for rehearsing regional evacuations.

## Step 8: Custom routing (paper design for gaming)
> **Why:** Standard accelerators route based on health + proximity. **Custom Routing Accelerators** deterministically map specific client IP/port tuples to specific EC2 instance/port destinations — mandatory for matchmaking architectures (200 players per match on a specific server instance).

Architecture narrative for `custom-routing.md`:
- Create a Custom Routing Accelerator.
- Provision a fleet of dedicated match-server EC2s in a VPC, each listening on ports 10000-10100.
- Matchmaker service allocates a free `(instance, port)` tuple per match and returns the tuple's GA anycast IP + port to clients.
- GA rewrites `(clientIP:clientPort)` → exact `(instanceIP:instancePort)` — no load balancing, no health-based rerouting; completely deterministic.
- Security: port mappings enforce that only matched clients reach the match server.

## Step 9: Compare to CloudFront
> **Why:** Interviews and architecture reviews will force you to defend GA vs CloudFront. Knowing the axes cold is the difference between "uses edge network" (wrong) and a real answer.

Write `ga-vs-cf.md` with this comparison:

| Axis                   | Global Accelerator                     | CloudFront                               |
|------------------------|-----------------------------------------|-------------------------------------------|
| Layer                  | L4 (TCP/UDP, any port)                  | L7 (HTTP/HTTPS + WebSockets only)         |
| Caching                | None — pure pass-through                | Yes — that's the primary value            |
| Failover speed         | ~30s (edge health checks)               | Origin-failover per request (stateless)   |
| Static IPs             | Yes (2 IPv4, 2 IPv6, allowlist-friendly)| No — dynamic edge IPs                     |
| Fixed cost             | $18/month                               | $0 base                                   |
| Data cost              | Premium DT $0.015/GB + egress           | Tiered, typically cheaper at volume       |
| Custom origins         | ALB/NLB/EC2/EIP                         | Any HTTP origin, incl. S3, MediaStore     |
| Cache invalidation     | N/A                                     | `CreateInvalidation` API                  |
| TLS termination        | Pass-through to your LB                 | Terminates at edge (ACM certs)            |
| WAF                    | Not integrated                          | AWS WAF integrates directly               |
| Use case fit           | gRPC, gaming, VoIP, SFTP, IP allowlists | Websites, APIs, videos, anything cacheable|

Rule of thumb: **HTTP + cacheable → CloudFront.** **TCP/UDP, static IPs required, or failover speed-critical → GA.** Many real architectures use **both**: CloudFront in front for HTTP caching, GA fronting NLBs for WebSocket/gRPC backends.

## Cleanup
> **Why:** $18/month fixed fee keeps ticking whether you use it or not. Accelerators don't pause.

```bash
cdk destroy Ga
cdk destroy Alb-EUW1 Alb-USE1
```

Double-check nothing left:
```bash
aws globalaccelerator list-accelerators --region us-west-2
```

## Common Errors
- **`ValidationException: Region us-east-1 is not supported`** → the accelerator *stack* must deploy to `us-west-2`; the endpoints can live anywhere.
- **Endpoint group stuck `INITIAL`** → ALB health check path returns non-2xx. GA uses ALB-level health, and also does its own probe to the listener.
- **Failover doesn't happen** → both endpoint groups are in the same region, or `trafficDialPercentage` is non-zero on the broken one. GA only fails over if the target group has zero healthy hosts *and* the dial is 0.
- **Same region always wins despite healthy other** → `clientAffinity: SOURCE_IP` plus you're testing from a single IP. Use `NONE` to verify cross-region behavior.
- **`InvalidArgument: Endpoint not found`** → the ALB's region doesn't match the endpoint group's `region`. The endpoint group's region must be the ALB's region.
- **Costs higher than expected** → DT-Premium applies on all bytes egressing the AWS backbone. Disabling idle endpoint groups does *not* reduce the $18/month base fee; you must delete the accelerator.
- **Static IPs change after delete/recreate** → yes, they're allocated from AWS's anycast pool and are not portable. Use **BYOIP** (bring-your-own IP) if you need to keep the same IP across recreates.
