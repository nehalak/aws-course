# 01 — Lambda Advanced

## Concept
Layers for shared code. Provisioned concurrency = pre-warmed. VPC mode needs ENIs. SnapStart for JVM cold starts. Powertools for observability.

## Exercises
1. **Layer**: create a layer with `axios` + shared utils. Reference from 2 functions.
2. **VPC Lambda**: place Lambda in private subnet with SG. Access RDS from it. Observe ~1s cold start increase first time.
3. **Provisioned concurrency**: set 5. Load test with `hey`; compare p99 vs on-demand.
4. **SnapStart** (Java): deploy a Spring Boot hello. Enable SnapStart. Compare cold start before/after (10x improvement typical).
5. **Powertools**: add `@aws-lambda-powertools/logger`, `tracer`, `metrics`. Ship to CloudWatch + X-Ray.
6. **Destinations**: async Lambda with SQS on success, SNS on failure destinations.
7. **Function URL**: enable; call directly via HTTPS without API GW.

## Verification
- Layer reduces deployment package size.
- Provisioned concurrency → p99 < 50ms.

## Gotchas
- VPC Lambda ENI creation during init (historic); modern = Hyperplane ENIs.
- Layer size counts against 250MB unzipped limit.
- Provisioned concurrency costs whether used or not.

## Cleanup
```bash
cdk destroy
```
