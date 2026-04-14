# 01 — CloudWatch Logs & Metrics

## Concept
Central logging + metrics + alarms. Log groups → log streams. Metric filters extract numbers from logs.

## Exercises
1. **Lambda → Logs**: use Lambda from `compute/01`. Log a structured JSON line with `duration_ms`.
2. **Metric filter**: create a filter on `$.duration_ms` → publishes custom metric `MyApp/Duration`.
3. **Alarm**: alarm if p95 `Duration` > 1000 ms over 5 min. Wire to SNS topic → your email.
4. **Dashboard**: add 3 widgets (invocations, errors, p95 duration). Share public link.
5. **Logs Insights query**:
   ```
   fields @timestamp, @message
   | filter @message like /ERROR/
   | stats count() by bin(1m)
   ```
6. **Retention**: set log group retention to 7 days (default is never — expensive).

## Verification
- Alarm fires when you inject slow invocations.
- Email arrives from SNS.

## Gotchas
- Default retention = infinite = $$$.
- Metric filters only apply to NEW logs.
- Custom metrics cost $0.30/month each.

## Cleanup
```bash
cdk destroy
```
