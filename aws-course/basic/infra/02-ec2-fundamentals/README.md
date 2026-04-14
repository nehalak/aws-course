# 02 — EC2 Fundamentals

## Concept
Virtual machines. AMI = image template. Instance type = CPU/RAM. User data = boot script. Key pair = SSH.

## Exercises
1. **CDK EC2** in the VPC from lesson 01:
   - `t3.micro` (free tier), Amazon Linux 2023 AMI.
   - User data installs nginx and serves "hello".
   - Security group: allow 80 from `0.0.0.0/0`, 22 from your IP only.
2. **SSH in** using Session Manager (no key needed — attach `AmazonSSMManagedInstanceCore` role).
3. **curl the public IP** → see nginx.
4. **Stop vs Terminate**: stop instance, observe ephemeral storage survives. Terminate, observe EBS goes unless `deleteOnTermination=false`.
5. **Instance types**: launch one `t3.micro` and inspect `/proc/cpuinfo`, `free -h`. Compare to `m7g.medium` (ARM Graviton).
6. **AMI lookup**: use `ec2.MachineImage.latestAmazonLinux2023()` — verify CDK resolves the correct AMI ID for your region.

## Verification
- HTTP 200 from curl.
- Session Manager connection works without SSH key.

## Gotchas
- Security group to `0.0.0.0/0` on port 22 → bots find you in minutes.
- EBS charges even when instance is stopped.
- Free tier = 750 hrs/mo of `t3.micro` only.

## Cleanup
```bash
cdk destroy
```
