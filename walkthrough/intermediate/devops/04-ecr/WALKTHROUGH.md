# Walkthrough — 04 ECR

## About this service
**Amazon Elastic Container Registry (ECR)** is a managed Docker registry. You get private repositories with IAM-based access, automatic image scanning (basic CVE scan on push, or continuous Enhanced scanning via Inspector), lifecycle policies that expire old images, and a pull-through cache that mirrors `public.ecr.aws`, Docker Hub, Quay, and GitHub Container Registry into your account. It's tightly integrated with ECS, EKS, Lambda (container images), and App Runner — pulls from same-region ECR are free and fast.

**Why it matters:** Docker Hub rate limits (100 pulls / 6h unauth) will break your CI on a bad morning. ECR with pull-through cache costs pennies and never rate-limits you. Image scanning + lifecycle policies are table-stakes for any SOC 2 program.

**When to use:** any workload running ECS/EKS/Fargate/Lambda containers, any team hit by Docker Hub rate limits, regulated environments that need image provenance and CVE reports.
**When NOT to use:** purely public open-source distributions (use ECR **Public** or GitHub Container Registry), or if you need multi-cloud pulls from outside AWS (egress cost).

## Estimated cost
- **Storage: $0.10/GB/month** (compressed image layers)
- **Data transfer OUT to internet: $0.09/GB** (first 100GB/mo); **free** to same-region AWS services
- **Basic scanning: free**
- **Enhanced scanning (Inspector): $0.09 per image scan** initial + $0.01/month re-scan
- **Pull-through cache: $0.10/GB/mo storage** — same as regular images
- Example: 20 images × 500MB = 10GB × $0.10 = $1/mo; 50 same-region pulls/day = $0 egress
- Total for this lesson: **~$1/month**

---

## Step 1: Private repo with scan-on-push + lifecycle
> **Why:** `imageScanOnPush` gives you a free CVE scan every push — no ops. The lifecycle policy is critical: every CI run pushes a new tag, and without expiry your bill climbs silently. "Keep last 10" is a safe default for non-prod.

```bash
mkdir ecr-demo && cd ecr-demo
cdk init app --language=typescript
npm install aws-cdk-lib constructs
```

`lib/ecr-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ecr from 'aws-cdk-lib/aws-ecr';

export class EcrStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const repo = new ecr.Repository(this, 'AppRepo', {
      repositoryName: 'myapp',
      imageScanOnPush: true,
      imageTagMutability: ecr.TagMutability.IMMUTABLE,
      encryption: ecr.RepositoryEncryption.AES_256,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      emptyOnDelete: true,
      lifecycleRules: [
        { description: 'Keep last 10 images', maxImageCount: 10 },
        { description: 'Expire untagged after 1 day',
          tagStatus: ecr.TagStatus.UNTAGGED,
          maxImageAge: cdk.Duration.days(1) },
      ],
    });

    new cdk.CfnOutput(this, 'RepoUri', { value: repo.repositoryUri });
  }
}
```

Deploy:
```bash
cdk deploy
# Outputs:
# EcrStack.RepoUri = 111111111111.dkr.ecr.us-east-1.amazonaws.com/myapp
```

## Step 2: Build and push your first image
> **Why:** `get-login-password` is the only supported auth flow — the old `aws ecr get-login` (v1) was deprecated. Tag with the full repo URI before pushing; ECR rejects local-only tag names.

Create a tiny `Dockerfile`:
```dockerfile
FROM public.ecr.aws/docker/library/alpine:3.19
CMD ["echo", "hello from ecr"]
```

```bash
REPO=111111111111.dkr.ecr.us-east-1.amazonaws.com/myapp

aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin $REPO

docker build -t myapp .
docker tag myapp:latest $REPO:v1
docker push $REPO:v1
# v1: digest: sha256:abc123... size: 528
```

## Step 3: Inspect scan findings
> **Why:** Scan-on-push runs in ~30 seconds. Getting in the habit of checking findings before you promote an image to prod catches critical CVEs before your security team does.

```bash
aws ecr describe-image-scan-findings \
  --repository-name myapp --image-id imageTag=v1 \
  --query 'imageScanFindings.findingSeverityCounts'
# { "CRITICAL": 0, "HIGH": 1, "MEDIUM": 3, "LOW": 7 }

aws ecr describe-images --repository-name myapp
# Shows: imageDigest, imageTags, imageSizeInBytes, imagePushedAt
```

## Step 4: Exercise the lifecycle policy
> **Why:** Lifecycle runs asynchronously (~24h) but can be previewed on demand. Previewing is how you sanity-check a new policy before it deletes production images.

Push 15 versioned images, then preview:
```bash
for i in $(seq 1 15); do
  docker tag myapp:latest $REPO:v$i
  docker push $REPO:v$i
done

aws ecr start-lifecycle-policy-preview \
  --repository-name myapp \
  --lifecycle-policy-text "$(aws ecr get-lifecycle-policy --repository-name myapp --query lifecyclePolicyText --output text)"

aws ecr get-lifecycle-policy-preview --repository-name myapp \
  --query 'previewResults[].{tag:imageTags[0],action:action.type}'
# [ { "tag": "v1", "action": "EXPIRE" }, ... keeping v6..v15 ]
```

## Step 5: Pull-through cache for Docker Hub
> **Why:** A pull-through cache lets you write `Dockerfile` FROM lines pointed at your ECR URL; on miss, ECR pulls from Docker Hub once, caches the layers, and serves every subsequent pull locally — no rate limits, no egress from Docker Hub.

```typescript
import * as ecr from 'aws-cdk-lib/aws-ecr';

new ecr.CfnPullThroughCacheRule(this, 'DockerHubCache', {
  ecrRepositoryPrefix: 'dockerhub',
  upstreamRegistryUrl: 'registry-1.docker.io',
  // For authenticated pulls: credentialArn: secret.secretArn,
});
```

Then in any Dockerfile:
```dockerfile
FROM 111111111111.dkr.ecr.us-east-1.amazonaws.com/dockerhub/library/nginx:1.27
```

First pull caches; subsequent pulls are local. Inspect the mirror:
```bash
aws ecr describe-repositories --query 'repositories[?starts_with(repositoryName,`dockerhub/`)]'
```

## Step 6: Cross-account repo policy
> **Why:** When account B runs ECS tasks that pull from account A's registry, attaching a repo policy is simpler and more auditable than juggling IAM roles across accounts.

```typescript
import * as iam from 'aws-cdk-lib/aws-iam';

repo.addToResourcePolicy(new iam.PolicyStatement({
  sid: 'AllowAccountBPull',
  effect: iam.Effect.ALLOW,
  principals: [new iam.AccountPrincipal('222222222222')],
  actions: [
    'ecr:GetDownloadUrlForLayer',
    'ecr:BatchGetImage',
    'ecr:BatchCheckLayerAvailability',
  ],
}));
```

From account B:
```bash
aws ecr get-login-password --region us-east-1 | docker login ...
docker pull 111111111111.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
# Status: Downloaded newer image
```

## Step 7: Enable Enhanced scanning (Inspector)
> **Why:** Basic scan runs once on push. **Enhanced scanning** continuously re-scans every image against the latest CVE feed so a zero-day disclosed next week shows up without a rebuild. Costs $0.09/image initial + $0.01/month ongoing.

```bash
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"scanFrequency":"CONTINUOUS_SCAN","repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}]}]'
```

Findings now flow into Amazon Inspector → view in the Inspector console.

## Cleanup
```bash
cdk destroy
# Pull-through cache mirror repos don't delete with parent — clean manually:
aws ecr describe-repositories --query 'repositories[?starts_with(repositoryName,`dockerhub/`)].repositoryName' --output text \
  | xargs -n1 aws ecr delete-repository --force --repository-name
```

## Common Errors
- **`no basic auth credentials`** on `docker push` → forgot `aws ecr get-login-password | docker login`; creds expire after 12 hours.
- **`name unknown: The repository with name 'X' does not exist`** → repo wasn't created, or wrong region in the URI.
- **`ImageTagAlreadyExistsException`** → you set `imageTagMutability: IMMUTABLE` (good!) and tried to overwrite a tag; use a new tag or the immutable git SHA.
- **`toomanyrequests: You have reached your pull rate limit`** (Docker Hub) → switch the FROM line to a pull-through cache URL.
- **Lifecycle policy not deleting images** → policies run async; can take up to 24h on first application. Use `start-lifecycle-policy-preview` to verify rules.
- **Cross-account pull `denied: User ... is not authorized`** → both the repo policy (resource) AND the IAM role on the pulling side need ECR permissions.
