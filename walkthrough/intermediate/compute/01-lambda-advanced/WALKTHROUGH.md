# Walkthrough — 01 Lambda Advanced

## About this service
**AWS Lambda** in production involves more than a handler. Shared code goes into **layers**; pre-warmed capacity uses **provisioned concurrency**; private network access uses **VPC mode**; Java cold starts are fixed with **SnapStart**; observability comes from **Powertools**; async side-effects use **Destinations**; simple HTTPS entry uses **Function URLs**. Mastering these is the difference between a toy and a production Lambda.

**Why it matters:** Almost every real Lambda hits at least one of these features. Getting them wrong causes 5-second cold starts, $300/mo idle provisioned capacity, or VPC outages when ENIs exhaust.

**When to use:** layers (2+ functions share code/deps), provisioned concurrency (latency-sensitive user-facing APIs), VPC (need RDS/ElastiCache), SnapStart (JVM functions), Powertools (any production Lambda), Destinations (async pipelines), Function URL (simple webhooks, no API GW features needed).
**When NOT to use:** layers for one-off code; provisioned concurrency for low-traffic (just eat the cold start); VPC unless you truly need it (adds latency + ENI limits); Function URL when you need auth/throttling/WAF (use API GW).

## Estimated cost
- **Lambda requests: $0.20 per 1M** + $0.0000166667 per GB-second
- **Provisioned concurrency: $0.0000041667 per GB-second allocated** (charged even idle) → 5 × 512MB = ~$15/month
- **SnapStart: $0.0000015 per GB cached** + $0.0001397 per GB restore
- **X-Ray: $5.00 per 1M traces recorded**
- **CloudWatch Logs: $0.50/GB ingested**, $0.03/GB stored
- Total for this lesson: **~$18/month** (provisioned concurrency is the big line item; destroy it when done)

---

## Step 1: Project setup
> **Why:** One stack with multiple advanced patterns so you can compare. `aws-lambda-powertools` brings structured logging, metrics, and tracing without writing boilerplate.

```bash
mkdir lambda-advanced && cd lambda-advanced
cdk init app --language=typescript
npm install aws-cdk-lib constructs @types/aws-lambda
npm install --save @aws-lambda-powertools/logger @aws-lambda-powertools/tracer @aws-lambda-powertools/metrics axios
```

## Step 2: Build a Lambda layer
> **Why:** Layers strip shared dependencies from each function's package. A layer with `axios` shared by 10 functions saves ~400KB each and makes deploys faster. The layer counts against the 250MB unzipped limit of the function though — don't dump everything in.

```typescript
// lib/layer-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as path from 'path';

export class LayerStack extends cdk.Stack {
  public readonly sharedLayer: lambda.LayerVersion;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.sharedLayer = new lambda.LayerVersion(this, 'SharedLayer', {
      code: lambda.Code.fromAsset(path.join(__dirname, '../layer'), {
        bundling: {
          image: lambda.Runtime.NODEJS_20_X.bundlingImage,
          command: [
            'bash', '-c',
            'mkdir -p /asset-output/nodejs && cp -r . /asset-output/nodejs/ && cd /asset-output/nodejs && npm install --production',
          ],
        },
      }),
      compatibleRuntimes: [lambda.Runtime.NODEJS_20_X],
      description: 'axios + shared utils',
    });
  }
}
```

Create `layer/package.json`:
```json
{ "name": "shared", "version": "1.0.0", "dependencies": { "axios": "^1.6.0" } }
```

Create `layer/utils.js`:
```javascript
exports.hash = (s) => require('crypto').createHash('sha256').update(s).digest('hex');
```

## Step 3: Functions that consume the layer
> **Why:** Two functions share one layer. Notice neither has axios in its own deps — the layer brings it. `logRetention: 7` keeps log bills low.

```typescript
// lib/functions-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import * as path from 'path';

interface Props extends cdk.StackProps { sharedLayer: lambda.LayerVersion; }

export class FunctionsStack extends cdk.Stack {
  public readonly apiFn: NodejsFunction;

  constructor(scope: Construct, id: string, props: Props) {
    super(scope, id, props);

    const common = {
      runtime: lambda.Runtime.NODEJS_20_X,
      layers: [props.sharedLayer],
      memorySize: 512,
      timeout: cdk.Duration.seconds(10),
      logRetention: 7,
      tracing: lambda.Tracing.ACTIVE,
      bundling: { externalModules: ['axios', '/opt/nodejs/utils'] },
    };

    this.apiFn = new NodejsFunction(this, 'ApiFn', {
      ...common,
      entry: path.join(__dirname, '../src/api.ts'),
      environment: { POWERTOOLS_SERVICE_NAME: 'api', POWERTOOLS_METRICS_NAMESPACE: 'LambdaAdv' },
    });

    new NodejsFunction(this, 'WorkerFn', {
      ...common,
      entry: path.join(__dirname, '../src/worker.ts'),
    });
  }
}
```

Create `src/api.ts`:
```typescript
import { Logger } from '@aws-lambda-powertools/logger';
import { Tracer } from '@aws-lambda-powertools/tracer';
import { Metrics, MetricUnit } from '@aws-lambda-powertools/metrics';
import axios from 'axios';

const logger = new Logger();
const tracer = new Tracer();
const metrics = new Metrics();

export const handler = async (event: any) => {
  logger.info('request received', { event });
  metrics.addMetric('Invocation', MetricUnit.Count, 1);
  const seg = tracer.getSegment()?.addNewSubsegment('external-call');
  const r = await axios.get('https://httpbin.org/uuid');
  seg?.close();
  metrics.publishStoredMetrics();
  return { statusCode: 200, body: JSON.stringify(r.data) };
};
```

## Step 4: Provisioned concurrency via alias
> **Why:** Provisioned concurrency only works on a **version/alias**, not `$LATEST`. You pay per allocated GB-second whether traffic hits or not. p99 drops from ~800ms (cold) to <50ms (warm) for Node.

```typescript
const version = this.apiFn.currentVersion;
const alias = new lambda.Alias(this, 'ApiAlias', {
  aliasName: 'live',
  version,
  provisionedConcurrentExecutions: 5,
});
```

Load test:
```bash
# hey is a simple Go load tester
hey -n 1000 -c 20 https://<function-url>/
# Summary:
#   Requests/sec: 180
#   Latencies p50 28ms p95 42ms p99 48ms
```

Without provisioned concurrency, p99 typically 400-900ms.

## Step 5: VPC Lambda
> **Why:** A Lambda in a VPC uses Hyperplane ENIs — shared across functions, no per-invoke ENI creation. First-ever deploy still takes ~90s for the ENI to attach; subsequent invokes are normal. Place in **private subnets** with a NAT for outbound internet.

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';

const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

new NodejsFunction(this, 'VpcFn', {
  entry: path.join(__dirname, '../src/vpc.ts'),
  runtime: lambda.Runtime.NODEJS_20_X,
  vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  allowPublicSubnet: false,
  memorySize: 512,
  timeout: cdk.Duration.seconds(10),
  logRetention: 7,
});
```

Expected cold start on first invoke: ~1s. Subsequent: ~200ms.

## Step 6: Async destinations
> **Why:** For async invokes (S3, SNS, EventBridge → Lambda), **Destinations** route success/failure to another service without custom code. Cleaner than DLQs because they pass the original event + response.

```typescript
import * as destinations from 'aws-cdk-lib/aws-lambda-destinations';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sns from 'aws-cdk-lib/aws-sns';

const successQ = new sqs.Queue(this, 'SuccessQ');
const failureTopic = new sns.Topic(this, 'FailureTopic');

new NodejsFunction(this, 'AsyncFn', {
  entry: path.join(__dirname, '../src/async.ts'),
  runtime: lambda.Runtime.NODEJS_20_X,
  onSuccess: new destinations.SqsDestination(successQ),
  onFailure: new destinations.SnsDestination(failureTopic),
  retryAttempts: 2,
});
```

## Step 7: Function URL
> **Why:** For webhooks or simple HTTPS endpoints, skip API Gateway. Function URL is free, has built-in TLS, and CORS. No rate limiting, no auth beyond IAM-SigV4 or none.

```typescript
const fnUrl = this.apiFn.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.NONE,
  cors: { allowedOrigins: ['*'], allowedMethods: [lambda.HttpMethod.ALL] },
});
new cdk.CfnOutput(this, 'FnUrl', { value: fnUrl.url });
```

```bash
curl https://abc123xyz.lambda-url.us-east-1.on.aws/
# {"uuid":"8a7b9c..."}
```

## Step 8: SnapStart (Java)
> **Why:** Java cold starts can hit 3-8s due to JVM init. SnapStart snapshots the post-init state after version publish, then restores in ~200ms. Free to enable; you pay per restore.

```typescript
import { Runtime, Function as LambdaFn, Code, SnapStartConf } from 'aws-cdk-lib/aws-lambda';

new LambdaFn(this, 'JavaFn', {
  runtime: Runtime.JAVA_21,
  handler: 'com.example.App::handleRequest',
  code: Code.fromAsset('java-target/app.jar'),
  memorySize: 1024,
  snapStart: SnapStartConf.ON_PUBLISHED_VERSIONS,
});
```

Before SnapStart: `Init Duration: 4821.3 ms`. After: `Restore Duration: 210.8 ms`.

## Cleanup
```bash
# Delete provisioned concurrency FIRST or bill keeps running
aws lambda delete-alias --function-name <fn> --name live
cdk destroy
```

## Common Errors
- **`Cannot find module 'axios'`** → layer not attached, or bundling didn't mark it `external`. Add it to `bundling.externalModules`.
- **VPC Lambda times out calling internet** → missing NAT gateway or wrong subnet type (public has no outbound via ENI).
- **Provisioned concurrency stuck `IN_PROGRESS`** → version hash changed on redeploy; you're allocating a new version. Use `currentVersion` with stable hash.
- **SnapStart CRAC hooks needed** — state tied to init (DB connections, random seeds) breaks after restore. Use `Core.getGlobalContext().register(...)` hooks.
- **Function URL 403** → `authType: NONE` missing, or CORS preflight blocked.
