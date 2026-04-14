# Walkthrough — 02 Policy as Code

## About this service
**Policy as Code** means your guardrails are version-controlled, tested, and enforced mechanically — not in a Confluence page. The AWS stack is layered: **SCPs** (Organizations-level blanket deny), **cfn-guard** (static scan of CFN / synth output), **cdk-nag** (synth-time Aspects — see lesson 01), **OPA/Gatekeeper** or **Kyverno** (Kubernetes admission), and **IAM Access Analyzer** (generate least-privilege from CloudTrail).

**Why it matters:** Auditors want proof that policy is *enforced*, not merely documented. Static gates in CI fail the PR; admission controllers reject bad pods at `kubectl apply`. Both are cheaper than detective controls that find violations a week later.
**When to use:** multi-account orgs, EKS/K8s workloads, regulated environments (SOC 2, HIPAA, PCI), any team with >5 engineers touching infra.
**When NOT to use:** single sandbox account for one person — overhead > value until you scale.

## Estimated cost
- **AWS Organizations + SCPs**: free.
- **cfn-guard**: free (open-source CLI; runs in CI on GitHub Actions free tier for public repos).
- **OPA Gatekeeper** on EKS: no license; ~100 MiB RAM / 0.1 vCPU per replica × 3 replicas ≈ **$0** incremental if your cluster has headroom, otherwise ~$7/mo of m5.large fragment.
- **Kyverno**: same cost profile as Gatekeeper.
- **IAM Access Analyzer**: external-access analyzer free; **unused-access analyzer $0.20/IAM role/month** in us-east-1.
- **CloudTrail** (needed for AAA policy generation): management events free; ~$2/mo per trail with S3 storage for a small account.
- Total for this lesson: **~$5/month** (AAA on ~20 roles + CloudTrail S3).

---

## Step 1: SCP design for Dev / Prod OUs
> **Why:** SCPs are blanket — no resource scoping, no principal scoping beyond OU. Design rule: *Prod denies destructive/unsafe actions*; *Dev denies only truly dangerous ones (root key disable, org leave, CloudTrail stop)*.

`scps/deny-leave-org.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeaveOrg",
      "Effect": "Deny",
      "Action": [
        "organizations:LeaveOrganization",
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "config:DeleteConfigurationRecorder",
        "config:StopConfigurationRecorder"
      ],
      "Resource": "*"
    }
  ]
}
```

`scps/prod-region-lock.json` — Prod-only, confine to approved regions:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyOutsideApprovedRegions",
    "Effect": "Deny",
    "NotAction": [
      "iam:*", "organizations:*", "route53:*", "cloudfront:*",
      "waf:*", "wafv2:*", "support:*", "sts:*", "budgets:*",
      "ce:*", "cur:*", "globalaccelerator:*"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "us-west-2"] }
    }
  }]
}
```

Attach:
```bash
aws organizations create-policy --name DenyLeaveOrg --type SERVICE_CONTROL_POLICY \
  --content file://scps/deny-leave-org.json
aws organizations attach-policy --policy-id p-xxxxxxxx --target-id ou-xxxx-yyyyyyyy
```

## Step 2: cfn-guard rule — S3 must be encrypted
> **Why:** cfn-guard runs before deploy, reads synth'd CFN templates, and fails the CI job on violations. Rules are declarative (`Guard` DSL) — no Lambdas, no custom code.

Install:
```bash
# macOS
brew install cloudformation-guard
# Linux
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/aws-cloudformation/cloudformation-guard/main/install-guard.sh | sh
```

`guard-rules/s3-encryption.guard`:
```text
# Every S3 bucket must have SSE configured.
let s3_buckets = Resources.*[ Type == 'AWS::S3::Bucket' ]

rule s3_buckets_must_be_encrypted when %s3_buckets !empty {
  %s3_buckets.Properties.BucketEncryption exists
  <<
    Violation: S3 bucket is missing BucketEncryption.
    Fix: add Properties.BucketEncryption.ServerSideEncryptionConfiguration
  >>

  %s3_buckets.Properties.BucketEncryption.ServerSideEncryptionConfiguration[*]
    .ServerSideEncryptionByDefault.SSEAlgorithm IN ['AES256', 'aws:kms', 'aws:kms:dsse']
}

rule s3_buckets_must_block_public when %s3_buckets !empty {
  %s3_buckets.Properties.PublicAccessBlockConfiguration {
    BlockPublicAcls       == true
    BlockPublicPolicy     == true
    IgnorePublicAcls      == true
    RestrictPublicBuckets == true
  }
}
```

Run against a synth'd template:
```bash
npx cdk synth --json > cdk.out/template.json
cfn-guard validate --rules guard-rules/ --data cdk.out/template.json
# FAILED rules
#   s3-encryption.guard/s3_buckets_must_block_public FAIL
#   Resources.RawBucket.Properties.PublicAccessBlockConfiguration  Property does not exist
```

Wire into CI — `.github/workflows/policy.yml`:
```yaml
name: policy-as-code
on: [pull_request]
jobs:
  cfn-guard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci && npx cdk synth '*' --json > all.json
      - name: install cfn-guard
        run: |
          curl --proto '=https' --tlsv1.2 -sSf \
            https://raw.githubusercontent.com/aws-cloudformation/cloudformation-guard/main/install-guard.sh | sh
          echo "$HOME/.guard/bin" >> $GITHUB_PATH
      - run: cfn-guard validate --rules guard-rules/ --data all.json --show-summary all
```

## Step 3: OPA Gatekeeper on EKS
> **Why:** K8s admission runs inside the cluster — it blocks `kubectl apply` before the pod lands. Rego is expressive enough to encode cross-resource rules (e.g. "image must come from this ECR registry").

Install on EKS:
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

ConstraintTemplate — `templates/k8srequiredresources.yaml`:
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names: { kind: K8sRequiredResources }
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          c := input.review.object.spec.containers[_]
          not c.resources.limits.cpu
          msg := sprintf("container %q missing cpu limit", [c.name])
        }
        violation[{"msg": msg}] {
          c := input.review.object.spec.containers[_]
          not c.resources.limits.memory
          msg := sprintf("container %q missing memory limit", [c.name])
        }
        violation[{"msg": msg}] {
          c := input.review.object.spec.containers[_]
          not startswith(c.image, "123456789012.dkr.ecr.us-east-1.amazonaws.com/")
          msg := sprintf("container %q image not from approved ECR: %v", [c.name, c.image])
        }
```

Constraint — `constraints/require-resources-prod.yaml`:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resources-in-prod
spec:
  enforcementAction: deny
  match:
    kinds: [{ apiGroups: [""], kinds: ["Pod"] }]
    namespaces: ["prod"]
```

Apply + verify:
```bash
kubectl apply -f templates/k8srequiredresources.yaml
kubectl apply -f constraints/require-resources-prod.yaml

cat <<EOF | kubectl apply -n prod -f -
apiVersion: v1
kind: Pod
metadata: { name: bad }
spec:
  containers:
    - name: app
      image: nginx:1.27
EOF
# Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
# [require-resources-in-prod] container "app" missing cpu limit
# [require-resources-in-prod] container "app" image not from approved ECR: nginx:1.27
```

## Step 4: Kyverno as alternative
> **Why:** Kyverno policies are plain YAML — no Rego. UX is much gentler for platform engineers who don't want to learn a new language. Identical outcome to Gatekeeper.

`kyverno/require-resources.yaml`:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resources
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-cpu-memory-limits
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [prod]
      validate:
        message: "CPU and memory limits are required."
        pattern:
          spec:
            containers:
              - name: "*"
                resources:
                  limits:
                    cpu:    "?*"
                    memory: "?*"
    - name: require-approved-registry
      match:
        any:
          - resources: { kinds: [Pod], namespaces: [prod] }
      validate:
        message: "Images must come from corporate ECR."
        pattern:
          spec:
            containers:
              - name: "*"
                image: "123456789012.dkr.ecr.us-east-1.amazonaws.com/*"
```

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
kubectl apply -f kyverno/require-resources.yaml
```

Kyverno vs Gatekeeper tradeoffs:
| Dimension            | Gatekeeper (OPA) | Kyverno         |
|----------------------|------------------|-----------------|
| Policy language      | Rego             | YAML            |
| Learning curve       | Steep            | Gentle          |
| Mutation support     | Alpha            | First-class     |
| Cross-resource rules | Strong           | Weaker          |
| Ecosystem            | OPA (TF, Envoy)  | K8s-only        |

## Step 5: IAM Access Analyzer — generate least-privilege
> **Why:** Writing IAM from scratch is guesswork. AAA reads 90 days of CloudTrail, sees which actions the role actually called, and emits a policy that grants only those.

```bash
aws accessanalyzer start-policy-generation \
  --policy-generation-details '{"principalArn":"arn:aws:iam::123456789012:role/AppRole"}' \
  --cloud-trail-details '{"trails":[{"cloudTrailArn":"arn:aws:cloudtrail:us-east-1:123456789012:trail/org-trail","regions":["us-east-1"]}],"accessRole":"arn:aws:iam::123456789012:role/AccessAnalyzerCloudTrailAccess","startTime":"2026-01-14T00:00:00Z"}'
# { "JobId": "abc123..." }

aws accessanalyzer get-generated-policy --job-id abc123... --include-resource-placeholders > generated.json
```

Diff against the existing role, trim + re-apply. Enable the *unused-access* analyzer to keep finding over-grants:
```bash
aws accessanalyzer create-analyzer --analyzer-name unused-access --type ACCOUNT_UNUSED_ACCESS \
  --configuration '{"unusedAccess":{"unusedAccessAge":90}}'
```

## Step 6: Block PRs on policy violations
> **Why:** A gate that warns without blocking gets ignored. Required-status-checks in branch protection is what actually changes behavior.

GitHub: **Settings → Branches → Branch protection → `main`** → require status checks `policy-as-code / cfn-guard` + `cdk-nag`. Pairs with the lesson-01 cdk-nag workflow.

## Cleanup
```bash
# SCPs
aws organizations detach-policy --policy-id p-xxxxxxxx --target-id ou-xxxx-yyyyyyyy
aws organizations delete-policy --policy-id p-xxxxxxxx

# K8s
kubectl delete -f constraints/ -f templates/
helm uninstall kyverno -n kyverno

# AAA
aws accessanalyzer delete-analyzer --analyzer-name unused-access
```

## Common Errors
- **`cfn-guard` FAIL on a resource that doesn't exist** → rule `when` clause is wrong; guard the query with `!empty` or a type filter.
- **SCP `Deny` blocks the very admin fixing it** → SCPs apply even to `AdministratorAccess`. Always test in a sandbox OU first.
- **Gatekeeper constraint never triggers** → `enforcementAction: dryrun` still logs but doesn't block; switch to `deny` for enforcement.
- **Rego `undefined function data.inventory...`** → you referenced the sync cache without enabling `config.gatekeeper.sh` sync for that resource kind.
- **Kyverno policy works in `Audit` but not `Enforce`** → CRD validation passed but the webhook timed out; raise `failurePolicy` timeout or add more replicas.
- **AAA policy generation returns nothing** → CloudTrail didn't cover the time window, or `accessRole` lacks `s3:GetObject` on the trail bucket.
