# Walkthrough — 03 EKS Intro

## About this service
**Amazon EKS** is a managed Kubernetes control plane. You pay a flat $0.10/hr for the control plane; you bring the data plane (EC2 managed node groups, Fargate profiles, or Karpenter-provisioned nodes). Integrations with AWS: **IRSA** (pod → IAM role without node role pollution), **AWS Load Balancer Controller** (Ingress → ALB), **EBS CSI driver** (PVCs → EBS volumes).

**Why it matters:** Teams picking EKS over ECS usually want the k8s ecosystem (Helm, operators, Argo, Istio) or multi-cloud portability. It's more moving parts but unlocks a huge open-source world.

**When to use:** need Helm charts/operators, polyglot platform team, many services with complex networking, multi-cloud/on-prem portability.
**When NOT to use:** 1-5 services (ECS is simpler and cheaper), team has no k8s expertise, don't want to pay $73/mo control plane for lab workloads.

## Estimated cost
- **EKS control plane: $0.10/hr = ~$73/month** (flat, per cluster)
- **Managed node group: 2 × t3.medium on-demand = ~$60/month** ($0.0416/hr each)
- **NLB (from Service type=LoadBalancer): $16/month** + LCU
- **ALB (from Ingress): $16/month** + LCU
- **EBS gp3: $0.08/GB-month** → 20GB PVC = $1.60/month
- **NAT Gateway: $32/month**
- Total for this lesson: **~$200/month** — **destroy immediately after**. The $73 control plane hurts even idle.

---

## Step 1: Project setup
> **Why:** `@aws-cdk/lambda-layer-kubectl-v29` gives kubectl inside the CDK custom resource that manages Kubernetes manifests. Match its version to your cluster version.

```bash
mkdir eks-intro && cd eks-intro
cdk init app --language=typescript
npm install aws-cdk-lib constructs @aws-cdk/lambda-layer-kubectl-v29
```

## Step 2: EKS cluster with managed node group
> **Why:** `authenticationMode: API_AND_CONFIG_MAP` supports newer access entries API (safer than `aws-auth` ConfigMap lockouts). `defaultCapacity: 0` lets you add node groups explicitly with your own sizing.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as eks from 'aws-cdk-lib/aws-eks';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import { KubectlV29Layer } from '@aws-cdk/lambda-layer-kubectl-v29';

export class EksIntroStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });
    const adminRole = new iam.Role(this, 'Admin', {
      assumedBy: new iam.AccountRootPrincipal(),
    });

    const cluster = new eks.Cluster(this, 'Cluster', {
      vpc,
      version: eks.KubernetesVersion.V1_29,
      kubectlLayer: new KubectlV29Layer(this, 'kl'),
      mastersRole: adminRole,
      authenticationMode: eks.AuthenticationMode.API_AND_CONFIG_MAP,
      defaultCapacity: 0,
    });

    cluster.addNodegroupCapacity('NG', {
      instanceTypes: [new ec2.InstanceType('t3.medium')],
      minSize: 2, maxSize: 4, desiredSize: 2,
    });

    new cdk.CfnOutput(this, 'KubeConfigCmd', {
      value: `aws eks update-kubeconfig --name ${cluster.clusterName} --region ${this.region} --role-arn ${adminRole.roleArn}`,
    });
  }
}
```

```bash
cdk deploy
# takes ~15 min
aws eks update-kubeconfig --name <name> --region us-east-1 --role-arn <admin-role>
kubectl get nodes
# NAME                           STATUS   ROLES    AGE   VERSION
# ip-10-0-1-23.ec2.internal      Ready    <none>   3m    v1.29.0-eks-xxxxx
# ip-10-0-2-45.ec2.internal      Ready    <none>   3m    v1.29.0-eks-xxxxx
```

## Step 3: Deploy nginx with Service type LoadBalancer
> **Why:** By default, Service type `LoadBalancer` on EKS creates an NLB (classic ELB in old clusters). This is the fastest path to a public endpoint.

```typescript
cluster.addManifest('Nginx', {
  apiVersion: 'apps/v1', kind: 'Deployment',
  metadata: { name: 'nginx' },
  spec: {
    replicas: 2,
    selector: { matchLabels: { app: 'nginx' } },
    template: {
      metadata: { labels: { app: 'nginx' } },
      spec: { containers: [{ name: 'nginx', image: 'nginx:1.27', ports: [{ containerPort: 80 }] }] },
    },
  },
}, {
  apiVersion: 'v1', kind: 'Service',
  metadata: { name: 'nginx' },
  spec: {
    type: 'LoadBalancer',
    ports: [{ port: 80, targetPort: 80 }],
    selector: { app: 'nginx' },
  },
});
```

```bash
kubectl get svc nginx
# NAME    TYPE           EXTERNAL-IP                                          PORT(S)
# nginx   LoadBalancer   a1b2c3...elb.us-east-1.amazonaws.com                 80:32145/TCP
curl http://a1b2c3....elb.us-east-1.amazonaws.com
# <!DOCTYPE html>...Welcome to nginx!...
```

## Step 4: IRSA — pod with S3 access
> **Why:** IAM Roles for Service Accounts maps a k8s ServiceAccount to an IAM role via OIDC. The pod gets temporary creds without you touching the node role (which every pod would share).

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

const bucket = new s3.Bucket(this, 'DataBucket');

const sa = cluster.addServiceAccount('S3Reader', { name: 's3-reader', namespace: 'default' });
bucket.grantRead(sa);

cluster.addManifest('S3Pod', {
  apiVersion: 'v1', kind: 'Pod',
  metadata: { name: 's3-lister', namespace: 'default' },
  spec: {
    serviceAccountName: 's3-reader',
    containers: [{
      name: 'cli',
      image: 'amazon/aws-cli:2.15.0',
      command: ['sh', '-c', 'aws s3 ls && sleep 3600'],
    }],
  },
});
```

```bash
kubectl logs s3-lister
# 2026-04-14 10:00:00 eksintrostack-databucket123abc
```

No `aws configure`, no keys — pure OIDC.

## Step 5: AWS Load Balancer Controller + Ingress → ALB
> **Why:** The controller watches `Ingress` resources and creates ALBs. Gives you path-based routing, WAF integration, SSL termination — none of which NLB/Service type=LB provides.

```typescript
cluster.addHelmChart('LbController', {
  chart: 'aws-load-balancer-controller',
  repository: 'https://aws.github.io/eks-charts',
  namespace: 'kube-system',
  release: 'aws-lbc',
  values: {
    clusterName: cluster.clusterName,
    serviceAccount: { create: true, name: 'aws-load-balancer-controller' },
  },
});
```

Then apply an Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: nginx, port: { number: 80 } } }
```

## Step 6: EBS CSI driver + PVC
> **Why:** Stateful pods (Postgres, Redis single-instance) need persistent disks. EBS CSI driver provisions `gp3` volumes dynamically from PVCs.

```typescript
new eks.CfnAddon(this, 'EbsCsi', {
  clusterName: cluster.clusterName,
  addonName: 'aws-ebs-csi-driver',
});
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources: { requests: { storage: 20Gi } }
```

## Step 7: Karpenter (optional, advanced)
> **Why:** Karpenter replaces managed node groups with just-in-time, right-sized node provisioning. Deploy 10 × 1Gi pod → Karpenter picks cheapest instance that fits all 10. Much better bin-packing than static node groups.

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version 1.0.6 --namespace karpenter --create-namespace \
  --set "settings.clusterName=<cluster>" \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=<karpenter-role>"

kubectl apply -f - <<EOF
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: { name: default }
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c","m","r"]
EOF
```

## Cleanup
```bash
# Delete ALBs/NLBs FIRST or VPC deletion fails
kubectl delete svc --all
kubectl delete ingress --all
cdk destroy
```

## Common Errors
- **`error: You must be logged in`** → run `aws eks update-kubeconfig` with the `--role-arn` matching mastersRole.
- **Pod stuck `Pending` — no node selector match** → node group AZ mismatch vs PVC zone, or taints on nodes.
- **IRSA: `Unable to locate credentials`** → pod spec missing `serviceAccountName`, or SA not annotated with role ARN.
- **Cluster delete fails: VPC has dependencies** → leftover ENIs from LB controller or Fargate. Delete LBs/Ingresses before `cdk destroy`.
- **$73/mo charge even when idle** → control plane is flat-rate. Destroy when done or use a single shared lab cluster.
