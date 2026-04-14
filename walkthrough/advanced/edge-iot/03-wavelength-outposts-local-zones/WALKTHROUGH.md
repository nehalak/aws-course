# Walkthrough — 03 Wavelength, Outposts, Local Zones

## About this service
Three AWS products all labelled "edge", each serving a different latency/locality problem. **Local Zones** are small AWS extensions in metro areas (Boston, LA, Miami, etc.); you launch standard EC2/EBS/VPC resources and traffic stays within the metro — ~5–10 ms to local users vs 20–40 ms to parent region. **Wavelength Zones** embed AWS compute inside carrier 5G networks (Verizon, KDDI, Vodafone) so mobile devices reach compute without traversing the public internet — single-digit ms, carrier-gated. **Outposts** is physical AWS hardware shipped to your data center; you get an EC2/EBS/S3 on Outposts/RDS subset managed by AWS but running behind your firewall for latency, data residency, or regulatory reasons.

**Why it matters:** CloudFront/Edge Functions give you low-latency **content**; these three give you low-latency **compute + state**. If you're building real-time gaming, AR/VR, industrial control, or regulated on-prem workloads, the main region isn't close enough — physically or legally.
**When to use Local Zones:** latency-sensitive b2c apps (gaming, live video) with users clustered in supported metros.
**When to use Wavelength:** mobile apps whose users are on 5G and need <10 ms (AR/VR, connected vehicle, real-time video analytics).
**When to use Outposts:** on-prem data residency, factory-floor latency, hybrid during cloud migration, HIPAA/defense workloads.
**When NOT to use any:** "we want edge" with no latency SLA — CloudFront + regional is almost always cheaper and simpler. Don't buy Outposts to avoid a cloud migration; you'll pay AWS prices AND run a DC.

## Estimated cost
- **Local Zones**: free to enable. EC2/EBS priced at LZ rates — typically ~15% premium over parent region. A `t3.medium` is ~$0.048/hr in `us-east-1-bos-1a` (vs $0.0416 in us-east-1). **Data transfer LZ→internet: $0.09/GB**, LZ↔parent region: $0.02/GB.
- **Wavelength**: same-instance-type pricing ~25% premium. t3.medium in `us-east-1-wl1-bos-wlz-1` ≈ $0.052/hr. **Carrier gateway data: $0.09/GB**.
- **Outposts**: minimum rack ≈ **$150k–$225k upfront** OR 3-yr commit from **~$6,000/month** for the smallest Outposts server (1U, 8 vCPU). You also supply power, space, network (10 Gbps back to parent region). This is a real capex decision, not a sandbox toggle.
- Total for this lesson: **~$3/month** (one Local Zone t3.micro for 24 hrs = $0.024 * 24 = $0.58 + ~$0.50 EBS). Wavelength/Outposts are paper exercises — $0.

---

## Step 1: Enable a Local Zone and launch EC2 there
> **Why:** LZs are opt-in per account — until you opt in, the AZ ID doesn't appear in `describe-availability-zones`. Opting in is free. You still need a VPC subnet in that zone; LZ subnets are a special kind tied to the zone ID.

```bash
# List candidate zones (look for ZoneType=local-zone)
aws ec2 describe-availability-zones --all-availability-zones \
  --filters Name=zone-type,Values=local-zone --region us-east-1 \
  --query 'AvailabilityZones[].[ZoneName,GroupName,OptInStatus]' --output table

# Opt in (one-time per group, e.g. us-east-1-bos-1)
aws ec2 modify-availability-zone-group \
  --group-name us-east-1-bos-1 \
  --opt-in-status opted-in --region us-east-1
# Status transitions opted-in over ~5 min
```

CDK — LZ subnet attached to existing VPC:

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';

const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcId: 'vpc-xxxx' });

const lzSubnet = new ec2.CfnSubnet(this, 'LzSubnet', {
  vpcId: vpc.vpcId,
  cidrBlock: '10.0.128.0/24',
  availabilityZone: 'us-east-1-bos-1a',
  mapPublicIpOnLaunch: true,
});

// Route table: default route via parent-region IGW (LZ traffic hairpins through parent IGW by default)
// For direct internet in LZ, attach a carrier/LZ-local IGW instead.

new ec2.CfnInstance(this, 'LzEc2', {
  instanceType: 't3.medium', // not all types available in every LZ
  imageId: ec2.MachineImage.latestAmazonLinux2023().getImage(this).imageId,
  subnetId: lzSubnet.ref,
  keyName: 'my-key',
});
```

Measure latency from a Boston-area ISP:

```bash
ping -c 20 <lz-ec2-public-ip>
# expect ~8-12 ms from Boston ISP
ping -c 20 <us-east-1-ec2-public-ip>
# expect ~25-40 ms (Northern Virginia)
```

## Step 2: Architecture — ultra-low-latency gaming with LZ + Global Accelerator (paper)
> **Why:** Real-time multiplayer needs <30 ms RTT. A single region can't cover a continent. Local Zones in major metros + Global Accelerator anycast IPs give players the nearest edge automatically.

Write `design.md`:

```markdown
# Gaming match server — US coverage via Local Zones

## Goal
< 25 ms p95 RTT for 90% of US players on real-time FPS match servers.

## Topology
- Parent region: us-east-1 (control plane: matchmaking API, player DB = Aurora Global, lobby)
- Local Zones (game-server data plane):
  - us-east-1-bos-1a  (Boston)
  - us-east-1-nyc-1a  (NYC)
  - us-east-1-mia-1a  (Miami)
  - us-east-1-chi-1a  (Chicago)
  - us-west-2-lax-1a  (LA)
  - us-west-2-phx-1a  (Phoenix)
- Each LZ runs an Auto Scaling group of c6i.2xlarge dedicated match hosts.
- Global Accelerator with one listener (UDP 30000-30099), endpoint groups per LZ.
  GA anycast routes player UDP to nearest LZ over AWS backbone.

## Data path
1. Client → matchmaker (Route 53 latency routing → regional API GW).
2. Matchmaker assigns (region, LZ, match-server-IP) based on player geolocation.
3. Client opens UDP directly to GA anycast IP; GA forwards to chosen LZ host.
4. Match server flushes events to Kinesis in parent region every 30s (eventual).

## Why not CloudFront?
CloudFront is HTTP/TCP cached content. Gaming is stateful UDP → need real compute at edge, not cache.

## Why not Wavelength?
WL is carrier-gated; only reachable from that carrier's 5G subscribers. Our players are on home Wi-Fi.

## Cost sketch (1,000 concurrent players, 50/match)
- 20 match hosts across 6 LZs, c6i.2xlarge @ LZ premium ≈ $0.40/hr × 24 × 30 × 20 = $5,760/mo
- Global Accelerator: 2 x $18/mo + $0.015/GB (say 500 GB/mo) = ~$44/mo
- Aurora Global (control plane): ~$400/mo
- Total: ~$6,200/mo
```

## Step 3: Architecture — 5G AR app on Wavelength (paper)
> **Why:** An AR overlay redrawn every frame can't tolerate >20 ms round-trip. Wavelength puts compute **inside** the carrier's 5G packet core, so traffic never leaves Verizon's network on the way to the GPU.

Write `wavelength-design.md`:

```markdown
# 5G AR navigation — Wavelength architecture

## Goal
< 15 ms glass-to-compute RTT for a city-scale AR wayfinding app.

## Topology
- Carrier: Verizon 5G UWB
- Wavelength Zones:
  - us-east-1-wl1-bos-wlz-1 (Boston)
  - us-east-1-wl1-nyc-wlz-1 (NYC)
  - us-east-1-wl1-was-wlz-1 (DC)
  - us-west-2-wl1-sfo-wlz-1 (SF)
- In each WL zone: g4dn.xlarge (NVIDIA T4) running the AR scene renderer / SLAM backend behind an NLB.
- Carrier Gateway (CAGW) instead of IGW — traffic arrives via 5G, not public internet.
- Parent region us-east-1: control plane (user accounts in Cognito, map tiles in S3, metrics in Timestream).

## Data path
1. Device attaches to Verizon 5G UWB → gets carrier IP.
2. Device resolves `ar.example.com` via Route 53 geolocation → WL-zone NLB IP (routable only from Verizon network).
3. UDP/QUIC session to g4dn; sub-10 ms typical.
4. Model weights + large tile fetches go to S3 in parent region (not latency-critical).

## Restrictions (why this is a paper exercise)
- Verizon business agreement required — not self-service for sandbox accounts.
- Device must be on Verizon 5G UWB; LTE/Wi-Fi fallback must go to parent region.

## Cost (1,000 DAU, 10 min sessions)
- 4 WL zones × 2 g4dn.xlarge @ ~$0.66/hr × 24 × 30 = $3,800/mo
- Carrier gateway data: ~200 GB/mo × $0.09 = $18/mo
```

## Step 4: Architecture — HIPAA Outposts deployment (paper)
> **Why:** Regulatory constraints sometimes require PHI never to traverse the public internet or leave a physical facility. Outposts gives you AWS APIs + managed hardware in the hospital's own closet, with KMS keys that can be pinned so decryption happens only on-premises.

Write `outposts-design.md`:

```markdown
# HIPAA medical imaging — Outposts deployment

## Goal
Run a DICOM viewer + ML inference pipeline where image data never leaves the hospital's LAN, while retaining AWS APIs and managed services.

## Hardware
- 1× Outposts rack (42U), 2× 100 Gbps uplinks to parent region us-east-1 over redundant AWS Direct Connect.
- Installed in hospital's MDF, AWS handles delivery + rack-and-stack + firmware.

## Services on-rack
- EC2 (m5 / r5 / g4dn for inference)
- EBS gp2
- S3 on Outposts (bucket type: on-outposts; PHI never syncs to region)
- RDS on Outposts (Postgres)
- ECS / EKS on Outposts for containerized viewer

## Network
- Local Gateway (LGW) replaces IGW — all subnets route to LGW CIDR owned by hospital.
- No IGW / NAT GW at all — outbound to parent region AWS endpoints only via DX private VIFs (VPC endpoints on-region).
- Hospital firewall only permits DX address space.

## KMS / encryption
- CMKs for PHI buckets / EBS: created as **multi-region KMS keys** with replication disabled, OR use AWS KMS custom key store backed by CloudHSM in-region but gated by Outposts-only resource policy that restricts `aws:SourceVpce` to on-prem VPC endpoint.
- All data at rest encrypted with customer-managed CMK; key policies reference Outposts-local IAM roles only.

## Data residency guarantees
- S3 on Outposts bucket policy: `Deny` on any action without `aws:SourceVpc = vpc-onprem`.
- VPC flow logs shipped to a CloudWatch Log Group on-region via DX; metadata only, no payloads.
- BAA with AWS covers EC2/EBS/S3 on Outposts.

## Failure modes
- DX outage: Outposts continues to run EC2/EBS/S3/RDS locally (service link disconnected mode); no new IAM changes possible but existing auth tokens work. Reconnects automatically.
- Rack failure: 2nd Outposts rack in second MDF, active/passive via Route 53 on internal DNS.

## Cost (3-yr upfront)
- Rack: ~$500k-$1M upfront OR ~$15-25k/mo on 3-yr commit (depends on mix).
- DX: 2 × 10 Gbps @ $2,250/mo port + $0.02/GB = ~$5,000/mo.
```

## Step 5: Compare — `edge-options.md`
> **Why:** These four products overlap in marketing but not in engineering. A one-page decision matrix beats re-explaining the diff every design review.

```markdown
# Edge options compared

| Dimension         | CloudFront Functions / L@E | Local Zones                  | Wavelength             | Outposts                  |
|-------------------|----------------------------|------------------------------|------------------------|---------------------------|
| Latency target    | 20-100 ms (cached)         | 5-15 ms (metro)              | < 10 ms (5G device)    | < 2 ms (on-prem LAN)      |
| Runs              | JS / Node / Python         | Any EC2 workload             | Any EC2 workload       | Any EC2 workload + S3/RDS |
| Persistent state  | No                         | Yes (EBS, limited RDS)       | Yes (EBS, limited)     | Yes (full stack)          |
| Who can reach it  | Anyone, HTTPS              | Anyone, internet             | Only carrier 5G subs   | Only on-prem LAN / DX     |
| Self-service      | Yes                        | Yes (opt-in zone)            | Yes + carrier contract | Sales cycle, racks        |
| Upfront cost      | $0                         | $0                           | $0                     | $150k+ or multi-yr commit |
| Use case          | Headers, rewrites, auth    | Gaming, live video, HFT      | AR/VR, connected car   | Data residency, factory   |
| Data residency    | Global (CF POPs)           | Metro of LZ                  | Carrier network        | Your facility             |

## Decision flow
1. Can CloudFront + regional solve it? → start there.
2. Do users need <15 ms and are in supported metros? → Local Zones.
3. Are users mobile on 5G AND carrier supports WL? → Wavelength.
4. Does data need to never leave a physical site? → Outposts.
5. Combination? Often LZ + GA (gaming) or Outposts + region (hybrid) — rarely Wavelength + Outposts.
```

## Cleanup
```bash
# Local Zones: terminate EC2, delete LZ subnet; LZ opt-in itself is free to keep
aws ec2 terminate-instances --instance-ids i-xxxxx --region us-east-1
aws ec2 delete-subnet --subnet-id subnet-xxx --region us-east-1
# Optional opt-out:
aws ec2 modify-availability-zone-group \
  --group-name us-east-1-bos-1 --opt-in-status not-opted-in --region us-east-1

# Wavelength / Outposts: paper only — no teardown.
```

## Common Errors
- **`OptInRequired` when launching LZ instance** → zone group not opted-in, or opt-in still propagating (wait 5 min).
- **`InsufficientInstanceCapacity` in a Local Zone** → LZs have much smaller capacity pools. Try a smaller size (t3 not m5) or a different LZ.
- **`UnsupportedOperation: The requested instance type is not supported in this Availability Zone`** → not every instance family is in every LZ. Check `describe-instance-type-offerings --location-type availability-zone --filters Name=location,Values=us-east-1-bos-1a`.
- **LZ instance has no internet** → default route table inherited from VPC points to region IGW; that works but data hairpins to parent region. For local internet egress, attach an LZ-local IGW.
- **Wavelength instance unreachable from laptop** → WL CAGW only routes carrier-assigned IPs. Test from a 5G UE on the right carrier, not from home Wi-Fi.
- **Outposts "capacity task" stuck** → parent-region service link down. Check DX BGP and service-link endpoints; Outposts local plane keeps running, control plane is degraded.
