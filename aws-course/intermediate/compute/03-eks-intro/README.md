# 03 — EKS Intro

## Concept
Managed Kubernetes control plane. You bring data plane (EC2 node groups, Fargate profiles, or Karpenter).

## Exercises
1. **CDK EKS cluster**: 1.29, 2 managed node groups, `kubectl` access via `aws eks update-kubeconfig`.
2. **Deploy**: `kubectl apply` nginx Deployment + Service type LoadBalancer. Verify NLB created.
3. **IRSA** (IAM Roles for Service Accounts): grant a pod S3 access. `aws s3 ls` from pod works without node role changes.
4. **Karpenter**: install; delete managed node group. Deploy 10 replicas of a 1Gi pod; watch Karpenter provision right-sized nodes.
5. **AWS Load Balancer Controller**: install; use Ingress (ALB) instead of Service LB.
6. **EBS CSI driver**: PVC with `gp3` StorageClass; mount in a pod; write file; delete pod; remount.

## Verification
- `kubectl get nodes` shows Karpenter-provisioned nodes.
- IRSA pod lists S3 buckets.

## Gotchas
- EKS control plane = $73/mo. Delete when done.
- `aws-auth` ConfigMap lockouts. Use access entries (newer).
- Karpenter v1 API changed from Provisioner → NodePool.

## Cleanup
```bash
cdk destroy
# may need to manually delete ALBs created by controller
```
