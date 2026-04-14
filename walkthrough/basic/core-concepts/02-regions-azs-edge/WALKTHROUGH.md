# Walkthrough — 02 Regions, AZs, Edge

## About this lesson
AWS's physical footprint has three levels: **Regions** (isolated geographies like `us-east-1`), **Availability Zones** (isolated datacenters within a region — typically 3+), and **Edge locations** (CloudFront PoPs close to users).

**Why it matters:** latency follows physics — a round-trip across the Pacific is ~200ms no matter how much money you throw at it. Compliance (GDPR, HIPAA, data residency) forces specific regions. Service availability varies — Bedrock, Local Zones, etc. aren't in every region.

**When to use each:**
- Region choice → always (picking where workloads live)
- AZs → always (multi-AZ for HA)
- Edge → when serving global users (latency-sensitive content)

**When NOT to worry:** dev/learning accounts; just use one region (`us-east-1` is cheapest and has everything).

## Estimated cost
- Running probes + describing AZs/regions: **free** (API calls are free)
- Total: **$0.00**

---

## Step 1: Latency probes
> **Why:** Numbers beat intuition. Measuring RTT to 3 regions makes the latency-vs-cost trade-off concrete for later decisions.

```bash
ping -c 5 ec2.us-east-1.amazonaws.com
ping -c 5 ec2.eu-west-1.amazonaws.com
ping -c 5 ec2.ap-south-1.amazonaws.com
```

Record the `avg` RTT. Example from US East Coast:
| Region        | RTT avg |
|---------------|---------|
| us-east-1     | ~15 ms  |
| eu-west-1     | ~80 ms  |
| ap-south-1    | ~230 ms |

## Step 2: List AZs
> **Why:** You'll learn that `us-east-1a` in your account is NOT `us-east-1a` in a coworker's account. AWS randomizes the letters per account to spread load across physical AZs. Use **AZ IDs** (`use1-az1`) for anything cross-account.

```bash
aws ec2 describe-availability-zones --region us-east-1 \
  --query 'AvailabilityZones[].[ZoneName,ZoneId]' --output table
```

Expected:
```
+---------------+------------------+
|  us-east-1a   |  use1-az4        |
|  us-east-1b   |  use1-az6        |
|  us-east-1c   |  use1-az1        |
|  us-east-1d   |  use1-az2        |
|  us-east-1e   |  use1-az3        |
|  us-east-1f   |  use1-az5        |
+---------------+------------------+
```

## Step 3: Service availability matrix
> **Why:** Picking a region without checking service availability is how people get stuck mid-project. Bedrock, SageMaker features, and new services are often us-east-1 / us-west-2 first.

Use https://docs.aws.amazon.com/general/latest/gr/rande.html. Write `services.md`:

```markdown
| Service            | us-east-1 | eu-west-1 | ap-south-1 | Notes                       |
|--------------------|-----------|-----------|------------|------------------------------|
| Bedrock            | ✓         | ✓         | ✗          | Expanding rapidly            |
| Local Zones        | ✓ (many)  | ~         | ✗          | City-level                   |
| Outposts           | all       | all       | all        | Needs hardware install       |
| SageMaker HyperPod | ✓         | limited   | ✗          | New service                  |
| CodeCatalyst       | limited   | ✗         | ✗          | Service-specific endpoint    |
```

## Step 4: Decision doc
> **Why:** Practicing the decision process here makes real-world architecture calls easier. The constraints (users, compliance, service availability) compound.

Create `decision.md`:

```markdown
# Region choice — users in India + EU under GDPR

## Decision
Primary: `eu-central-1` (Frankfurt)
Secondary: `ap-south-1` (Mumbai)

## Why
- GDPR: eu-central-1 is in EU, data never leaves unless replicated explicitly.
- India users: ap-south-1 gives ~30ms latency vs 200ms to EU.
- Avoid `us-*`: cross-border transfer friction under GDPR Schrems II.

## Trade-offs
- Bedrock not in ap-south-1 (as of writing) → AI workloads run from eu-central-1.
- Higher inter-region data transfer cost vs single-region.
```

## Verification
You should now know:
- AZ names are account-specific; AZ IDs are stable.
- us-east-1 is default but hosts global control planes → coupling risk.
- Not every service is in every region.
