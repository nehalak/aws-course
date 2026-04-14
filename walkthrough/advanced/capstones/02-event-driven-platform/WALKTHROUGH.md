# Walkthrough — Capstone 02 Event-Driven Platform

## About this capstone
You will build an order-processing platform modeled on what Amazon, Shopify, and DoorDash run internally: an EventBridge custom bus as the nervous system, five Fargate microservices publishing domain events, a Step Functions saga orchestrating the fulfillment workflow with compensations, SQS FIFO for notifications, and EventBridge Pipes wiring DynamoDB streams into downstream consumers. This capstone synthesizes everything about async messaging, idempotency, exactly-once semantics (or the pragmatic lack thereof), and distributed transactions.

**Why it matters:** Monolithic order flows become the bottleneck of any e-commerce business past a certain scale. The event-driven pattern you are building is what lets a company add a new service (fraud detection, loyalty points, email preferences) without editing the order service. Getting the saga right is what lets you refund the customer when the warehouse is out of stock after the payment was captured — the single hardest distributed-systems problem most backend teams face.

**Prerequisites:**
- `intermediate/eventbridge` — rules, targets, buses.
- `intermediate/sqs` — FIFO, DLQ, visibility timeout.
- `intermediate/step-functions` — saga pattern, `Parallel`, `Catch`.
- `intermediate/fargate` — task definitions, service discovery.
- `intermediate/dynamodb-streams` — stream ARNs.
- `advanced/observability` — X-Ray, structured logging, correlation IDs.

## Estimated cost
- EventBridge: $1 per million custom events + archive $0.10/GB-mo.
- Schema Registry: first 5M events/mo free.
- Fargate (5 services × 0.25 vCPU / 0.5 GB, always on): ~$9/task/mo × 5 = **$45/mo**.
- NAT Gateway (required for Fargate pulling public images + calling EB): $32/mo + $0.045/GB.
- Step Functions Standard: $0.025 / 1k transitions — a saga per order = ~7 transitions.
- SQS FIFO: $0.50 per million requests.
- DynamoDB on-demand + streams: ~$5/mo idle, more under load.
- X-Ray: $5 / million traces.
- Total for this capstone: **~$90–130/month idle** driven mostly by Fargate + NAT. **WARN:** leave 5 Fargate tasks running for a weekend and the NAT + tasks will eat $30+ alone. **Destroy after each session — or scale services to `desiredCount:0` between sessions.**

---

## Architecture

```
  +-----------+   +-----------+   +-----------+   +-----------+   +-----------+
  |  Order    |   |  Payment  |   | Inventory |   | Shipping  |   |  Notify   |
  | (Fargate) |   | (Fargate) |   | (Fargate) |   | (Fargate) |   | (Fargate) |
  +-----+-----+   +-----+-----+   +-----+-----+   +-----+-----+   +-----+-----+
        |               |               |               |               ^
        v  publish      v               v               v               |
  +------------------------------------------------------------------+  |
  |                EventBridge custom bus: "orders"                  |--+ (SQS FIFO target)
  |   - archive (30d replay)    - schema registry    - rules/targets|
  +------------------------------------------------------------------+
        |
        v  rule "OrderCreated"
  +------------------------+         +------------------+
  | Step Functions saga    | <------ | DynamoDB Streams |
  |  (fulfill order)       |  Pipes  | orders table     |
  +------------------------+         +------------------+
```

## Step 1: CDK project layout
> **Why:** Five services + a bus + a saga + pipes is too much for one stack. Split by lifecycle: the bus almost never changes, services change weekly, the saga changes daily.

```
orders-platform/
├── bin/app.ts
├── lib/
│   ├── bus-stack.ts          # EventBridge bus, archive, schema registry
│   ├── network-stack.ts      # VPC, NAT, service discovery
│   ├── data-stack.ts         # DynamoDB orders + streams
│   ├── services-stack.ts     # 5 Fargate services
│   ├── saga-stack.ts         # Step Functions + Pipes
│   └── notify-stack.ts       # SQS FIFO + DLQ
├── services/
│   ├── order/{Dockerfile,src/}
│   ├── payment/...
│   ├── inventory/...
│   ├── shipping/...
│   └── notify/...
└── cdk.json
```

```bash
npx cdk init app --language typescript
npm i aws-cdk-lib constructs
npm i @aws-sdk/client-eventbridge @aws-sdk/client-dynamodb
```

## Step 2: Bus + archive + schema registry
> **Why:** The archive lets you replay events — priceless when debugging a saga bug in production. The schema registry enforces backward-compatible contracts between services; without it, the payment team deploys a breaking change on Thursday and takes down the order team on Friday.

```typescript
// lib/bus-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as events from 'aws-cdk-lib/aws-events';
import { CfnDiscoverer, CfnRegistry } from 'aws-cdk-lib/aws-eventschemas';
import { Construct } from 'constructs';

export class BusStack extends cdk.Stack {
  public readonly bus: events.EventBus;
  constructor(scope: Construct, id: string) {
    super(scope, id);

    this.bus = new events.EventBus(this, 'OrdersBus', { eventBusName: 'orders' });

    new events.Archive(this, 'Archive', {
      sourceEventBus: this.bus,
      archiveName: 'orders-30d',
      retention: cdk.Duration.days(30),
      eventPattern: { source: events.Match.prefix('orders.') },
    });

    const registry = new CfnRegistry(this, 'Registry', { registryName: 'orders-registry' });
    new CfnDiscoverer(this, 'Discoverer', {
      sourceArn: this.bus.eventBusArn,
      description: 'auto-discover schemas on orders bus',
    });
  }
}
```

## Step 3: Network + data
> **Why:** Fargate services need a VPC with egress (for ECR pulls + EventBridge calls). You could use interface endpoints to skip NAT but that adds $8/endpoint/month — for five endpoints you break even at ~50 GB of NAT egress, so for a capstone stick with one NAT.

```typescript
// lib/network-stack.ts (excerpt)
const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });
const cluster = new ecs.Cluster(this, 'Cluster', { vpc });
new servicediscovery.PrivateDnsNamespace(this, 'Ns', { name: 'orders.local', vpc });

// lib/data-stack.ts
const table = new ddb.Table(this, 'Orders', {
  partitionKey: { name: 'orderId', type: ddb.AttributeType.STRING },
  stream: ddb.StreamViewType.NEW_AND_OLD_IMAGES,
  billingMode: ddb.BillingMode.PAY_PER_REQUEST,
});
```

## Step 4: Services as Fargate tasks
> **Why:** Each service gets its own IAM role with only the EventBridge `PutEvents` and DynamoDB permissions it needs. Service boundaries are IAM boundaries — that is what makes "microservices" actually micro.

```typescript
// lib/services-stack.ts (one service; repeat for 5)
const orderTd = new ecs.FargateTaskDefinition(this, 'OrderTd', { cpu: 256, memoryLimitMiB: 512 });
orderTd.addContainer('app', {
  image: ecs.ContainerImage.fromAsset('services/order'),
  logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'order' }),
  environment: { BUS_NAME: bus.eventBusName, TABLE: table.tableName },
});
bus.grantPutEventsTo(orderTd.taskRole);
table.grantReadWriteData(orderTd.taskRole);

new ecs.FargateService(this, 'OrderSvc', {
  cluster, taskDefinition: orderTd, desiredCount: 1,
  cloudMapOptions: { name: 'order' },
});
```

Each service emits events in the canonical shape:

```json
{
  "source": "orders.payment",
  "detail-type": "PaymentAuthorized",
  "detail": { "orderId": "o_123", "amount": 4200, "correlationId": "c_abc" }
}
```

## Step 5: Saga — Step Functions fulfillment
> **Why:** The saga is a long-running state machine that reserves inventory, authorizes payment, creates the shipment, and — critically — runs compensations in reverse if any step fails. You could do this with choreography (services listening to each other) but with 5+ services you lose visibility; orchestration makes the flow auditable.

```typescript
// lib/saga-stack.ts (excerpt)
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';

const reserveInv = new tasks.EventBridgePutEvents(this, 'ReserveInv', {
  entries: [{ detailType: 'ReserveInventory', detail: sfn.TaskInput.fromJsonPathAt('$'),
              source: 'orders.saga', eventBus: bus }],
  resultPath: '$.invRes',
});

const authPay   = new tasks.EventBridgePutEvents(this, 'AuthPayment', { /* ... */ });
const ship      = new tasks.EventBridgePutEvents(this, 'Ship', { /* ... */ });

// compensations
const releaseInv = new tasks.EventBridgePutEvents(this, 'ReleaseInv', { /* ... */ });
const refund     = new tasks.EventBridgePutEvents(this, 'Refund', { /* ... */ });

const failed = new sfn.Fail(this, 'Failed');

const chain = reserveInv
  .addCatch(failed, { resultPath: '$.err' })
  .next(authPay.addCatch(releaseInv.next(failed), { resultPath: '$.err' }))
  .next(ship.addCatch(refund.next(releaseInv).next(failed), { resultPath: '$.err' }))
  .next(new sfn.Succeed(this, 'Done'));

const sm = new sfn.StateMachine(this, 'FulfillSaga', {
  definition: chain, stateMachineType: sfn.StateMachineType.STANDARD,
});

// wire bus rule: OrderCreated -> start saga
new events.Rule(this, 'OnOrderCreated', {
  eventBus: bus,
  eventPattern: { source: ['orders.order'], detailType: ['OrderCreated'] },
  targets: [new targets.SfnStateMachine(sm)],
});
```

**Critical:** services are async. The saga publishes `ReserveInventory` — the inventory service consumes it, does work, and publishes `InventoryReserved`. To wait for the response inside the saga, use the **callback pattern** (`.waitForTaskToken`) with a task token threaded through the event; otherwise use a longer chain of rules.

## Step 6: EventBridge Pipes — streams → SQS
> **Why:** Pipes eliminate the Lambda that used to sit between DynamoDB streams and an SQS queue. Less code, per-event filtering, and built-in retry. This is 2024+ AWS idiom; Lambda-as-glue is legacy.

```typescript
// lib/saga-stack.ts (pipes)
import * as pipes from 'aws-cdk-lib/aws-pipes';

new pipes.CfnPipe(this, 'OrdersStreamPipe', {
  roleArn: pipeRole.roleArn,
  source: table.tableStreamArn!,
  sourceParameters: {
    dynamoDbStreamParameters: { startingPosition: 'LATEST', batchSize: 10 },
    filterCriteria: {
      filters: [{ pattern: JSON.stringify({ eventName: ['INSERT', 'MODIFY'] }) }],
    },
  },
  target: notifyFifo.queueArn,
  targetParameters: {
    sqsQueueParameters: { messageGroupId: '$.dynamodb.Keys.orderId.S' },
    inputTemplate: '{"orderId": <$.dynamodb.NewImage.orderId.S>, "status": <$.dynamodb.NewImage.status.S>}',
  },
});
```

## Step 7: Notifications with SQS FIFO + DLQ
> **Why:** Emails must go out in order per customer (confirmation before shipped). FIFO with `MessageGroupId = orderId` preserves per-order ordering without serializing across customers. DLQ catches poison messages so the main queue keeps draining.

```typescript
const dlq = new sqs.Queue(this, 'NotifyDlq', { fifo: true, contentBasedDeduplication: true });
const notifyFifo = new sqs.Queue(this, 'NotifyFifo', {
  fifo: true, contentBasedDeduplication: true,
  visibilityTimeout: cdk.Duration.seconds(60),
  deadLetterQueue: { queue: dlq, maxReceiveCount: 5 },
});
```

## Step 8: Idempotency + correlation
> **Why:** EventBridge is **at-least-once**. If your inventory service decrements stock every time it receives an event you will oversell. Every consumer MUST dedupe on `detail.idempotencyKey` (or event ID) via a conditional `PutItem`.

```typescript
// inside inventory service handler
await ddb.put({
  TableName: 'inv-idem',
  Item: { key: event.id, ttl: now + 86400 },
  ConditionExpression: 'attribute_not_exists(#k)',
  ExpressionAttributeNames: { '#k': 'key' },
}).promise().catch(e => { if (e.name === 'ConditionalCheckFailedException') return; throw e; });
```

Also: propagate `correlationId` through every event so X-Ray traces one order end-to-end.

## Step 9: Deploy + seed
> **Why:** Order matters — bus first, network+data, then services, then saga.

```bash
npx cdk deploy BusStack NetworkStack DataStack
npx cdk deploy ServicesStack
npx cdk deploy SagaStack NotifyStack
```

## Step 10: Happy-path load test
> **Why:** You need to prove 1,000 orders/sec stays under a sensible latency budget before claiming done.

```bash
# artillery, posting to order service via NLB
artillery quick --count 1000 --num 10 --rate 100 https://order.example.com/orders

# observe
aws stepfunctions list-executions --state-machine-arn $SM --status-filter SUCCEEDED --max-items 50
```

Target: e2e p95 latency (OrderCreated → OrderShipped event on bus) < 5s at 1k rps.

## Step 11: Chaos test — kill payment service mid-saga
> **Why:** If you have not verified compensation works under failure, you do not have a saga — you have optimistic code.

```bash
# scale payment service to 0 while a flight of orders is in-flight
aws ecs update-service --cluster $CL --service payment --desired-count 0

# send 50 orders
...

# a few minutes later, expect:
# - saga executions in FAILED state with ReleaseInventory visible in the timeline
# - inventory DynamoDB counter back to original level
```

## Step 12: Replay from archive
> **Why:** Fix the bug, replay the bad window against the (fixed) consumer, data converges. This is the operational superpower of event-driven.

```bash
aws events start-replay --replay-name fix-2026-04-14 \
  --event-source-arn $ARCHIVE_ARN \
  --event-start-time 2026-04-14T10:00:00Z \
  --event-end-time   2026-04-14T11:00:00Z \
  --destination '{"Arn":"'$BUS_ARN'","FilterArns":["'$RULE_ARN'"]}'
```

## Verification / success criteria
- Happy path: 1,000 orders → ≥ 995 `OrderShipped` events within 5s p95.
- Chaos: 0 orphaned inventory reservations 5 min after payment service kill.
- DLQ: injected poison message ends up in notify-DLQ after 5 receives, main queue keeps draining.
- Schema registry: introducing a breaking schema change on `PaymentAuthorized` fails the discoverer check.
- X-Ray service map shows all 5 services + Step Functions + DynamoDB with one trace per `correlationId`.

## Cleanup
```bash
# drain queues first, then
for svc in order payment inventory shipping notify; do
  aws ecs update-service --cluster $CL --service $svc --desired-count 0
done
npx cdk destroy --all
# empty + delete the archive manually if needed
```

## Common Errors
- **Events published but no rule fires** → `source` or `detail-type` mismatch (case sensitive), or rule on default bus instead of custom bus.
- **Saga stuck in `RUNNING` forever** → you used `.waitForTaskToken` but the consumer never calls `SendTaskSuccess`; check the token plumbing.
- **Double decrements in inventory** → idempotency table missing or TTL expired before duplicate arrived; widen TTL or use the event `id`.
- **SQS FIFO `MessageGroupId` not found** → Pipes input template referenced a missing JSON path; test with `InputTemplate` + `aws pipes test-pipe`.
- **Fargate tasks die with `CannotPullContainerError`** → NAT or VPC endpoint misconfigured; subnets must have a route to 0.0.0.0/0 via NAT.
- **`ThrottlingException` on EventBridge** → default limit 10k PutEvents/s per region; request quota raise or batch.
- **Replay targets original time** → replays re-fire events with their original timestamps; consumers relying on "now" need to be idempotent and time-agnostic.
- **Schema registry discoverer off** → schemas not auto-inferred; enable `State: STARTED` or pay no attention to schema drift.
