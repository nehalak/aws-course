# 02 — CloudWatch Advanced

## Concept
Metric math, EMF (Embedded Metric Format), dashboards, Synthetics canaries, ServiceLens, Contributor Insights.

## Exercises
1. **EMF from Lambda**: log a JSON EMF block — metric auto-created without API call. Use Powertools Metrics.
2. **Metric math dashboard widget**: `errorRate = errors / invocations * 100` using `e1 = m1/m2*100`.
3. **Composite alarm**: `(HighErrorRate AND HighLatency) OR ServiceDown` — alarms an alarm.
4. **Synthetics canary**: heartbeat every 1 min against `/health`. Screenshot on fail.
5. **Contributor Insights**: enable on DynamoDB table; find top hot key.
6. **Anomaly detection alarm**: band-based alarm on invocation count.
7. **Log Insights scheduled query**: daily roll-up → SNS email.

## Verification
- EMF metric shows up auto in namespace.
- Canary runs on schedule; breaks on endpoint down.

## Gotchas
- Synthetics = $0.0012/run (adds up at 1/min).
- EMF log size counts against CloudWatch ingest.

## Cleanup
```bash
cdk destroy
```
