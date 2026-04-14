# Walkthrough — 01 X-Ray

## About this service
**AWS X-Ray** is a distributed tracing service. It records **segments** (work done by a single service) and **subsegments** (calls to downstream services like DynamoDB, HTTP APIs, or other Lambdas). The **service map** visualizes topology and latency between services, and **traces** show the full timeline of a single request across all hops. Sampling rules control how many requests you trace (tracing every request is expensive and unnecessary).

**Why it matters:** in a microservices or serverless world, a single request touches many services. Logs only show you one hop at a time. Traces show you the whole chain, where latency is spent, and where errors originate.

**When to use:** API Gateway + Lambda + DynamoDB/SQS/SNS stacks, Step Functions, ECS/EKS services calling each other, anything with >2 hops per request.
**When NOT to use:** single-Lambda apps with no downstream calls (logs are enough), extremely high-throughput systems where even 1% sampling is costly (use OTel with tail-sampling instead), apps where you need long-term trace retention (X-Ray only keeps 30 days — export to S3/Honeycomb).

## Estimated cost
- **Traces recorded: $5.00 per 1M traces** (first 100k/month free)
- **Traces retrieved/scanned: $0.50 per 1M**
- **Sampling at 5%** on 10M requests/month = 500k traces = **~$2/month**
- Lambda tracing overhead: negligible (<1ms per invocation)
- Total for this lesson: **~$1/month** (sits in free tier for low-traffic learning)

---

## Step 1: Lambda instrumented with Powertools Tracer
> **Why:** Powertools Tracer wraps the X-Ray SDK and auto-captures AWS SDK calls, cold starts, and handler errors. Much less boilerplate than the raw `aws-xray-sdk`. Under the hood it uses the X-Ray daemon that runs inside the Lambda execution environment.

`lambda/handler.ts`:
```typescript
import { Tracer } from '@aws-lambda-powertools/tracer';
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';

const tracer = new Tracer({ serviceName: 'orders-api' });
const ddb = tracer.captureAWSv3Client(new DynamoDBClient({}));

export const handler = async (event: any) => {
  const segment = tracer.getSegment();
  const sub = segment?.addNewSubsegment('## business-logic');
  tracer.setSegment(sub!);

  try {
    tracer.putAnnotation('userId', event.userId ?? 'anon');
    const res = await ddb.send(new GetItemCommand({
      TableName: process.env.TABLE!,
      Key: { pk: { S: event.userId ?? 'anon' } },
    }));

    // external HTTP call — also traced automatically by Powertools
    const r = await fetch('https://httpbin.org/get');
    tracer.putMetadata('httpStatus', r.status);

    return { statusCode: 200, body: JSON.stringify({ item: res.Item }) };
  } catch (err) {
    tracer.addErrorAsMetadata(err as Error);
    throw err;
  } finally {
    sub?.close();
    tracer.setSegment(segment!);
  }
};
```

## Step 2: CDK stack with tracing enabled
> **Why:** `tracing: Tracing.ACTIVE` on the Lambda and `tracingEnabled: true` on API Gateway are the two flags that wire everything up. Without them, segments are dropped even if the SDK is instrumented. The Lambda also needs `AWSXRayDaemonWriteAccess` (CDK adds this automatically when `tracing` is set).

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import * as apigw from 'aws-cdk-lib/aws-apigateway';
import * as ddb from 'aws-cdk-lib/aws-dynamodb';
import * as xray from 'aws-cdk-lib/aws-xray';

export class XRayStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const table = new ddb.Table(this, 'Orders', {
      partitionKey: { name: 'pk', type: ddb.AttributeType.STRING },
      billingMode: ddb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    const fn = new NodejsFunction(this, 'OrdersFn', {
      entry: 'lambda/handler.ts',
      runtime: lambda.Runtime.NODEJS_20_X,
      tracing: lambda.Tracing.ACTIVE,
      environment: {
        TABLE: table.tableName,
        POWERTOOLS_SERVICE_NAME: 'orders-api',
      },
      logRetention: 7,
    });
    table.grantReadData(fn);

    const api = new apigw.RestApi(this, 'Api', {
      deployOptions: { tracingEnabled: true, stageName: 'prod' },
    });
    api.root.addResource('orders').addMethod('GET', new apigw.LambdaIntegration(fn));
    api.root.addResource('health').addMethod('GET', new apigw.LambdaIntegration(fn));
    api.root.addResource('checkout').addMethod('POST', new apigw.LambdaIntegration(fn));

    // Step 3: custom sampling rule — 100% of /checkout, 1% of /health
    new xray.CfnSamplingRule(this, 'CheckoutRule', {
      samplingRule: {
        ruleName: 'checkout-full',
        priority: 100,
        fixedRate: 1.0,
        reservoirSize: 1,
        serviceName: '*',
        serviceType: '*',
        host: '*',
        httpMethod: '*',
        urlPath: '/checkout*',
        resourceArn: '*',
        version: 1,
      },
    });
    new xray.CfnSamplingRule(this, 'HealthRule', {
      samplingRule: {
        ruleName: 'health-low',
        priority: 200,
        fixedRate: 0.01,
        reservoirSize: 0,
        serviceName: '*',
        serviceType: '*',
        host: '*',
        httpMethod: '*',
        urlPath: '/health*',
        resourceArn: '*',
        version: 1,
      },
    });

    new cdk.CfnOutput(this, 'ApiUrl', { value: api.url });
  }
}
```

## Step 3: Deploy and generate traffic
> **Why:** X-Ray only displays what was sampled. You need a handful of invocations before the service map populates. The console lags ~30s behind real requests.

```bash
npm install @aws-lambda-powertools/tracer @aws-sdk/client-dynamodb
cdk deploy

API=$(aws cloudformation describe-stacks --stack-name XRayStack \
  --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text)

for i in {1..50}; do curl -s "${API}orders?userId=u$i" > /dev/null; done
for i in {1..100}; do curl -s "${API}health" > /dev/null; done
curl -s -X POST "${API}checkout"
```

Expected output in the X-Ray console (CloudWatch > X-Ray traces > Service map):
```
[client] -> [API Gateway: Api/prod] -> [AWS::Lambda: OrdersFn] -> [AWS::DynamoDB::Table: Orders]
                                                              -> [httpbin.org]
```

## Step 4: Query traces with filter expressions
> **Why:** Raw traces are noisy. Filter expressions narrow to slow or error traces — the ones you actually care about. Learn these; they're the equivalent of Logs Insights for tracing.

In the X-Ray console, run:
```
service("orders-api") { responsetime > 1 }
service("orders-api") { error OR fault }
annotation.userId = "u5"
```

Expected: list of trace IDs, each with a timeline showing segment/subsegment breakdown. Click one trace to see `## business-logic`, `DynamoDB`, and `httpbin.org` subsegments with their individual durations.

## Step 5: Add a second Lambda and propagate trace context
> **Why:** The whole point of distributed tracing is seeing across services. Trace context (the `X-Amzn-Trace-Id` header) must propagate. AWS SDK calls propagate automatically; raw `fetch` does too when the X-Ray SDK is active.

Add to the stack:
```typescript
const downstream = new NodejsFunction(this, 'Enricher', {
  entry: 'lambda/enricher.ts',
  tracing: lambda.Tracing.ACTIVE,
  logRetention: 7,
});
downstream.grantInvoke(fn);
fn.addEnvironment('ENRICHER_NAME', downstream.functionName);
```

In `handler.ts` call it via SDK:
```typescript
import { LambdaClient, InvokeCommand } from '@aws-sdk/client-lambda';
const lambdaClient = tracer.captureAWSv3Client(new LambdaClient({}));
await lambdaClient.send(new InvokeCommand({
  FunctionName: process.env.ENRICHER_NAME,
  Payload: Buffer.from(JSON.stringify({ userId: event.userId })),
}));
```

Redeploy, invoke `/orders`, and the service map now shows a second Lambda node downstream of the first.

## Cleanup
```bash
cdk destroy
# Sampling rules are deleted with the stack. No orphan resources.
```

## Common Errors
- **Service map is empty after deploy** — `tracing: Tracing.ACTIVE` missing on the Lambda, OR `tracingEnabled` missing on API Gateway stage. Both are required.
- **`Missing AWS X-Ray Segment` errors in logs** — you called the tracer outside the Lambda handler (e.g., at module init). Only wrap code that runs inside `handler`.
- **Subsegment shows 0ms** — you forgot to `await` the async work before `sub.close()`.
- **Sampling rule ignored** — priority collision with the default rule (priority 10000). Use priority <10000 for custom rules.
- **Cross-service trace broken at `fetch`** — using Node 18+ native fetch without patching. Either import `https` or use `@aws-sdk/node-http-handler` which the X-Ray SDK patches automatically.
- **`TooManyRequests` on sampling** — `reservoirSize` too small under burst. Increase to 5-10 for busy paths.
