# 03 — Detective & Incident Response

## Concept
Amazon Detective = graph of behavior for investigation. IR runbooks automate response.

## Exercises
1. **Enable Detective**; ingest GuardDuty findings. Explore graphs (users, IPs, instances).
2. **Investigate a finding**: pick a GuardDuty sample finding; walk the Detective graph; identify scope.
3. **IR runbook (SSM)**: automation doc that (a) snapshots volumes, (b) isolates instance SG, (c) captures memory via SSM, (d) notifies SNS.
4. **Tag-based quarantine**: Lambda that moves tagged instance to quarantine SG.
5. **Forensics bucket**: dedicated account + bucket with object lock + KMS for evidence.
6. **Tabletop exercise**: write `incident-playbook.md` for "compromised IAM key" scenario.

## Verification
- SSM runbook isolates instance in < 30 sec.
- Evidence bucket immutable under object lock.

## Gotchas
- Detective data ingest = $$. Scope tightly.
- Object lock governance vs compliance mode — irreversible if misconfigured.

## Cleanup
```bash
cdk destroy
# disable Detective
```
