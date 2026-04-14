# Walkthrough — 04 EBS & EFS

## About these services
**EBS (Elastic Block Store)** = virtual disk attached to a single EC2 instance. Like plugging a drive into a VM. Persists across stop/start. AZ-bound.

**EFS (Elastic File System)** = managed NFS. Multiple instances mount it simultaneously. Auto-scales capacity. Multi-AZ by default.

**Why these matter:** EC2 needs storage. Pick wrong and you either can't share files across instances (EBS-only) or you pay 3-5x more than needed (EFS everywhere).

**When to use EBS:** OS disk, databases, single-writer workloads. Default choice.
**When to use EFS:** shared home dirs, CMS with multiple web nodes, ML training datasets shared across workers.
**When NOT to use either:** object storage use case (use S3). Short-lived data (use instance store / tmpfs).

## Estimated cost
- **EBS gp3: $0.08/GB/mo** + $0.005 per provisioned IOPS above 3000 baseline
- **EBS io2: $0.125/GB/mo** + IOPS charges (expensive)
- **EBS snapshots: $0.05/GB/mo**
- **EFS Standard: $0.30/GB/mo** (10x more expensive than S3 standard!)
- **EFS IA: $0.025/GB/mo** (for rarely-accessed files)
- Total for this lesson: **~$1/month** if you destroy at the end

---

## Part A: EBS

### Step 1: Launch EC2 with extra volume
> **Why:** Most EC2s have root volume + data volume pattern. Separating them makes backups and sizing independent.

Add to your EC2 stack:
```typescript
const instance = new ec2.Instance(this, 'EbsBox', {
  vpc,
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
  machineImage: ec2.MachineImage.latestAmazonLinux2023(),
  blockDevices: [
    { deviceName: '/dev/xvda', volume: ec2.BlockDeviceVolume.ebs(8) },         // root
    { deviceName: '/dev/sdf',  volume: ec2.BlockDeviceVolume.ebs(20, {
      volumeType: ec2.EbsDeviceVolumeType.GP3,
    }) },
  ],
});
```

### Step 2: Format & mount (inside instance via SSM)
> **Why:** AWS provisions the volume but doesn't format it — the VM owns the filesystem. Standard Linux admin skills apply.

```bash
lsblk
# nvme1n1 appears (the /dev/sdf)
sudo mkfs.ext4 /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
echo "hello EBS" | sudo tee /data/test.txt

# Persist across reboot
echo "$(sudo blkid /dev/nvme1n1 -s UUID -o export) /data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

### Step 3: Snapshot
> **Why:** Snapshots = point-in-time copies stored in S3 internally. Cheaper than a live volume. Basis of AWS Backup for anything EBS.

```bash
VOLUME=$(aws ec2 describe-volumes --filters "Name=attachment.device,Values=/dev/sdf" \
  --query 'Volumes[0].VolumeId' --output text)
SNAP=$(aws ec2 create-snapshot --volume-id $VOLUME --description "learn snap" \
  --query 'SnapshotId' --output text)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP
```

### Step 4: Restore from snapshot
> **Why:** This is the disaster-recovery move. Also: same technique to migrate volumes between AZs (just create the new volume in a different AZ).

```bash
NEW_VOL=$(aws ec2 create-volume --snapshot-id $SNAP \
  --availability-zone <az-of-new-instance> --volume-type gp3 \
  --query 'VolumeId' --output text)
aws ec2 attach-volume --volume-id $NEW_VOL --instance-id $NEW_INSTANCE --device /dev/sdg
```

Inside second instance:
```bash
sudo mount /dev/nvme1n1 /mnt
cat /mnt/test.txt   # "hello EBS"
```

### Step 5: Volume type cheat sheet
| Type   | IOPS       | Throughput | Use case                      |
|--------|------------|------------|-------------------------------|
| gp3    | 3000–16000 | 125–1000 MB/s | General purpose (default)  |
| io2    | 100–64000  | 4000 MB/s  | High-perf DBs (99.999% dur)   |
| st1    | 500 (burst)| 500 MB/s   | Big sequential (logs, DW)     |
| sc1    | 250        | 250 MB/s   | Cold sequential               |

## Part B: EFS

### Step 1: CDK
> **Why:** EFS needs mount targets per AZ (one ENI per AZ that instances connect to) + SG allowing port 2049 (NFS). CDK's L2 wraps this, but understand the parts.

```typescript
import * as efs from 'aws-cdk-lib/aws-efs';

const fs = new efs.FileSystem(this, 'Fs', {
  vpc,
  lifecyclePolicy: efs.LifecyclePolicy.AFTER_30_DAYS,
  performanceMode: efs.PerformanceMode.GENERAL_PURPOSE,
  throughputMode: efs.ThroughputMode.ELASTIC,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
fs.connections.allowDefaultPortFrom(instanceA);
fs.connections.allowDefaultPortFrom(instanceB);
```

### Step 2: Mount on 2 EC2s
> **Why:** The amazon-efs-utils package handles TLS + automount; prefer it over raw NFS client. Proving 2 instances can both mount demonstrates EFS's core value.

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir /shared
sudo mount -t efs <fs-id>:/ /shared
```

### Step 3: Cross-instance test
Instance A:
```bash
echo "from A" | sudo tee /shared/hello
```
Instance B:
```bash
cat /shared/hello    # "from A"
```

## Cleanup
```bash
aws ec2 delete-snapshot --snapshot-id $SNAP
cdk destroy
```

## Common Errors
- **Volume attach fails** — AZ mismatch (volume and instance must be same AZ).
- **EFS mount "connection refused"** — SG doesn't allow 2049 or missing mount target in the AZ.
- **Encrypted snapshot copy cross-region** — must re-encrypt with destination-region KMS key.
