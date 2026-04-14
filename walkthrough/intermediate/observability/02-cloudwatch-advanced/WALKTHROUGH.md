# Walkthrough — 02 CloudWatch Advanced

## About this service
This lesson stacks the power features of CloudWatch on top of basic logs/metrics/alarms: **EMF (Embedded Metric Format)** emits metrics as a JSON log line — no PutMetricData API call, no SDK, no extra latency. **Metric math** computes derived metrics (error rate, p95-over-avg) directly in dashboards and alarms. **Composite alarms** combine other alarms with boolean logic. **Synthetics canaries** are headless Chrome scripts that poll your endpoints on a schedule. **Contributor Insights** surfaces top-N entities (hot partition keys, noisy IPs) from log patterns. **Anomaly detection** trains a model on a metric's history and alarms on deviation bands.

**Why it matters:** basic CloudWatch tells you what's broken. These features tell you *why*, *who*, and *how often* — and they do it at a fraction of the cost of running a dedicated APM agent.

**When to use:** production systems needing SLO-style alerting (composite alarms), high-cardinality custom metrics cheaply (EMF), synthetic uptime checks, or finding hot keys (Contributor Insights on DynamoDB).
**When NOT to use:** when you already own Datadog/New Relic — they do most of this better and unify tracing/logs/metrics. EMF loses value if your logs bill is already painful.

## Estimated cost
- **EMF:** metric is free; you pay standard log ingestion ($0.50/GB) + custom metric ($0.30/metric/month over 10)
- **Composite alarms: $0.50/alarm/month**
- **Synthetics canaries: $0.0012/run** → 1/min = **~$52/month**, 1/5min = **~$10/month**
- **Contributor Insights rule: $0.50/rule/month + $0.02/M log events processed**
- **Anomaly detection alarm: $0.30/month per metric + standard alarm cost**
- **Logs Insights: $0.0050 per GB scanned**
- Total for this lesson (canary at 5min cadence): **~$15/month**

---

## Step 1: EMF metrics from Lambda
> **Why:** EMF writes a special JSON structure to stdout; the CloudWatch agent in the Lambda environment parses it and creates metrics automatically. Zero extra latency because no API call. Perfect for per-request metrics with high dimensional cardinality.

`lambda/handler.ts`:
```typescript
import { Metrics, MetricUnit } from '@aws-lambda-powertools/metrics';

const metrics = new Metrics({
  namespace: 'MyApp',
  serviceName: 'checkout',
});

export const handler = async (event: any) => {
  const start = Date.now();
  metrics.addDimension('tenant', event.tenant ?? 'free');

  try {
    if (Math.random() < 0.1) throw new Error('flaky downstream');
    await new Promise(r => setTimeout(r, Math.random() * 500));

    metrics.addMetric('OrderProcessed', MetricUnit.Count, 1);
    metrics.addMetric('Latency', MetricUnit.Milliseconds, Date.now() - start);
    return { ok: true };
  } catch (err) {
    metrics.addMetric('OrderFailed', MetricUnit.Count, 1);
    throw err;
  } finally {
    metrics.publishStoredMetrics(); // flushes the EMF log line
  }
};
```

A single invocation writes:
```json
{"_aws":{"Timestamp":1712000000000,"CloudWatchMetrics":[{"Namespace":"MyApp","Dimensions":[["service","tenant"]],"Metrics":[{"Name":"Latency","Unit":"Milliseconds"}]}]},"service":"checkout","tenant":"free","Latency":342}
```

## Step 2: CDK stack with dashboard + metric math + composite alarm
> **Why:** Metric math lets an alarm compute `errorRate = errors / invocations * 100` instead of alarming on raw counts (which would miss a 50% error rate on low traffic). Composite alarms let you page only when multiple signals agree — drastically reducing noise.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import * as cw from 'aws-cdk-lib/aws-cloudwatch';
import * as cwa from 'aws-cdk-lib/aws-cloudwatch-actions';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as snssub from 'aws-cdk-lib/aws-sns-subscriptions';
import * as synthetics from 'aws-cdk-lib/aws-synthetics';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class CWAdvancedStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const fn = new NodejsFunction(this, 'Checkout', {
      entry: 'lambda/handler.ts',
      runtime: lambda.Runtime.NODEJS_20_X,
      logRetention: 7,
      environment: { POWERTOOLS_SERVICE_NAME: 'checkout' },
    });

    const fnUrl = fn.addFunctionUrl({ authType: lambda.FunctionUrlAuthType.NONE });

    const topic = new sns.Topic(this, 'Alerts');
    topic.addSubscription(new snssub.EmailSubscription('you@example.com'));

    // --- Metric math: error rate ---
    const invocations = fn.metricInvocations({ period: cdk.Duration.minutes(5) });
    const errors = fn.metricErrors({ period: cdk.Duration.minutes(5) });

    const errorRate = new cw.MathExpression({
      expression: '(errors / invocations) * 100',
      usingMetrics: { errors, invocations },
      label: 'ErrorRate %',
      period: cdk.Duration.minutes(5),
    });

    const highErrorAlarm = new cw.Alarm(this, 'HighErrorRate', {
      metric: errorRate,
      threshold: 5,
      evaluationPeriods: 2,
      treatMissingData: cw.TreatMissingData.NOT_BREACHING,
    });

    // --- Anomaly detection on invocation count ---
    const anomalyAlarm = new cw.CfnAlarm(this, 'TrafficAnomaly', {
      alarmName: 'TrafficAnomaly',
      comparisonOperator: 'LessThanLowerOrGreaterThanUpperThreshold',
      evaluationPeriods: 2,
      thresholdMetricId: 'ad1',
      metrics: [
        {
          id: 'm1',
          metricStat: {
            metric: { namespace: 'AWS/Lambda', metricName: 'Invocations',
              dimensions: [{ name: 'FunctionName', value: fn.functionName }] },
            period: 300, stat: 'Sum',
          }, returnData: true,
        },
        { id: 'ad1', expression: 'ANOMALY_DETECTION_BAND(m1, 2)', returnData: true },
      ],
    });

    // --- p95 Latency alarm (from EMF metric) ---
    const latencyAlarm = new cw.Alarm(this, 'HighLatency', {
      metric: new cw.Metric({
        namespace: 'MyApp', metricName: 'Latency',
        statistic: 'p95', period: cdk.Duration.minutes(5),
        dimensionsMap: { service: 'checkout' },
      }),
      threshold: 1000,
      evaluationPeriods: 2,
    });

    // --- Composite alarm ---
    const composite = new cw.CompositeAlarm(this, 'ServiceHealth', {
      alarmRule: cw.AlarmRule.anyOf(
        cw.AlarmRule.allOf(
          cw.AlarmRule.fromAlarm(highErrorAlarm, cw.AlarmState.ALARM),
          cw.AlarmRule.fromAlarm(latencyAlarm, cw.AlarmState.ALARM),
        ),
        cw.AlarmRule.fromAlarm(
          cw.Alarm.fromAlarmArn(this, 'AnomRef',
            `arn:aws:cloudwatch:${this.region}:${this.account}:alarm:TrafficAnomaly`),
          cw.AlarmState.ALARM),
      ),
      compositeAlarmName: 'CheckoutServiceHealth',
    });
    composite.addAlarmAction(new cwa.SnsAction(topic));

    // --- Dashboard ---
    new cw.Dashboard(this, 'Dash', {
      dashboardName: 'CheckoutAdvanced',
      widgets: [
        [new cw.GraphWidget({ title: 'Error Rate %', left: [errorRate] })],
        [new cw.GraphWidget({
          title: 'Latency p50/p95/p99',
          left: [
            new cw.Metric({ namespace: 'MyApp', metricName: 'Latency', statistic: 'p50' }),
            new cw.Metric({ namespace: 'MyApp', metricName: 'Latency', statistic: 'p95' }),
            new cw.Metric({ namespace: 'MyApp', metricName: 'Latency', statistic: 'p99' }),
          ],
        })],
        [new cw.SingleValueWidget({ title: 'Invocations', metrics: [invocations] })],
      ],
    });

    // --- Synthetics canary ---
    const canaryBucket = new s3.Bucket(this, 'CanaryArtifacts', {
      removalPolicy: cdk.RemovalPolicy.DESTROY, autoDeleteObjects: true,
    });

    new synthetics.Canary(this, 'Heartbeat', {
      schedule: synthetics.Schedule.rate(cdk.Duration.minutes(5)),
      test: synthetics.Test.custom({
        code: synthetics.Code.fromInline(`
const synthetics = require('Synthetics');
const log = require('SyntheticsLogger');
exports.handler = async () => {
  const page = await synthetics.getPage();
  const url = '${fnUrl.url}';
  const resp = await page.goto(url, { waitUntil: 'networkidle0', timeout: 30000 });
  if (!resp.ok()) throw new Error('Status ' + resp.status());
  await synthetics.takeScreenshot('ok', 'loaded');
  log.info('healthy');
};`),
        handler: 'index.handler',
      }),
      runtime: synthetics.Runtime.SYNTHETICS_NODEJS_PUPPETEER_7_0,
      artifactsBucketLocation: { bucket: canaryBucket },
    });

    new cdk.CfnOutput(this, 'FnUrl', { value: fnUrl.url });
  }
}
```

## Step 3: Deploy and load-test
> **Why:** Without traffic you see nothing. Generate invocations with a mix of success and failure so the error rate metric math has something to calculate on.

```bash
npm install @aws-lambda-powertools/metrics
cdk deploy

URL=$(aws cloudformation describe-stacks --stack-name CWAdvancedStack \
  --query "Stacks[0].Outputs[?OutputKey=='FnUrl'].OutputValue" --output text)

for i in {1..200}; do
  curl -s -X POST "$URL" -d "{\"tenant\":\"t$((i%3))\"}" > /dev/null &
done
wait
```

Expected: in the `CheckoutAdvanced` dashboard, after ~2 min, you see `Error Rate %` hovering near 10, a `Latency` p95 around 450ms, and invocations climb to 200.

## Step 4: Logs Insights scheduled digest
> **Why:** Most teams want a once-a-day roll-up instead of real-time noise. A scheduled query + SNS delivers a daily email without building infrastructure.

```typescript
import * as logs from 'aws-cdk-lib/aws-logs';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

const digestFn = new NodejsFunction(this, 'Digest', {
  entry: 'lambda/digest.ts',
  timeout: cdk.Duration.minutes(2),
  environment: { TOPIC_ARN: topic.topicArn, LOG_GROUP: fn.logGroup.logGroupName },
});
topic.grantPublish(digestFn);
fn.logGroup.grant(digestFn, 'logs:StartQuery', 'logs:GetQueryResults');

new events.Rule(this, 'DailyDigest', {
  schedule: events.Schedule.cron({ minute: '0', hour: '13' }), // 13:00 UTC
  targets: [new targets.LambdaFunction(digestFn)],
});
```

`lambda/digest.ts` runs the Insights query:
```typescript
filter @type = "REPORT"
| stats count(), avg(@duration), pct(@duration, 95) by bin(1d)
```
…and publishes the formatted result to SNS.

## Step 5: Contributor Insights on a log group
> **Why:** When a metric spikes, the first question is "which tenant / user / IP caused it?". Contributor Insights answers this without shipping logs to a separate tool.

```typescript
new cw.CfnInsightRule(this, 'TopErrorTenants', {
  ruleName: 'TopErrorTenants',
  ruleState: 'ENABLED',
  ruleBody: JSON.stringify({
    Schema: { Name: 'CloudWatchLogRule', Version: 1 },
    LogGroupNames: [fn.logGroup.logGroupName],
    LogFormat: 'JSON',
    Contribution: {
      Keys: ['$.tenant'],
      Filters: [{ Match: '$.OrderFailed', GreaterThan: 0 }],
    },
    AggregateOn: 'Count',
  }),
});
```

After a few minutes of traffic the rule's widget shows the top-N tenants by failure count.

## Cleanup
```bash
cdk destroy
# canary artifact bucket is auto-deleted (autoDeleteObjects: true)
# anomaly detector model auto-evicts 14 days after last evaluation
```

## Common Errors
- **EMF metric missing from namespace** — Powertools didn't flush. You forgot `metrics.publishStoredMetrics()` in `finally`, or you called it outside the handler scope.
- **Composite alarm `AlarmNotFound`** — referenced a CfnAlarm by ARN before it exists. Use L2 `Alarm` objects and `fromAlarm()` so CDK wires the dependency.
- **Canary stuck `FAILED` with `Cannot find module`** — wrong runtime version. Use `SYNTHETICS_NODEJS_PUPPETEER_7_0` or newer; older runtimes are deprecated.
- **Anomaly detection alarm always OK** — needs at least 2 weeks of history to train meaningfully. Use static thresholds for the first 14 days.
- **Metric math expression returns no data** — the referenced metrics used different periods. All metrics in a math expression must share the same period.
- **Insights rule `Contribution not valid`** — JSON log fields missing. Verify the log line actually contains the key (`$.tenant`) you reference.
