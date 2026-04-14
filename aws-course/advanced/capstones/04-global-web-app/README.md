# Capstone 04 — Global Web App

## Goal
Active-active global web app: <200ms to users anywhere, survives region failure.

## Requirements
- CloudFront in front of regional ALBs (us-east-1 + eu-west-1 + ap-southeast-1)
- Latency-based Route 53 for API (`api.domain`)
- Fargate services in all 3 regions
- Aurora Global Database (writer us-east-1; readers elsewhere)
- DynamoDB Global Tables for session data
- S3 multi-region access point for assets
- Cognito in one region (user pool is regional — plan for failover)
- ACM certs per region
- WAF attached globally on CloudFront

## Deliverables
- CDK (multi-stack, multi-region)
- Synthetics canaries from 3 locations
- Chaos test: induce us-east-1 outage; measure RTO
- Cost report cross-region transfer

## Verification
- p95 latency < 200ms from 3 test locations.
- Failover RTO < 5 min with DNS TTL 60s.

## Gotchas
- Aurora Global writer is single-region (manual promote on DR).
- Cognito: consider SCIM sync or User Pool exports.

## Cleanup
```bash
cdk destroy  # in each region
```
