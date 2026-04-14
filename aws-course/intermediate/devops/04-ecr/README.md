# 04 — ECR

## Concept
Managed Docker registry. Scan on push. Lifecycle policies. Pull-through cache for public registries.

## Exercises
1. **Private repo**: CDK ECR repo with `imageScanOnPush: true`, lifecycle rule = keep last 10 images.
2. **Push**:
   ```bash
   aws ecr get-login-password | docker login --username AWS --password-stdin <id>.dkr.ecr...
   docker build -t myapp . && docker tag myapp <repo>:v1 && docker push <repo>:v1
   ```
3. **Inspect scan findings** in console (CVE list).
4. **Lifecycle cleanup**: push 15 images; verify lifecycle deletes oldest 5.
5. **Pull-through cache**: configure for `public.ecr.aws` + Docker Hub. Pull an image — it appears cached.
6. **Cross-account image sharing**: resource policy allowing account B to pull.
7. **Image signing with Notation**: sign images and enforce verification in deploys.

## Verification
- `aws ecr describe-images` lists your tags.
- Lifecycle report shows expired images.

## Gotchas
- ECR storage = $0.10/GB/mo. Clean up.
- Docker Hub rate limits — use pull-through.
- Scan requires Enhanced (Inspector) for continuous.

## Cleanup
```bash
cdk destroy
```
