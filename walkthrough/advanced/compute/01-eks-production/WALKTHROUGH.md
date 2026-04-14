# Walkthrough — 01 EKS Production

## About this service
**EKS (Elastic Kubernetes Service)** is AWS's managed Kubernetes control plane. "Production EKS" means the layers you bolt on top: **GitOps** (ArgoCD/Flux) for declarative deploys, a **service mesh** (Istio/App Mesh) for mTLS + traffic shifting, **Karpenter** for just-in-time node provisioning, **External Secrets Operator (ESO)** to sync Secrets Manager into K8s Secrets, and **Pod Security Standards** + **NetworkPolicies** for defense-in-depth. None of these are AWS-managed — you install them via Helm on top of the cluster.

**Why it matters:** a vanilla EKS cluster without these add-ons rots fast. GitOps eliminates `kubectl apply` drift, Karpenter cuts EC2 waste 30-60%, mesh enforces zero-trust, and ESO is the only sane way to rotate secrets without restarts.
**When to use:** multi-team platforms, regulated environments (PCI/HIPAA/SOC2), heterogeneous workloads (GPU + CPU + spot), orgs already invested in Kubernetes tooling.
**When NOT to use:** single-app teams (ECS/Fargate is cheaper ops), <5 engineers (operational burden dwarfs benefit), pure event-driven workloads (Lambda), Windows-only shops.

## Estimated cost
- **EKS control plane: $0.10/hour = $73/month per cluster** (always on)
- **Karpenter-managed nodes**: example 3× m5.large on-demand = $207/month, or mostly-spot ~$70/month
- **NAT Gateway: $32/month + $0.045/GB** (Istio cross-AZ chatter adds up)
- **Load Balancer Controller ALBs**: $16/month each; Istio ingress gateway NLB $16/month
- **Secrets Manager**: $0.40/secret/month + $0.05 per 10k API calls (ESO polls often — batch it)
- **ArgoCD, Karpenter, ESO, Istio**: free (OSS), but consume ~1.5 vCPU + 3 GB across the cluster
- Total for this lesson: **~$180/month** with 2 spot nodes + 1 ALB + 1 NLB. Destroy after!

---

## Step 1: Bootstrap the EKS cluster with CDK
> **Why:** You need a working cluster before you can install anything on it. `eks.Cluster` L2 provisions the control plane, managed node group, OIDC provider (required for IRSA), and wires up aws-auth so `kubectl` works from your laptop. Enabling IRSA here unlocks every add-on we install later.

`lib/eks-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as eks from 'aws-cdk-lib/aws-eks';
import * as iam from 'aws-cdk-lib/aws-iam';
import { KubectlV30Layer } from '@aws-cdk/lambda-layer-kubectl-v30';

export class EksStack extends cdk.Stack {
  public readonly cluster: eks.Cluster;
  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 3, natGateways: 1 });

    const adminRole = new iam.Role(this, 'AdminRole', {
      assumedBy: new iam.AccountRootPrincipal(),
    });

    this.cluster = new eks.Cluster(this, 'Cluster', {
      vpc,
      version: eks.KubernetesVersion.V1_30,
      kubectlLayer: new KubectlV30Layer(this, 'KubectlLayer'),
      defaultCapacity: 0, // Karpenter will provision
      mastersRole: adminRole,
      endpointAccess: eks.EndpointAccess.PUBLIC_AND_PRIVATE,
    });

    // Minimal system node group for Karpenter itself + CoreDNS
    this.cluster.addNodegroupCapacity('system', {
      instanceTypes: [new ec2.InstanceType('t3.medium')],
      minSize: 2, maxSize: 3,
      labels: { role: 'system' },
    });

    new cdk.CfnOutput(this, 'ConfigCommand', {
      value: `aws eks update-kubeconfig --name ${this.cluster.clusterName} --role-arn ${adminRole.roleArn}`,
    });
  }
}
```

```bash
cdk deploy        # 15-20 min on first create
aws eks update-kubeconfig --name <cluster> --role-arn <arn>
kubectl get nodes # 2 system nodes Ready
```

## Step 2: Install ArgoCD via Helm
> **Why:** GitOps flips the deployment model — instead of CI pushing to the cluster (requires cluster creds in CI), the cluster pulls from Git. ArgoCD reconciles actual state to desired state every 3 minutes and self-heals drift. Installing via Helm keeps the install reproducible.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --set server.service.type=LoadBalancer \
  --version 7.6.12

# Fetch initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

ARGO_LB=$(kubectl -n argocd get svc argocd-server \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
open https://$ARGO_LB   # login: admin / <password>
```

## Step 3: App-of-Apps root application
> **Why:** Instead of registering 50 Argo apps by hand, one root `Application` points at a Git folder full of Application manifests. Add a new app → commit YAML → Argo discovers and deploys it. This is how you scale GitOps past 3 services.

`gitops/root-app.yaml` (commit to a Git repo Argo can read):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/eks-gitops
    targetRevision: main
    path: apps         # contains child Application manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

```bash
kubectl apply -f gitops/root-app.yaml
argocd app list   # root + every child app appears
```

Test self-heal: `kubectl delete deploy sample-app` — watch Argo re-create it within 3 min.

## Step 4: Install Karpenter
> **Why:** Managed node groups make you pre-declare instance types and scale via Cluster Autoscaler (slow, coarse). Karpenter provisions EC2 directly in ~45s, picks the cheapest instance that fits pending pods across dozens of types, and consolidates underutilized nodes. Expect 30-60% cost reduction vs fixed ASGs.

```bash
KARPENTER_NAMESPACE=kube-system
CLUSTER_NAME=<your-cluster>

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version 1.0.6 \
  --namespace $KARPENTER_NAMESPACE \
  --set settings.clusterName=$CLUSTER_NAME \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<karpenter-irsa-arn>
```

`karpenter/nodepool.yaml` — spot-first diverse pool with consolidation:
```yaml
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
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot","on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64","arm64"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
  limits: { cpu: "100", memory: "400Gi" }
```

```bash
kubectl apply -f karpenter/nodepool.yaml
kubectl scale deploy sample-app --replicas=20
kubectl get nodes -w   # Karpenter spins up a single large node instead of many small
```

## Step 5: External Secrets Operator (ESO)
> **Why:** Storing secrets in Git-committed ConfigMaps is a breach waiting to happen. Sealed-secrets is operationally painful. ESO is the standard: put the secret in Secrets Manager, declare a `SecretStore` + `ExternalSecret`, and a native K8s `Secret` materialises and stays in sync. Rotate in Secrets Manager → pods pick it up on next poll (1 min default).

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

`secretstore.yaml`:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata: { name: aws-sm }
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef: { name: eso-sa, namespace: external-secrets }
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds, namespace: app }
spec:
  refreshInterval: 1m
  secretStoreRef: { name: aws-sm, kind: ClusterSecretStore }
  target: { name: db-creds }
  data:
    - secretKey: password
      remoteRef: { key: prod/db, property: password }
```

Verify rotation:
```bash
aws secretsmanager update-secret --secret-id prod/db \
  --secret-string '{"password":"newpass"}'
sleep 90
kubectl -n app get secret db-creds -o jsonpath='{.data.password}' | base64 -d
# newpass
```

## Step 6: Install Istio + canary traffic split
> **Why:** Without a mesh, mTLS between services means each team writes TLS code. Istio injects an Envoy sidecar that handles mTLS, retries, circuit breakers, and weighted traffic splits — all declaratively. The Bookinfo sample lets us demonstrate a 90/10 canary without modifying app code.

```bash
istioctl install --set profile=demo -y
kubectl label namespace app istio-injection=enabled
kubectl apply -n app -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml
```

`canary.yaml`:
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: reviews, namespace: app }
spec:
  hosts: [reviews]
  http:
    - route:
        - destination: { host: reviews, subset: v1 }
          weight: 90
        - destination: { host: reviews, subset: v3 }
          weight: 10
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: { name: reviews, namespace: app }
spec:
  host: reviews
  subsets:
    - name: v1
      labels: { version: v1 }
    - name: v3
      labels: { version: v3 }
```

```bash
kubectl apply -f canary.yaml
for i in {1..100}; do curl -s http://$GATEWAY/productpage | grep -o 'stars'; done | sort | uniq -c
# ~90 v1 responses, ~10 v3 responses
```

## Step 7: Pod Security Standards + NetworkPolicy
> **Why:** Default K8s namespaces let pods run as root, mount hostPath, and reach any other pod. PSS `restricted` blocks the worst abuses at the API server with zero runtime cost. A default-deny NetworkPolicy forces teams to declare what their app actually needs to talk to — catching supply-chain compromises that try to beacon out.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.30
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: app }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-reviews, namespace: app }
spec:
  podSelector: { matchLabels: { app: reviews } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: productpage } }
      ports: [{ port: 9080 }]
```

Install Cilium or Calico for NetworkPolicy enforcement (VPC CNI alone does not enforce egress).

## Step 8: Blue/green cluster upgrade
> **Why:** In-place control plane upgrades work but add-on compatibility (Karpenter, Istio, AWS Load Balancer Controller) is the real risk. The safer pattern: upgrade control plane, then stand up a **new** managed node group on the new version, cordon/drain the old one, then delete it. Pods reschedule onto new nodes under your control.

```bash
# 1. Upgrade control plane
aws eks update-cluster-version --name <cluster> --kubernetes-version 1.31

# 2. Add new node group (v1.31 AMI)
aws eks create-nodegroup --cluster-name <cluster> \
  --nodegroup-name system-1-31 --version 1.31 --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=3,desiredSize=2 \
  --subnets subnet-xxx subnet-yyy

# 3. Cordon + drain old nodes
kubectl cordon -l eks.amazonaws.com/nodegroup=system
kubectl drain -l eks.amazonaws.com/nodegroup=system --ignore-daemonsets --delete-emptydir-data

# 4. Delete old node group
aws eks delete-nodegroup --cluster-name <cluster> --nodegroup-name system
```

## Cleanup
```bash
# Delete service LBs before cdk destroy (orphaned ELBs block VPC deletion)
kubectl delete svc --all -A
helm uninstall argocd -n argocd
helm uninstall karpenter -n kube-system
istioctl uninstall --purge -y
cdk destroy
```

## Common Errors
- **`error: You must be logged in to the server (Unauthorized)`** → your IAM identity is not in `aws-auth` ConfigMap. Assume the `mastersRole` or add your ARN via `eksctl create iamidentitymapping`.
- **Karpenter launches nodes but pods stay Pending** → missing `karpenter.sh/discovery` tag on subnets/SGs, or NodeClass IAM instance profile not allowed to pull from ECR.
- **ESO `ClusterSecretStore` InvalidIdentityToken** → IRSA trust policy missing; service account annotation `eks.amazonaws.com/role-arn` wrong.
- **Istio sidecars not injected** → namespace label `istio-injection=enabled` missing, or pod has `sidecar.istio.io/inject: "false"`.
- **ArgoCD stuck `OutOfSync` on CRDs** → set `ServerSideApply=true` in sync options; client-side apply truncates large CRDs.
- **NetworkPolicy not enforced** → running VPC CNI without Cilium/Calico. VPC CNI only enforces policies on EKS 1.25+ with the network policy agent enabled.
- **Cluster upgrade stuck at 1.31** → PodDisruptionBudget with `minAvailable: 100%` blocks drain. Fix the PDB first.
