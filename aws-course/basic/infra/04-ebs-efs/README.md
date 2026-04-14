# 04 — EBS and EFS

## Concept
EBS = block device attached to one EC2 (like a disk). EFS = NFS, shared across many instances.

## Exercises
1. **EBS volumes**: launch EC2 with a second `gp3` volume (20GB). SSH in, `lsblk`, format as ext4, mount at `/data`. Write a file.
2. **Snapshot**: `aws ec2 create-snapshot`. Delete volume. Create new volume from snapshot, attach to a new EC2. Verify file.
3. **Volume types**: compare `gp3` vs `io2` vs `st1` in docs — write a cheat sheet of IOPS/throughput/cost.
4. **EFS**: CDK an EFS filesystem. Mount it on 2 EC2 instances in different AZs. Write from one, read from the other.
5. **Throughput modes**: switch EFS between Bursting and Elastic; run `dd` to measure.

## Verification
- Snapshot restore gives same file.
- EFS reads from instance B what was written from instance A.

## Gotchas
- EBS is AZ-bound. Snapshot to move across AZ/region.
- EFS needs mount targets per AZ + security group allowing NFS (2049).
- Provisioned IOPS on `io2` gets expensive fast.

## Cleanup
```bash
cdk destroy
aws ec2 delete-snapshot --snapshot-id <id>
```
