# Walkthrough — 06 EventBridge, SNS, SQS

## About this service
Three messaging services that solve different problems:
- **SQS (Simple Queue Service)** — pull-based queue. Producers write, consumers poll and delete. Buffers spikes, decouples services, lets you retry failed work. Standard (at-least-once, unordered, high throughput) or FIFO (exactly-once, ordered per group, 300 msg/s).
- **SNS (Simple Notification Service)** — push-based pub/sub. Topic fan-out to SQS, Lambda, HTTP(S), email, SMS. One publish → N subscribers.
- **EventBridge** — event bus with rule-based routing, pattern matching, cross-account delivery, schema registry, scheduling, replay, and integrations with dozens of SaaS providers.

**Why it matters:** synchronous systems don't survive traffic spikes or downstream failures. These three are the foundation of every async/event-driven architecture on AWS.

**When to use SQS:** a producer wants to hand work to a worker pool. Retries and DLQs included.
**When to use SNS:** pure fan-out with push semantics to many subscribers.
**When to use EventBridge:** anything with routing logic (`"source":"myapp","detail-type":"order.created"`), scheduled tasks, or cross-account.
**When NOT to use:** ultra-low-latency (<10ms) paths — use direct SDK calls or Kinesis. Ordering across millions of keys — use Kafka/MSK instead of FIFO SQS.

## Estimated cost
- **SQS: $0.40/M requests** (first 1M free each month)
- **SNS: $0.50/M publishes** + delivery cost (email free, SMS pricey)
- **EventBridge: $1.00/M events** (AWS-service events from the default bus are free)
- **EventBridge Scheduler: $1.00/M invocations**
- **Lambda invocations**: $0.20/M
- Total for this lesson at low volume: **<$1/month**.

---

## Step 1: Scaffold
> **Why:** All six exercises fit in one stack because the resources are small and mostly independent.

```bash
mkdir events && cd events
cdk init app --language=typescript
npm install aws-cdk-lib
```

## Step 2: SQS standard + DLQ + Lambda consumer
> **Why:** The default durable-work pattern. Producer writes; consumer polls in batches of 10; after 3 receives the message moves to a DLQ for investigation.

`lib/events-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subs from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as sources from 'aws-cdk-lib/aws-lambda-event-sources';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as scheduler from 'aws-cdk-lib/aws-scheduler';
import * as iam from 'aws-cdk-lib/aws-iam';

export class EventsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // --- Step 2: Standard queue + DLQ ---
    const dlq = new sqs.Queue(this, 'Dlq', { retentionPeriod: cdk.Duration.days(14) });
    const queue = new sqs.Queue(this, 'Q', {
      visibilityTimeout: cdk.Duration.seconds(60),
      deadLetterQueue: { queue: dlq, maxReceiveCount: 3 },
    });

    const consumer = new lambda.Function(this, 'Consumer', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      timeout: cdk.Duration.seconds(30),
      code: lambda.Code.fromInline(`
exports.handler = async (e) => {
  for (const r of e.Records) {
    console.log('msg', r.messageId, r.body);
    if (r.body.includes('poison')) throw new Error('poison pill');
  }
};`),
    });
    consumer.addEventSource(new sources.SqsEventSource(queue, { batchSize: 10, reportBatchItemFailures: true }));

    // --- Step 3: FIFO queue ---
    const fifo = new sqs.Queue(this, 'Fifo.fifo', {
      fifo: true, contentBasedDeduplication: true,
      visibilityTimeout: cdk.Duration.seconds(30),
    });

    // --- Step 4: SNS fan-out ---
    const topic = new sns.Topic(this, 'Topic');
    const qA = new sqs.Queue(this, 'FanA');
    const qB = new sqs.Queue(this, 'FanB');
    topic.addSubscription(new subs.SqsSubscription(qA));
    topic.addSubscription(new subs.SqsSubscription(qB, {
      filterPolicy: {
        eventType: sns.SubscriptionFilter.stringFilter({ allowlist: ['order.created'] }),
      },
    }));
    topic.addSubscription(new subs.EmailSubscription('you@example.com'));

    // --- Step 5: EventBridge default bus → Lambda on EC2 state change ---
    const ebFn = new lambda.Function(this, 'EbFn', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`exports.handler = async (e) => console.log('event', JSON.stringify(e));`),
    });
    new events.Rule(this, 'Ec2Rule', {
      eventPattern: { source: ['aws.ec2'], detailType: ['EC2 Instance State-change Notification'] },
      targets: [new targets.LambdaFunction(ebFn)],
    });

    // --- Step 6: Custom bus + archive ---
    const bus = new events.EventBus(this, 'AppBus');
    new events.Archive(this, 'Archive', {
      sourceEventBus: bus,
      eventPattern: { source: ['myapp'] },
      retention: cdk.Duration.days(7),
    });
    new events.Rule(this, 'OrderRule', {
      eventBus: bus,
      eventPattern: { source: ['myapp'], detailType: ['order.created'] },
      targets: [new targets.LambdaFunction(ebFn)],
    });

    // --- Step 7: EventBridge Scheduler ---
    const schedRole = new iam.Role(this, 'SchedRole', {
      assumedBy: new iam.ServicePrincipal('scheduler.amazonaws.com'),
    });
    ebFn.grantInvoke(schedRole);
    new scheduler.CfnSchedule(this, 'WeeklySched', {
      flexibleTimeWindow: { mode: 'OFF' },
      scheduleExpression: 'cron(0 9 ? * MON *)',
      scheduleExpressionTimezone: 'America/New_York',
      target: { arn: ebFn.functionArn, roleArn: schedRole.roleArn },
    });

    new cdk.CfnOutput(this, 'QueueUrl', { value: queue.queueUrl });
    new cdk.CfnOutput(this, 'FifoUrl',  { value: fifo.queueUrl });
    new cdk.CfnOutput(this, 'TopicArn', { value: topic.topicArn });
    new cdk.CfnOutput(this, 'BusName',  { value: bus.eventBusName });
  }
}
```

Deploy:
```bash
cdk deploy
```

Send messages, trigger DLQ routing:
```bash
aws sqs send-message --queue-url <q-url> --message-body '{"ok":true}'
aws sqs send-message --queue-url <q-url> --message-body 'poison'
# wait ~3 minutes; consumer retries the poison 3x, then it flips to DLQ
aws sqs receive-message --queue-url <dlq-url>
```

Expected CloudWatch log output for the consumer:
```
START RequestId: ...
msg 0123... poison
ERROR poison pill
END RequestId: ...
# 3 such attempts, then moves to DLQ
```

## Step 3: FIFO queue — ordering
> **Why:** Standard SQS is unordered; that breaks workflows like "process invoice before payment." FIFO preserves order per `MessageGroupId` and deduplicates within a 5-minute window.

```bash
for i in 1 2 3 4 5; do
  aws sqs send-message --queue-url <fifo-url> \
    --message-body "msg-$i" --message-group-id customer-42
done

aws sqs receive-message --queue-url <fifo-url> --max-number-of-messages 10 \
  --query 'Messages[].Body'
```

Expected output (stable order):
```
[ "msg-1", "msg-2", "msg-3", "msg-4", "msg-5" ]
```

Different `MessageGroupId` values can be processed in parallel; same group is strictly serial.

## Step 4: SNS fan-out with filter policies
> **Why:** Fan-out avoids N point-to-point integrations: one publish, many subscribers. Filter policies let subscribers opt in/out without bothering the publisher.

Confirm email subscription (check inbox), then publish:

```bash
aws sns publish --topic-arn <topic-arn> \
  --message '{"id":"1"}' \
  --message-attributes '{"eventType":{"DataType":"String","StringValue":"order.created"}}'
# → qA receives, qB receives (matches filter), email receives

aws sns publish --topic-arn <topic-arn> \
  --message '{"id":"2"}' \
  --message-attributes '{"eventType":{"DataType":"String","StringValue":"user.login"}}'
# → qA receives (no filter), qB does NOT (filter mismatch), email receives
```

Verify:
```bash
aws sqs receive-message --queue-url <qA-url> --max-number-of-messages 10 --query 'length(Messages)'
# 2
aws sqs receive-message --queue-url <qB-url> --max-number-of-messages 10 --query 'length(Messages)'
# 1
```

## Step 5: EventBridge default bus rule
> **Why:** Every AWS service emits events to the default bus for free. A three-line rule turns "EC2 started" into a Lambda trigger without polling.

```bash
aws ec2 run-instances --image-id ami-xxxx --instance-type t3.micro --max-count 1 --min-count 1
# within ~30s the EbFn log shows the event
aws logs tail /aws/lambda/EventsStack-EbFn... --since 2m
```

## Step 6: Custom bus + archive + replay
> **Why:** Custom buses isolate your app's events from AWS service noise. Archive lets you replay historical events into the bus — invaluable for reprocessing after a consumer bug.

Publish:
```bash
aws events put-events --entries '[{
  "EventBusName":"<bus-name>",
  "Source":"myapp",
  "DetailType":"order.created",
  "Detail":"{\"orderId\":\"o-1\",\"amount\":42}"
}]'
```

Replay last hour into the bus:
```bash
aws events start-replay --replay-name demo \
  --event-source-arn <archive-arn> \
  --destination '{"Arn":"<bus-arn>","FilterArns":["<order-rule-arn>"]}' \
  --event-start-time $(date -u -d '-1 hour' +%s) \
  --event-end-time   $(date -u +%s)
```

## Step 7: EventBridge Scheduler (cron)
> **Why:** Scheduler replaces the old EventBridge rate/cron rules with a dedicated, higher-scale service. Supports per-schedule timezones (no more UTC math), flexible time windows, and one-time schedules.

Already defined (`cron(0 9 ? * MON *)` New York time). Verify:
```bash
aws scheduler list-schedules --query 'Schedules[].[Name,State,ScheduleExpression]' --output table
```

Expected:
```
---------------------------------------------------------
|  WeeklySched  | ENABLED  | cron(0 9 ? * MON *)        |
---------------------------------------------------------
```

## Cleanup
> **Why:** Cheap to leave but nothing useful running.

```bash
cdk destroy
# Scheduler and Archive may need explicit retention override if they refuse to delete:
aws scheduler delete-schedule --name WeeklySched
aws events delete-archive --archive-name Archive
```

## Common Errors
- **Consumer keeps reprocessing same message** → visibility timeout shorter than Lambda runtime. Set `visibilityTimeout >= Lambda.timeout * 6`.
- **FIFO throughput cap of 300 msg/s** → enable high-throughput mode (`deduplicationScope: MESSAGE_GROUP`, `fifoThroughputLimit: PER_MESSAGE_GROUP_ID`).
- **Email subscription never delivers** → unconfirmed. Click the link in the first email SNS sent.
- **EventBridge rule doesn't fire** → pattern mismatch; test with `aws events test-event-pattern`.
- **"EventSize exceeds 256KB"** → put the payload in S3 and send a pointer event instead.
- **Scheduler says "target not invokable"** → role missing `lambda:InvokeFunction` on the target ARN.
- **DLQ fills silently** → no alarm. Add CloudWatch alarm on `ApproximateNumberOfMessagesVisible > 0` for every DLQ.
