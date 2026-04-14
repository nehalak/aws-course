# 01 — VPC Advanced

## Concept
NAT for private egress. VPC endpoints avoid NAT $$ for AWS services. Transit Gateway for hub-spoke. Peering for 1:1.

## Exercises
1. **NAT Gateway vs NAT Instance**: CDK VPC with NAT GW. Then create a NAT Instance manually (cheaper but unmanaged). Compare.
2. **Gateway endpoints** (free): add S3 + DynamoDB gateway endpoints. Test `aws s3 ls` from a private-subnet EC2 with NAT removed.
3. **Interface endpoints** (paid): add one for SSM. Use Session Manager from a private EC2 without NAT.
4. **VPC Peering**: create 2 VPCs with non-overlapping CIDRs, peer them, add route table entries both sides. Ping across.
5. **Transit Gateway**: create TGW, attach 3 VPCs. Update route tables. Ping across.
6. **Flow Logs**: enable at VPC level → S3 → query with Athena (find rejected connections).

## Verification
- EC2 in isolated subnet (no NAT) reaches S3 via gateway endpoint.
- TGW enables 3-way routing.

## Gotchas
- Interface endpoint = $7/mo/AZ. Pick AZs.
- TGW = $36/mo per attachment.
- Flow logs at REJECT only saves cost.

## Cleanup
```bash
cdk destroy  # watch for orphaned ENIs from endpoints
```
