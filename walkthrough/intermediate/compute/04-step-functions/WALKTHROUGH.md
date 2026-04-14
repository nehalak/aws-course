# Walkthrough — 04 Step Functions

## About this service
**AWS Step Functions** orchestrates AWS services as a state machine. You write a flow (`Task → Choice → Parallel → Map`) in Amazon States Language (or CDK `sfn.Chain` for readability); SFN handles retries, timeouts, error routing, and execution history. Two flavors: **Standard** (durable, 1-year max, $25 per 1M state transitions) and **Express** (high-volume short, $1 per 1M + duration).

**Why it matters:** Any workflow with >3 steps, retries, or human-in-the-loop becomes a mess of Lambdas-calling-Lambdas. SFN replaces that ad-hoc glue with a visual graph, built-in error handling, and full audit log.

**When to use:** multi-step order/payment flows, ETL with branching, long-running approvals, sagas with compensation, fan-out/fan-in over arrays.
**When NOT to use:** simple single-Lambda jobs, ultra-low-latency APIs (transition overhead ~25ms Standard), high-volume event fan-out (use EventBridge/SQS).

## Estimated cost
- **Standard workflow: $25 per 1M state transitions** (each Task/Choice/etc counts)
- **Express workflow: $1 per 1M executions** + $0.00001667 per GB-second
- **Lambda invocations, DynamoDB writes, SNS publishes**: billed normally
- Example: 10,000 executions × 8 transitions/day × 30 days = 2.4M transitions = $60/mo Standard
- Total for this lesson: **~$5/month** (low-volume lab)

---

## Step 1: Project setup
> **Why:** CDK's `aws-stepfunctions-tasks` has first-class integrations for 200+ AWS services. You almost never write raw ASL JSON anymore.

```bash
mkdir step-functions && cd step-functions
cdk init app --language=typescript
npm install aws-cdk-lib constructs @types/aws-lambda
```

## Step 2: Linear flow (3 Lambdas)
> **Why:** The simplest orchestration: validate → process → notify. If any step fails, execution stops and the failure is visible in the console with the full input/output of each state.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import * as path from 'path';

export class StepFunctionsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const mk = (name: string) => new NodejsFunction(this, name, {
      entry: path.join(__dirname, `../src/${name}.ts`),
      runtime: lambda.Runtime.NODEJS_20_X,
      logRetention: 7,
    });

    const validate = mk('validate');
    const process = mk('process');
    const notify = mk('notify');

    const chain = new tasks.LambdaInvoke(this, 'Validate', { lambdaFunction: validate, outputPath: '$.Payload' })
      .next(new tasks.LambdaInvoke(this, 'Process', { lambdaFunction: process, outputPath: '$.Payload' }))
      .next(new tasks.LambdaInvoke(this, 'Notify', { lambdaFunction: notify, outputPath: '$.Payload' }));

    new sfn.StateMachine(this, 'Linear', {
      definitionBody: sfn.DefinitionBody.fromChainable(chain),
      stateMachineType: sfn.StateMachineType.STANDARD,
      timeout: cdk.Duration.minutes(5),
    });
  }
}
```

Start:
```bash
aws stepfunctions start-execution --state-machine-arn <arn> --input '{"orderId":"o-1"}'
aws stepfunctions describe-execution --execution-arn <execArn>
# "status": "SUCCEEDED"
```

## Step 3: Choice + Parallel fan-out
> **Why:** Real workflows branch. `Choice` routes on input; `Parallel` runs N branches concurrently and collects their results into an array. The Parallel state fails if any branch fails (unless caught).

```typescript
const premium = new tasks.LambdaInvoke(this, 'Premium', { lambdaFunction: process });
const standard = new tasks.LambdaInvoke(this, 'Standard', { lambdaFunction: process });

const parallel = new sfn.Parallel(this, 'Fanout')
  .branch(new tasks.LambdaInvoke(this, 'Email', { lambdaFunction: notify }))
  .branch(new tasks.LambdaInvoke(this, 'SMS', { lambdaFunction: notify }))
  .branch(new tasks.LambdaInvoke(this, 'Push', { lambdaFunction: notify }));

const choice = new sfn.Choice(this, 'Tier')
  .when(sfn.Condition.stringEquals('$.tier', 'premium'), premium.next(parallel))
  .otherwise(standard.next(parallel));
```

## Step 4: Map state (iterate + concurrency)
> **Why:** When input has an array (`items: [...]`), Map runs the inner workflow for each item. `maxConcurrency: 10` throttles to avoid hammering downstream services.

```typescript
const mapState = new sfn.Map(this, 'PerItem', {
  itemsPath: '$.items',
  maxConcurrency: 10,
  resultPath: '$.results',
});
mapState.itemProcessor(
  new tasks.LambdaInvoke(this, 'ProcessItem', { lambdaFunction: process })
);
```

Input `{ "items": [{"id":1},{"id":2},...,{"id":100}] }` → 10 parallel workers until all 100 done.

## Step 5: Retry + Catch
> **Why:** Transient failures (rate limit, 503) should auto-retry with exponential backoff. Permanent failures should route to a fallback/DLQ state. Without this, one flake fails the whole execution.

```typescript
const process2 = new tasks.LambdaInvoke(this, 'ProcessWithRetry', {
  lambdaFunction: process,
  outputPath: '$.Payload',
}).addRetry({
  errors: ['Lambda.TooManyRequestsException', 'States.Timeout'],
  interval: cdk.Duration.seconds(2),
  maxAttempts: 3,
  backoffRate: 2.0,
}).addCatch(
  new sfn.Pass(this, 'Fallback', { result: sfn.Result.fromObject({ fallback: true }) }),
  { errors: ['States.ALL'], resultPath: '$.error' }
);
```

Induce failure → see retries in execution history with 2s, 4s, 8s gaps.

## Step 6: Saga (compensating transactions)
> **Why:** In distributed systems you can't 2PC. The saga pattern does forward steps; on failure, runs compensating actions in reverse to undo work. Example: book flight → charge card → confirm. If confirm fails, refund card, then cancel flight.

```typescript
const bookFlight = new tasks.LambdaInvoke(this, 'BookFlight', { lambdaFunction: process, resultPath: '$.booking' });
const chargeCard = new tasks.LambdaInvoke(this, 'ChargeCard', { lambdaFunction: process, resultPath: '$.charge' });
const confirm = new tasks.LambdaInvoke(this, 'Confirm', { lambdaFunction: process });

const cancelFlight = new tasks.LambdaInvoke(this, 'CancelFlight', { lambdaFunction: process });
const refund = new tasks.LambdaInvoke(this, 'Refund', { lambdaFunction: process });
const failState = new sfn.Fail(this, 'SagaFailed');

confirm.addCatch(refund.next(cancelFlight).next(failState), { errors: ['States.ALL'] });
chargeCard.addCatch(cancelFlight.next(failState), { errors: ['States.ALL'] });

const saga = bookFlight.next(chargeCard).next(confirm);
```

## Step 7: Direct AWS SDK integrations (no Lambda)
> **Why:** Tasks like DynamoDB PutItem or SNS Publish don't need a Lambda wrapper. Direct integrations are cheaper and faster — one less hop.

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as sns from 'aws-cdk-lib/aws-sns';

const table = new dynamodb.Table(this, 'T', { partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING } });
const topic = new sns.Topic(this, 'Topic');

const putItem = new tasks.DynamoPutItem(this, 'Save', {
  table,
  item: { pk: tasks.DynamoAttributeValue.fromString(sfn.JsonPath.stringAt('$.id')) },
});
const publish = new tasks.SnsPublish(this, 'Notify', {
  topic,
  message: sfn.TaskInput.fromJsonPathAt('$'),
});

new sfn.StateMachine(this, 'Direct', {
  definitionBody: sfn.DefinitionBody.fromChainable(putItem.next(publish)),
});
```

## Step 8: Callback pattern (`waitForTaskToken`)
> **Why:** For human approval or external system integration, pause the workflow and resume when someone POSTs back. SFN holds execution state up to 1 year (Standard).

```typescript
const waitForApproval = new tasks.SnsPublish(this, 'AskApproval', {
  topic,
  integrationPattern: sfn.IntegrationPattern.WAIT_FOR_TASK_TOKEN,
  message: sfn.TaskInput.fromObject({
    token: sfn.JsonPath.taskToken,
    orderId: sfn.JsonPath.stringAt('$.orderId'),
  }),
});
```

External service sends approval:
```bash
aws stepfunctions send-task-success --task-token <token> --output '{"approved":true}'
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **`States.TaskFailed` with no detail** → Lambda errored but didn't serialize error; inspect CloudWatch Logs, not just SFN history.
- **`States.DataLimitExceeded`** → payload between states > 256KB. Use S3 + pass a reference.
- **Express workflow runs forever charges surprise** → Express duration component can dwarf per-exec cost. Set `timeout`.
- **Map concurrency higher than expected** → old `Map` state (legacy) uses different semantics than `DISTRIBUTED` Map; check `maxConcurrency` + `itemProcessor` config.
- **`The state machine definition has state X that is not reachable`** → forgot to connect a `Choice.otherwise()` branch to terminal state.
