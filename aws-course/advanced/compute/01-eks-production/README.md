# 01 — EKS Production

## Concept
GitOps (ArgoCD/Flux), service mesh (App Mesh/Istio), Fargate profiles, Karpenter, secrets via ESO.

## Exercises
1. **ArgoCD**: install via Helm. Point to a Git repo with manifests. Observe auto-sync.
2. **App of Apps pattern**: one root Argo app manages N child apps.
3. **External Secrets Operator (ESO)**: sync Secrets Manager → K8s Secret. Rotate secret; observe ESO updates.
4. **Istio**: install; enable sidecar injection; deploy Bookinfo sample; add canary traffic split.
5. **Karpenter NodePool tuning**: diverse instance families, spot-first with on-demand fallback, consolidation.
6. **Network policies** (Calico or Cilium): default deny, allow specific app flows.
7. **Pod Security Standards**: enforce `restricted` on app namespaces.
8. **Cluster upgrade**: upgrade control plane + node groups using Blue/Green node groups.

## Verification
- Argo auto-heals drifted resources.
- Istio traffic split 90/10 works via curl loop.

## Gotchas
- Many options — pick one (Argo vs Flux, Istio vs App Mesh).
- Mesh adds ~1-2ms latency, 5-10% memory overhead.

## Cleanup
```bash
kubectl delete svc --all  # removes auto-created LBs first
cdk destroy
```
