# 01 — AWS Account Setup

## Concept
Secure account baseline before touching any service. Root account is a nuclear key — lock it away.

## Prereqs
- Fresh AWS account
- Credit card (free tier eligible)

## Exercises
1. **Lock the root user**
   - Enable MFA (hardware or virtual) on root.
   - Delete any root access keys.
   - Set strong password policy (IAM → Account settings).
2. **Create IAM user `cdk-dev`**
   - Attach `AdministratorAccess` (learning only — we'll tighten later).
   - Generate access keys, store in `~/.aws/credentials`.
   - Enable MFA on this user.
3. **Billing guardrails**
   - Enable "IAM user access to Billing".
   - Create a Budget: $20/month with 50%/80%/100% alerts to email.
   - Enable Cost Explorer.
4. **Enable Organizations** (even for a single account)
   - Create one org. Explore SCPs in console (don't attach yet).
5. **CloudTrail**
   - Create a multi-region trail → new S3 bucket.

## Verification
- `aws sts get-caller-identity` returns your `cdk-dev` user.
- Budget shows in console.
- CloudTrail writes events to S3 within 15 min.

## Gotchas
- Root MFA — if you lose it, account recovery is painful.
- Access keys in code → AWS scans GitHub and disables them.

## Cleanup
Keep everything — this is your baseline.
