# Walkthrough — 01 CloudWatch Logs & Metrics

## About this service
**CloudWatch** is AWS's observability platform. **Logs** store log events in log groups/streams. **Metrics** are time-series numbers (CPU %, invocations, custom metrics). **Alarms** watch metrics and trigger actions (SNS, autoscaling). **Dashboards** show metrics. **Logs Insights** queries logs with a SQL-like language.

**Why it matters:** if you can't observe it, you can't debug or improve it. Every AWS service emits metrics; logs are the breadcrumb trail during incidents.

**When to use CloudWatch:** always — it's the default. Most AWS services push metrics automatically.
**When NOT to use CloudWatch only:** complex distributed traces (add X-Ray or OTel), high-cardinality data (Honeycomb/Datadog are better), tight budget at log scale (costs explode — consider shipping to cheaper S3).

## Estimated cost
- **Logs ingestion: $0.50/GB** (this is often the biggest surprise bill)
- **Logs storage: $0.03/GB/mo**
- **Custom metrics: $0.30 per metric/month** (after 10 free)
- **Standard metrics**: free (CPU, network, etc.)
- **Dashboards: $3/month** per dashboard over 3
- **Alarms: $0.10/alarm/month** (standard)
- **Synthetics: $0.0012/run**
- Total for this lesson: **~$1/month** — but enable `logRetention` on all Lambdas!

---

## Step 1: Lambda emitting structured logs
> **Why:** Structured (JSON) logs are queryable. Plain-text logs require regex. Always emit JSON in production. Powertools makes this easier (covered in intermediate).

`lambda/handler.ts`:
```typescript
export const handler = async () => {
  const start = Date.now();
  await new Promise(r => setTimeout(r, Math.random() * 2000));
  const duration_ms = Date.now() - start;
  console.log(JSON.stringify({ level: 'INFO', event: 'handled', duration_ms }));
  return { ok: true };
};
```

## Step 2: Stack with filter + alarm
> **Why:** Metric filter extracts numbers from logs → creates a CloudWatch metric → alarm on it. This is the foundation of custom observability.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as cw from 'aws-cdk-lib/aws-cloudwatch';
import * as cwa from 'aws-cdk-lib/aws-cloudwatch-actions';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as snssub from 'aws-cdk-lib/aws-sns-subscriptions';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';

const fn = new NodejsFunction(this, 'App', {
  entry: 'lambda/handler.ts',
  logRetention: 7,
});

new logs.MetricFilter(this, 'DurFilter', {
  logGroup: fn.logGroup,
  filterPattern: logs.FilterPattern.exists('$.duration_ms'),
  metricName: 'Duration',
  metricNamespace: 'MyApp',
  metricValue: '$.duration_ms',
});

const topic = new sns.Topic(this, 'Alerts');
topic.addSubscription(new snssub.EmailSubscription('you@example.com'));

const alarm = new cw.Alarm(this, 'SlowAlarm', {
  metric: new cw.Metric({
    namespace: 'MyApp', metricName: 'Duration',
    statistic: cw.Stats.percentile(95),
    period: cdk.Duration.minutes(5),
  }),
  threshold: 1000,
  evaluationPeriods: 1,
});
alarm.addAlarmAction(new cwa.SnsAction(topic));

new cw.Dashboard(this, 'Dash', {
  dashboardName: 'MyAppDashboard',
  widgets: [
    [new cw.GraphWidget({ title: 'Invocations', left: [fn.metricInvocations()] })],
    [new cw.GraphWidget({ title: 'Errors', left: [fn.metricErrors()] })],
    [new cw.GraphWidget({
      title: 'p95 Duration',
      left: [new cw.Metric({ namespace: 'MyApp', metricName: 'Duration', statistic: 'p95' })],
    })],
  ],
});
```

## Step 3: Invoke and observe
> **Why:** Metric filters only process NEW log events — so nothing shows until after filter deployment. Invocations populate the metric. Patience: ~2min lag.

```bash
for i in {1..20}; do
  aws lambda invoke --function-name <fn> --payload '{}' \
    --cli-binary-format raw-in-base64-out /dev/null
done
```

## Step 4: Trigger alarm
> **Why:** Actually receiving the email cements that alarms work end-to-end. In production, you'd route to PagerDuty/Opsgenie instead of personal email.

Modify handler to always sleep 2000ms. Redeploy. Invoke 10x. Alarm fires within 5 min.

## Step 5: Logs Insights query
> **Why:** Logs Insights is incredibly useful for ad-hoc log analysis. Faster than tailing files. Learn the syntax once and it pays off forever.

```
fields @timestamp, @message
| filter duration_ms > 1000
| stats count() by bin(1m)
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **Metric filter doesn't populate metric** — filter only applies to NEW log events. Re-invoke.
- **Alarm stuck in INSUFFICIENT_DATA** — no data points, or metric name/namespace typo.
- **Email never arrives** — didn't click the SNS subscription confirmation.
