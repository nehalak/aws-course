# Walkthrough — 04 App Runner & Lightsail

## About this service
**App Runner** is AWS's managed container PaaS — closest AWS equivalent to Google Cloud Run or Heroku. Push an image to ECR (or point at a GitHub repo) and App Runner handles build, deploy, HTTPS, custom domains, auto-scaling, and health checks. No VPC/ALB/ECS plumbing to configure. **Lightsail** is a completely separate AWS product with its own console and pricing: fixed-price VPS, databases, and container services aimed at users who find EC2 overwhelming. Think "DigitalOcean inside AWS."

**Why it matters:** both trade flexibility for simplicity. They're how you ship a prototype in an afternoon without becoming a VPC expert. At modest scale (<$500/mo compute) they cost roughly the same as hand-rolled ECS but save weeks of setup.
**When to use App Runner:** stateless HTTP containers, small SaaS backends, internal tools, GitHub-to-URL workflows, teams without a platform engineer.
**When to use Lightsail:** WordPress / LAMP, personal projects, fixed-price predictability, customers migrating from DigitalOcean/Linode.
**When NOT to use either:** deep VPC integration (App Runner has a VPC connector but it's slow to warm), sub-100ms cold-start sensitivity, multi-region active/active, anything Kubernetes-shaped, workloads >$500/mo compute where ECS/EKS savings exceed operational cost.

## Estimated cost
- **App Runner provisioned (idle): $0.007/GB-hr for memory + $0.064/vCPU-hr active** — 1 vCPU / 2 GB always-on minimum: **~$5/month idle + usage**
- **App Runner active request: $0.064/vCPU-hr + $0.007/GB-hr** while serving, billed per second
- **Example**: 1 vCPU / 2 GB, 10% duty cycle 24/7 = ~$10/month
- **Custom domain + ACM**: free
- **VPC connector**: free, but consumes ENIs (~$0.01/hr each if NAT gateway needed, so add ~$7/month)
- **Lightsail instance**: **$3.50/mo** (512 MB / 2 vCPU) to **$160/mo** (32 GB / 8 vCPU), flat
- **Lightsail container service**: **$7/mo** nano to **$160/mo** xlarge, includes LB + HTTPS
- **Lightsail managed DB**: **$15/mo** micro to **$115/mo** large, includes backups
- Total for this lesson: **~$20/month** if you leave App Runner + 1 Lightsail instance running. Destroy after!

---

## Step 1: App Runner from ECR image
> **Why:** The ECR path is the production path — you control the image, CI pushes it, App Runner deploys. The L2 `apprunner-alpha` construct wires up the service, IAM access role (for pulling from ECR), and auto-scaling config. With `desiredCount: 1, maxConcurrency: 100` you pay ~$5/month idle and scale automatically.

`lib/apprunner-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as apprunner from 'aws-cdk-lib/aws-apprunner';
import * as ecr_assets from 'aws-cdk-lib/aws-ecr-assets';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as path from 'path';

export class AppRunnerStack extends cdk.Stack {
  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const image = new ecr_assets.DockerImageAsset(this, 'Img', {
      directory: path.join(__dirname, '../app'),
    });

    const accessRole = new iam.Role(this, 'AccessRole', {
      assumedBy: new iam.ServicePrincipal('build.apprunner.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSAppRunnerServicePolicyForECRAccess'),
      ],
    });

    const svc = new apprunner.CfnService(this, 'Svc', {
      serviceName: 'demo',
      sourceConfiguration: {
        authenticationConfiguration: { accessRoleArn: accessRole.roleArn },
        autoDeploymentsEnabled: true,
        imageRepository: {
          imageIdentifier: image.imageUri,
          imageRepositoryType: 'ECR',
          imageConfiguration: { port: '8080' },
        },
      },
      instanceConfiguration: { cpu: '1 vCPU', memory: '2 GB' },
      healthCheckConfiguration: { path: '/health', protocol: 'HTTP' },
    });

    new apprunner.CfnAutoScalingConfiguration(this, 'Asg', {
      autoScalingConfigurationName: 'demo',
      maxConcurrency: 100, minSize: 1, maxSize: 10,
    });

    new cdk.CfnOutput(this, 'Url', { value: svc.attrServiceUrl });
  }
}
```

```bash
cdk deploy   # 5-8 min
curl https://<service-url>.us-east-1.awsapprunner.com/
```

## Step 2: Attach custom domain
> **Why:** Every real app needs a custom domain, not a random awsapprunner.com URL. App Runner issues an ACM cert automatically (validated via DNS CNAME you add at your registrar). Propagation typically takes 5-15 minutes.

```bash
aws apprunner associate-custom-domain \
  --service-arn <svc-arn> \
  --domain-name api.example.com \
  --enable-www-subdomain

# App Runner returns CertificateValidationRecords — add the CNAMEs at your DNS provider
# Then wait for status 'active':
aws apprunner describe-custom-domains --service-arn <svc-arn>
```

## Step 3: App Runner from GitHub source
> **Why:** Skip the ECR/CI step entirely — push to main, App Runner builds and deploys. Best for prototypes and solo projects. In production, ECR is preferred because you get reproducible builds and a scannable image, but the GitHub path is unbeatable for iteration speed.

`apprunner.yaml` (commit to repo root):
```yaml
version: 1.0
runtime: nodejs18
build:
  commands:
    build:
      - npm ci
      - npm run build
run:
  runtime-version: 18
  command: node dist/server.js
  network:
    port: 8080
    env: PORT
  env:
    - name: NODE_ENV
      value: production
```

Create via console: App Runner → Create service → Source: Source code repository → Connect to GitHub → pick repo + branch. App Runner stores a CodeStar connection; CDK equivalent:
```typescript
sourceConfiguration: {
  authenticationConfiguration: { connectionArn: '<codestar-arn>' },
  autoDeploymentsEnabled: true,
  codeRepository: {
    repositoryUrl: 'https://github.com/you/app',
    sourceCodeVersion: { type: 'BRANCH', value: 'main' },
    codeConfiguration: { configurationSource: 'REPOSITORY' },
  },
},
```

## Step 4: VPC connector to private RDS
> **Why:** By default App Runner services run in AWS-managed VPCs and can only reach the public internet. To hit an RDS instance in a private subnet you attach a VPC connector: App Runner creates ENIs in your subnets and routes egress through them. Expect ~30s extra cold-start per scale-out event the first time.

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';

const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcId: 'vpc-xxx' });
const dbSg = ec2.SecurityGroup.fromSecurityGroupId(this, 'DbSg', 'sg-db');

const connectorSg = new ec2.SecurityGroup(this, 'ConnSg', { vpc });
dbSg.addIngressRule(connectorSg, ec2.Port.tcp(5432), 'app-runner');

const connector = new apprunner.CfnVpcConnector(this, 'Conn', {
  vpcConnectorName: 'demo',
  subnets: vpc.privateSubnets.map((s) => s.subnetId),
  securityGroups: [connectorSg.securityGroupId],
});

// Attach to the service
svc.networkConfiguration = {
  egressConfiguration: {
    egressType: 'VPC',
    vpcConnectorArn: connector.attrVpcConnectorArn,
  },
};
```

Verify from inside the container (use App Runner's `/api/v1/runtime` debug endpoint or CloudWatch logs):
```bash
# in-container check
nc -vz <rds-endpoint> 5432
# Connection to ... succeeded
```

## Step 5: Lightsail instance with WordPress blueprint
> **Why:** Separate console, separate billing, separate mental model — but one command stands up a fully-configured WordPress on Ubuntu with MySQL, Bitnami stack, and SSH. $5/mo flat, no surprise charges. Great for personal blogs; bad if you ever want to attach an RDS or put it behind CloudFront with an ACM cert on the LB.

```bash
aws lightsail create-instances \
  --instance-names wp-blog \
  --availability-zone us-east-1a \
  --blueprint-id wordpress \
  --bundle-id nano_3_0

# Wait for 'running'
aws lightsail get-instance --instance-name wp-blog

# SSH
aws lightsail get-instance-access-details --instance-name wp-blog
# Download .pem then:
ssh -i LightsailDefaultKey-us-east-1.pem bitnami@<public-ip>

# Grab WordPress admin password (printed on first boot)
cat bitnami_application_password
```

Open `http://<public-ip>/wp-admin`.

## Step 6: Lightsail container service vs App Runner
> **Why:** They overlap heavily. Lightsail containers: fixed-price, simpler console, includes LB. App Runner: usage-based, deeper AWS integration, VPC connector, GitHub builds. Writing down the decision forces you to check your real requirements instead of defaulting to the familiar one.

`decision.md`:
```markdown
# Choose App Runner when
- You need to reach resources in a VPC (RDS, ElastiCache, private APIs)
- You want GitHub-triggered builds without CI setup
- Traffic is bursty (scale to 1 saves money vs flat-fee Lightsail)
- You need custom domains with auto-ACM
- You're already using CDK / CloudFormation

# Choose Lightsail Containers when
- Predictable flat pricing matters (budget approvals, personal projects)
- You want everything in one simple console (no IAM deep-dive)
- You don't need VPC integration
- Team lacks AWS experience and will get lost in the main console

# Choose neither when
- >$500/mo compute (ECS saves 30-50%)
- Kubernetes skills already exist (EKS)
- Event-driven (Lambda)
- Windows / GPU workloads (EC2)
```

## Step 7: Document the limitations
> **Why:** Both products are "AWS-lite." Knowing where the abstraction leaks saves you from trying to bolt on features that simply don't exist. Write this down so the next engineer doesn't rediscover it.

`limitations.md`:
```markdown
## App Runner
- No CloudFront in front without an ALB you'd have to run separately (defeats the purpose)
- Max request timeout: 120s (no WebSockets-with-state, no long uploads)
- No sidecar containers (single container per service)
- VPC connector cold-start adds ~30s on scale-out
- Min cost ~$5/mo even idle; scale-to-zero not available
- No Spot-equivalent; pricing is flat

## Lightsail
- Different console, different APIs, different IAM (awslightsail.* actions)
- No CloudWatch integration for instances (limited built-in metrics only)
- Cannot attach an EBS volume from EC2 to a Lightsail instance
- No VPC peering except via the "Lightsail VPC peering" toggle (one per region, limited)
- Snapshots are Lightsail-only; can't restore into EC2 without export
- Managed DBs max out at 8 GB RAM; RDS is the answer above that
```

## Cleanup
```bash
# App Runner (CDK-managed)
cdk destroy

# If custom domain still attached, disassociate first:
aws apprunner disassociate-custom-domain \
  --service-arn <svc-arn> --domain-name api.example.com

# Lightsail (not in CDK — delete explicitly)
aws lightsail delete-instance --instance-name wp-blog
aws lightsail delete-container-service --service-name demo    # if used
aws lightsail delete-relational-database --relational-database-name demo  # if used
```

## Common Errors
- **App Runner service stuck in `OPERATION_IN_PROGRESS` for 20+ min** → health check path returns non-2xx. Check `/health` actually exists and listens on the configured port.
- **`Unable to pull image from ECR`** → access role missing `AWSAppRunnerServicePolicyForECRAccess`, or the ECR repo is in a different account without a resource policy.
- **Custom domain stuck `pending_certificate_dns_validation`** → CNAME not added at registrar, or TTL too long. Use Route 53 with the console's "create records" button.
- **VPC connector created but DB connection times out** → connector security group not added to DB SG ingress rules. App Runner ENI IPs aren't fixed; allow by SG, not CIDR.
- **App Runner costs exploding** → autoscaling misconfigured with `maxConcurrency: 1` spawning instances per request. Set `maxConcurrency: 50-100`.
- **Lightsail instance public IP changed after stop/start** → expected. Attach a static IP (free while attached, $0.005/hr when unattached).
- **Lightsail "cannot delete: attached to static IP"** → detach static IP first, then delete instance, then release IP.
- **Lightsail VPC peering broken** → you can only peer the Lightsail VPC to a single AWS VPC per region, and it must be in the same region. Hit the toggle in console once.
- **App Runner GitHub connector stuck "pending handshake"** → open the CodeStar connection in the AWS console and click "Update pending connection" to finish GitHub OAuth.
