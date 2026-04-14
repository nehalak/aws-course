# Capstone 01 — SaaS Multi-Tenant

## Goal
Build a multi-tenant SaaS backend with tenant isolation, per-tenant encryption, and usage metering.

## Requirements
- Cognito User Pool with `custom:tenantId` claim
- API Gateway → Lambda with tenant context extracted from JWT
- DynamoDB single table with `tenantId` prefix enforcing row-level isolation via IAM condition (`dynamodb:LeadingKeys`)
- S3 bucket per tenant OR single bucket with per-tenant prefix + access points
- Per-tenant KMS key (silo) for premium tier; shared key (pool) for free tier
- Usage metering → CloudWatch custom metrics by `tenantId`
- Tenant onboarding = Step Functions workflow

## Deliverables
- CDK app with above
- `architecture.md` explaining pool vs silo decisions
- Load test with 3 tenants proving isolation (tenant A JWT cannot read tenant B data)
- Cost attribution by tenant

## Verification
- Isolation test: crafted request with tenant A creds returning tenant B key → 403.
- CloudWatch dashboard: per-tenant request count.

## Gotchas
- Pool vs silo has cost/isolation trade-off — justify explicitly.
- `LeadingKeys` only works with Cognito Identity Pool or signed policies.

## Cleanup
```bash
cdk destroy
```
