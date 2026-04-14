# Walkthrough — 04 PrivateLink

## About this service
**AWS PrivateLink** lets a **provider** expose a service (fronted by a Network Load Balancer or Gateway Load Balancer) to **consumers** in other VPCs or accounts *without* VPC peering, Transit Gateway attachments, or any public internet exposure. Consumers create an **Interface VPC Endpoint** that materializes as ENIs in their own subnets with private IPs from their own CIDR — traffic flows over the AWS backbone and never touches the public internet.

**Why it matters:** peering doesn't scale (N² connections, CIDR overlap pain, full bidirectional reachability by default). PrivateLink gives you a **one-way, service-scoped** door: the consumer can reach *only* the specific service, not arbitrary hosts. This is how AWS itself exposes S3/DynamoDB/STS inside VPCs, how SaaS vendors (Snowflake, Datadog, MongoDB Atlas) deliver their service privately, and how you expose internal platforms to sibling accounts without a shared network.

**When to use:** exposing an internal API to many consumer VPCs/accounts; consuming a SaaS without public egress; replacing a VPC peering mesh; serving customers from a single provider VPC with overlapping CIDRs.
**When NOT to use:** bidirectional connectivity (consumer needs to be reachable from provider — PrivateLink is one-way); UDP traffic (NLB UDP works but many PrivateLink-behind-NLB nuances); tiny scale where a single VPC peering is simpler.

## Estimated cost
- **VPC Endpoint Service (provider side): free** (you pay for the NLB)
- **NLB: ~$16/month** + $0.006 per NLCU-hour (trivial for lab traffic)
- **Interface VPC Endpoint (consumer side): $0.01/hour per AZ ENI ≈ $7.20/month per AZ**
- **Data processing: $0.01/GB in each direction** through the endpoint
- Fargate task for the provider service (0.25 vCPU, 0.5 GB): **~$9/month**
- Total for this lesson (NLB + 2 AZ endpoint + small Fargate): **~$40/month**

---

## Step 1: Scaffold provider + consumer stacks
> **Why:** Provider and consumer are logically separate — in real life they're in different accounts. Keeping them as different CDK stacks from the start mirrors that reality and lets you tear them down independently.

```bash
mkdir privatelink-lab && cd privatelink-lab
cdk init app --language=typescript
```

`bin/app.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { ProviderStack } from '../lib/provider-stack';
import { ConsumerStack } from '../lib/consumer-stack';

const app = new cdk.App();
const env = { account: process.env.CDK_DEFAULT_ACCOUNT!, region: 'us-east-1' };

const provider = new ProviderStack(app, 'ProviderStack', { env });
new ConsumerStack(app, 'ConsumerStack', {
  env,
  endpointServiceName: provider.endpointServiceName,
});
```

## Step 2: Provider — VPC, Fargate service, internal NLB
> **Why:** PrivateLink endpoint services require an **NLB** (layer 4) or **GWLB** (layer 3, for transparent inline appliances). ALBs don't work — if your service is HTTP only, put an internal NLB in front of your ALB, or use an NLB with TLS listeners. Fargate on the backend is simplest for a lab.

`lib/provider-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';

export class ProviderStack extends cdk.Stack {
  public readonly endpointServiceName: string;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'ProviderVpc', {
      ipAddresses: ec2.IpAddresses.cidr('10.20.0.0/16'),
      maxAzs: 2,
      natGateways: 1,
    });

    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    const taskDef = new ecs.FargateTaskDefinition(this, 'Task', {
      cpu: 256,
      memoryLimitMiB: 512,
    });
    taskDef.addContainer('web', {
      image: ecs.ContainerImage.fromRegistry('public.ecr.aws/nginx/nginx:latest'),
      portMappings: [{ containerPort: 80 }],
      logging: ecs.LogDriver.awsLogs({ streamPrefix: 'provider' }),
    });

    const svc = new ecs.FargateService(this, 'Svc', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 2,
    });

    // Internal NLB — NOT internet-facing
    const nlb = new elbv2.NetworkLoadBalancer(this, 'Nlb', {
      vpc,
      internetFacing: false,
      crossZoneEnabled: true,
    });
    const listener = nlb.addListener('L80', { port: 80 });
    listener.addTargets('t', {
      port: 80,
      targets: [svc],
      healthCheck: { protocol: elbv2.Protocol.HTTP, path: '/' },
    });

    // Allow NLB health checks and traffic into Fargate tasks
    svc.connections.allowFrom(ec2.Peer.ipv4(vpc.vpcCidrBlock), ec2.Port.tcp(80));

    // VPC Endpoint Service — this is PrivateLink
    const endpointSvc = new ec2.VpcEndpointService(this, 'EndpointSvc', {
      vpcEndpointServiceLoadBalancers: [nlb],
      acceptanceRequired: true,                 // manual approval of consumers
      allowedPrincipals: [
        new cdk.aws_iam.ArnPrincipal(`arn:aws:iam::${this.account}:root`),
      ],
    });

    this.endpointServiceName = endpointSvc.vpcEndpointServiceName;
    new cdk.CfnOutput(this, 'ServiceName', { value: endpointSvc.vpcEndpointServiceName });
  }
}
```

Deploy:
```bash
cdk deploy ProviderStack
```

Expected output:
```
ProviderStack.ServiceName = com.amazonaws.vpce.us-east-1.vpce-svc-0abcd1234ef567890
```

## Step 3: Consumer — separate VPC + Interface Endpoint
> **Why:** The consumer creates an **Interface VPC Endpoint** pointing at the provider's service name. This drops ENIs into the consumer's own subnets. To the consumer's EC2s, the provider appears as a regular private-IP host in their VPC.

`lib/consumer-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export interface ConsumerProps extends cdk.StackProps {
  endpointServiceName: string;
}

export class ConsumerStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: ConsumerProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'ConsumerVpc', {
      ipAddresses: ec2.IpAddresses.cidr('10.30.0.0/16'),
      maxAzs: 2,
      natGateways: 1,
    });

    const endpointSg = new ec2.SecurityGroup(this, 'EndpointSg', { vpc });
    endpointSg.addIngressRule(ec2.Peer.ipv4(vpc.vpcCidrBlock), ec2.Port.tcp(80));

    const endpoint = new ec2.InterfaceVpcEndpoint(this, 'ProviderEndpoint', {
      vpc,
      service: new ec2.InterfaceVpcEndpointService(props.endpointServiceName, 80),
      privateDnsEnabled: false,   // custom services can't use AWS-provided private DNS without domain verification
      subnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      securityGroups: [endpointSg],
    });

    // Test EC2 in the consumer VPC
    const testHost = new ec2.Instance(this, 'TestEc2', {
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      instanceType: new ec2.InstanceType('t3.micro'),
      machineImage: ec2.MachineImage.latestAmazonLinux2023(),
      ssmSessionPermissions: true,
    });

    new cdk.CfnOutput(this, 'EndpointDns', {
      value: cdk.Fn.select(0, endpoint.vpcEndpointDnsEntries),
      description: 'Use the hosted-zone-Name part as the target hostname',
    });
    new cdk.CfnOutput(this, 'TestHostId', { value: testHost.instanceId });
  }
}
```

Deploy:
```bash
cdk deploy ConsumerStack
```

## Step 4: Accept the endpoint connection (provider side)
> **Why:** Because the provider set `acceptanceRequired: true`, each new consumer endpoint lands in a **pendingAcceptance** state. The provider approves per consumer — the whitelist model.

```bash
SERVICE_ID=$(aws ec2 describe-vpc-endpoint-services \
  --query "ServiceDetails[?ServiceName=='com.amazonaws.vpce.us-east-1.vpce-svc-0abcd1234ef567890'].ServiceId" \
  --output text)

# From provider context
aws ec2 describe-vpc-endpoint-connections \
  --filters "Name=service-id,Values=$SERVICE_ID"

# Accept
aws ec2 accept-vpc-endpoint-connections \
  --service-id "$SERVICE_ID" \
  --vpc-endpoint-ids vpce-0abc123def456
```

Expected:
```json
{ "Unsuccessful": [] }
```

## Step 5: Test from the consumer
> **Why:** Only traffic actually flowing proves the endpoint works. DNS setup is the most common stumbling block — note that the consumer must hit the endpoint-specific DNS name, not the provider's NLB name.

SSM into the consumer test host:
```bash
aws ssm start-session --target <TestHostId>
```

Inside the session:
```bash
# Get the endpoint DNS (from CDK output)
ENDPOINT_DNS="vpce-0abc-xyz.vpce-svc-0def.us-east-1.vpce.amazonaws.com"
curl -v http://$ENDPOINT_DNS/
```

Expected:
```
< HTTP/1.1 200 OK
< Server: nginx/1.25.3
...
<h1>Welcome to nginx!</h1>
```

Then verify the packet path never leaves AWS:
```bash
dig +short $ENDPOINT_DNS
# Returns RFC 1918 IPs from 10.30.0.0/16 — the consumer's own CIDR!
traceroute $ENDPOINT_DNS
# At most 1-2 hops, all internal AWS — no public gateway
```

## Step 6: Cross-account (paper design if only one account)
> **Why:** The real value of PrivateLink is cross-account SaaS delivery. The setup is identical except the `allowedPrincipals` on the provider side points to the consumer account's ARN, and the consumer creates the endpoint from their own account using the provider's service name.

Provider side change:
```typescript
allowedPrincipals: [
  new cdk.aws_iam.ArnPrincipal('arn:aws:iam::222222222222:root'),   // consumer account
],
```

Consumer (account `222222222222`) runs the exact same `ConsumerStack` with the service name string they were given. The connection shows up on the provider side as `pendingAcceptance`; the provider accepts it; traffic flows.

## Step 7: Custom Private DNS for a branded service name
> **Why:** You don't want consumers to hit `vpce-0abc-xyz.vpce-svc-0def...`. For SaaS, you brand it: `api.acme-saas.com`. Requires domain ownership verification and a Route 53 **Private Hosted Zone** in the consumer VPC (or the consumer's own DNS infra).

Provider side — verify domain:
```bash
aws ec2 modify-vpc-endpoint-service-configuration \
  --service-id "$SERVICE_ID" \
  --private-dns-name api.acme-saas.com

aws ec2 start-vpc-endpoint-service-private-dns-verification \
  --service-id "$SERVICE_ID"

aws ec2 describe-vpc-endpoint-service-configurations \
  --service-ids "$SERVICE_ID" \
  --query 'ServiceConfigurations[0].PrivateDnsNameConfiguration'
```

AWS gives you a TXT record to place in the public DNS of `acme-saas.com`. Once verified, consumers can set `privateDnsEnabled: true` on their interface endpoint and `api.acme-saas.com` resolves to the endpoint ENIs automatically from inside their VPC.

## Step 8: Architect SaaS for 50 customers (paper exercise)
> **Why:** PrivateLink's killer use case is multi-tenant SaaS. Understanding the scaling properties — one endpoint service, N customer endpoints, whitelist-controlled — is what turns "I ran a demo" into "I can design this for a real company".

Narrative design (write as `saas-design.md`):

- **Provider side:** one VPC per region, NLB in front of horizontally-scaled Fargate, single VPC Endpoint Service with `acceptanceRequired: true`.
- **Tenancy:** no per-customer VPC. Requests arrive on the NLB and carry a client TLS cert or a JWT identifying the tenant. App does tenant routing internally.
- **Onboarding flow:**
  1. Customer provides their 12-digit AWS account ID.
  2. Ops adds it to `allowedPrincipals` via IaC PR.
  3. Customer deploys the published CDK/Terraform module that creates their Interface Endpoint and the private-DNS override.
  4. Customer initiates connection; ops auto-accepts via a Lambda watching `EC2 CreateVpcEndpoint` CloudTrail events (filtering on their whitelisted accounts).
  5. Health check from customer endpoint → 200 OK.
- **Scale:** NLB supports millions of flows/sec. PrivateLink has a soft limit of 50 VPCs per endpoint service — request an increase early (support ticket, typically approved to low thousands).
- **Observability:** VPC Flow Logs on the endpoint ENIs (consumer side, visible to customer), CloudWatch NLB metrics (provider side). Per-tenant request metrics via app-level instrumentation keyed on the identifying claim.
- **Pricing to customer:** flat monthly + per-GB of PrivateLink data processing (passed through at cost + margin).

## Cleanup
> **Why:** Interface endpoints are $7.20/month per AZ and quietly accumulate when you spin up lab after lab.

```bash
cdk destroy ConsumerStack
cdk destroy ProviderStack
```

If you added private DNS verification, also remove the TXT record from your public DNS zone.

## Common Errors
- **`InvalidServiceName` when creating the consumer endpoint** → the service name is region-specific and case-sensitive. Copy it from the provider output verbatim.
- **Endpoint stuck in `pendingAcceptance` forever** → provider's `acceptanceRequired: true` but nobody clicked accept. Check `describe-vpc-endpoint-connections` on the provider side.
- **`curl` hangs / times out** → security group on the endpoint ENIs doesn't allow the consumer EC2, or the NLB has no healthy targets on the provider side. Check NLB target-group health in the provider account.
- **DNS resolves to public IP instead of endpoint IP** → `privateDnsEnabled` is false and you're using the wrong DNS name. Either enable private DNS (requires verified domain) or use the endpoint-specific `vpce-*.vpce.amazonaws.com` name.
- **Works from AZ-a, fails from AZ-b** → you only put the endpoint in one subnet. Add all AZ subnets to the endpoint.
- **Provider charged but no traffic flowing** → the NLB is idle and you're paying ~$16/month anyway. NLB has no pause-on-idle mode.
- **PrivateLink doesn't support UDP reliably** → for UDP, use NLB with UDP listener; TCP is the well-trodden path.
