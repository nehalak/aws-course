# 04 — App Runner & Lightsail

## Concept
App Runner = managed container PaaS (like Cloud Run). Lightsail = simplified VPS/DB at fixed price. Both trade flexibility for simplicity.

## Exercises
1. **App Runner from ECR**: deploy an image; auto-scale 1-10; custom domain + ACM.
2. **App Runner from source (GitHub)**: auto-build on push. Review apprunner.yaml.
3. **VPC connector**: let App Runner reach RDS in private subnet.
4. **Lightsail instance** ($5/mo): deploy WordPress blueprint. SSH in.
5. **Lightsail container service** vs App Runner — write `decision.md` when to use which.
6. **Limitations**: both lack deep VPC integration vs ECS; no custom networking. Document.

## Verification
- App Runner URL serves your app.
- Lightsail instance reaches from browser.

## Gotchas
- App Runner min cost ~$5/mo even idle (provisioned mode).
- Lightsail ≠ EC2 — separate console, separate billing.

## Cleanup
```bash
# delete via console or:
aws apprunner delete-service ...
aws lightsail delete-instance ...
```
