# 02 — Regions, AZs, and Edge

## Concept
Region = isolated geo. AZ = isolated datacenter within region. Edge = CloudFront PoP. Picking wrong = latency + compliance pain.

## Exercises
1. **Latency map**: from your machine, `ping` these endpoints and record RTT:
   - `ec2.us-east-1.amazonaws.com`
   - `ec2.eu-west-1.amazonaws.com`
   - `ec2.ap-south-1.amazonaws.com`
2. **List all AZs in 3 regions**:
   ```bash
   aws ec2 describe-availability-zones --region us-east-1
   aws ec2 describe-availability-zones --region eu-central-1
   aws ec2 describe-availability-zones --region ap-southeast-1
   ```
3. **Service availability matrix**: pick 5 services (Bedrock, Local Zones, Outposts, SageMaker HyperPod, CodeCatalyst) — check which regions support each.
4. **Write a decision doc** (`decision.md`): "If my users are in India + EU and I need GDPR, which regions?"

## Verification
- `decision.md` lists 2 regions with justification (latency + data residency).

## Gotchas
- `us-east-1` hosts global service control planes (IAM, Route53, CloudFront). Outages cascade.
- AZ IDs (`use1-az1`) ≠ AZ names (`us-east-1a`) — names are randomized per account.

## Cleanup
None.
