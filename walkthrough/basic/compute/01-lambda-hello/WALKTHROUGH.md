# Walkthrough — 01 Lambda Hello

## About this service
**AWS Lambda** runs code without servers. You upload a function; AWS runs it on demand triggered by events (HTTP, S3 upload, cron, queue message). You pay per invocation + per millisecond of execution × memory. Up to 15-min runtime, 10GB memory, 10GB ephemeral disk.

**Why it matters:** Lambda is the default for glue code, event handlers, low-traffic APIs, data transformations. No OS to patch, no capacity to plan. Scales from 0 to 10,000+ concurrent with zero ops.

**When to use Lambda:** event-driven, short-lived (<15min), bursty workloads, APIs with <1000 sustained RPS, ETL jobs.
**When NOT to use Lambda:** long-running processes (use Fargate/EC2), sustained high-RPS APIs where cost exceeds containers, GPU workloads, apps that need local state across requests, things needing >10GB memory.

## Estimated cost
- **First 1M invocations/mo: free** (always free, not just first year)
- **Beyond: $0.20 per 1M requests** + $0.0000166667 per GB-second
- **Free tier: 400,000 GB-seconds/mo** — this is generous
- Example: 1M invocations × 200ms × 128MB = ~25k GB-sec = free
- Total for this lesson: **$0.00** (well within free tier)

---

## Step 1: Project setup
> **Why:** `NodejsFunction` bundles TypeScript with esbuild automatically — no manual zip. This is the modern way.

```bash
mkdir lambda-hello && cd lambda-hello
cdk init app --language=typescript
npm install aws-cdk-lib constructs @types/aws-lambda
```

Create `lambda/handler.ts`:
```typescript
export const handler = async (event: any) => {
  console.log('event:', JSON.stringify(event));
  const greeting = process.env.GREETING ?? 'hello';
  return { statusCode: 200, body: `${greeting} world` };
};
```

## Step 2: Stack
> **Why:** 128MB is the cheapest but sometimes slowest option. `logRetention: 7` is critical — Lambda logs default to forever, and CloudWatch Logs costs add up fast.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import * as path from 'path';

export class LambdaHelloStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new NodejsFunction(this, 'Hello', {
      entry: path.join(__dirname, '../lambda/handler.ts'),
      handler: 'handler',
      runtime: lambda.Runtime.NODEJS_20_X,
      memorySize: 128,
      timeout: cdk.Duration.seconds(5),
      environment: { GREETING: 'hola' },
      logRetention: 7,
    });
  }
}
```

## Step 3: Invoke
> **Why:** `aws lambda invoke` directly tests your function without setting up triggers. Fastest feedback loop for dev.

```bash
FN=$(aws lambda list-functions --query "Functions[?contains(FunctionName, 'Hello')].FunctionName" --output text)
aws lambda invoke --function-name $FN --payload '{}' --cli-binary-format raw-in-base64-out out.json
cat out.json
# {"statusCode":200,"body":"hola world"}
```

## Step 4: Cold start observation
> **Why:** Cold start is the #1 Lambda gotcha. Understanding when they happen — and roughly how long they take — informs design decisions (should you use provisioned concurrency? Warm-up pings? Different runtime?).

Run 12 rapid invocations. CloudWatch Logs:
```
REPORT Duration: 12.34 ms   Billed Duration: 13 ms   Memory Size: 128 MB   Max Memory Used: 60 MB   Init Duration: 180.12 ms
```

`Init Duration` only appears on cold starts. ~1 cold + 11 warm out of 12 is typical.

## Step 5: Memory tuning
> **Why:** Lambda CPU scales with memory. Counterintuitive: more memory can cost LESS because the function finishes faster. Always profile.

CPU-bound handler:
```typescript
export const handler = async () => {
  const start = Date.now();
  let x = 0;
  for (let i = 0; i < 10_000_000; i++) x += Math.sqrt(i);
  return { ms: Date.now() - start };
};
```

Deploy at 128, 512, 1024, 2048 MB. Typical:
| Memory | Duration | Billed cost |
|--------|----------|-------------|
| 128    | 4800ms   | baseline    |
| 512    | 1200ms   | cheaper!    |
| 1024   | 600ms    | cheapest    |
| 2048   | 600ms    | CPU-capped  |

## Step 6: Timeout
> **Why:** Seeing a timeout in practice + how it appears in logs is faster than reading docs. Lambda kills your function mid-execution; there's no graceful shutdown.

Set `timeout: cdk.Duration.seconds(3)`, make handler sleep 5s → logs show `Task timed out after 3.00 seconds`.

## Step 7: Python variant
> **Why:** Language choice affects cold start. Python ~100ms, Node ~150ms, Java ~1-2s. Match runtime to performance requirements.

```typescript
import * as lp from 'aws-cdk-lib/aws-lambda-python-alpha';

new lp.PythonFunction(this, 'PyHello', {
  entry: 'lambda-py',
  runtime: lambda.Runtime.PYTHON_3_12,
});
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **"Unable to import module 'index'"** — handler string wrong (`index.handler` vs `handler.handler`).
- **esbuild not found** — `npm install esbuild --save-dev`.
