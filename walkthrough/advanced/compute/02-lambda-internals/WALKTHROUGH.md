# Walkthrough — 02 Lambda Internals

## About this service
**Lambda's runtime** is a Firecracker microVM that cycles through three phases: **Init** (download code, start runtime, run module-level code), **Invoke** (your handler runs, potentially many times on the same microVM), and **Shutdown** (best-effort, <500ms). **Extensions** are co-processes that share the microVM lifecycle — internal extensions run inside the runtime process, external extensions run as separate processes with their own binary. **Response streaming** lets you `flush()` bytes to the client before the handler returns, cutting TTFB for LLM-style workloads.

**Why it matters:** every performance and cost decision at scale comes down to these phases. Init is billed at full CPU regardless of your memory setting ("init burst"). Invoke is billed at your configured memory. Knowing the boundary lets you move expensive work (SDK clients, DB connections, model loads) into init and get it effectively free.
**When to use these techniques:** cold-start-sensitive APIs, streaming LLM responses, out-of-band log shipping, per-function observability without code changes.
**When NOT to use:** if P99 latency doesn't matter (batch jobs), if provisioned concurrency already solves your cold start, or if you can't justify the complexity of a custom extension.

## Estimated cost
- **Lambda compute: $0.0000166667/GB-second + $0.20 per 1M requests** (us-east-1, x86)
- **Arm64 (Graviton)**: 20% cheaper — $0.0000133334/GB-s
- **Response streaming**: same compute price, but streaming adds **$0.008/GB** for bytes streamed beyond the first 6 MB
- **Provisioned concurrency**: $0.0000041667/GB-s while provisioned + $0.0000097222/GB-s per invocation
- **CloudWatch Logs ingestion**: $0.50/GB (often bigger than the compute bill for chatty functions)
- Lesson example (10k invokes × 512 MB × 200ms): compute ~$0.002, requests ~$0.002, logs ~$0.05
- Total for this lesson: **~$1/month** if you don't leave a benchmark loop running. Watch the logs bill.

---

## Step 1: Provision a Lambda with Powertools tracing
> **Why:** To measure Init vs Invoke phases you need structured logs that mark phase boundaries. AWS Lambda Powertools writes a REPORT-aligned log with `initDuration` and `duration` fields already parsed. Without this you're squinting at REPORT lines.

`lib/lambda-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as nodejs from 'aws-cdk-lib/aws-lambda-nodejs';
import * as logs from 'aws-cdk-lib/aws-logs';

export class LambdaStack extends cdk.Stack {
  constructor(scope: any, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const fn = new nodejs.NodejsFunction(this, 'Fn', {
      entry: 'src/handler.ts',
      runtime: lambda.Runtime.NODEJS_20_X,
      architecture: lambda.Architecture.ARM_64,
      memorySize: 512,
      timeout: cdk.Duration.seconds(30),
      logRetention: logs.RetentionDays.ONE_WEEK,
      environment: {
        POWERTOOLS_SERVICE_NAME: 'cold-start-demo',
        POWERTOOLS_LOG_LEVEL: 'INFO',
      },
      bundling: {
        minify: true,
        sourceMap: true,
        externalModules: ['@aws-sdk/*'], // use runtime-provided SDK
      },
    });

    new cdk.CfnOutput(this, 'FnName', { value: fn.functionName });
  }
}
```

## Step 2: Measure init phase cost — outer vs inner scope
> **Why:** Module-top-level code runs once per microVM, then every warm invoke skips it. Code inside the handler runs every invoke. Moving a DynamoDB client from inner to outer scope is ~40ms of savings on **every** warm invoke for the life of that microVM (minutes to hours).

`src/handler.ts`:
```typescript
import { Logger } from '@aws-lambda-powertools/logger';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';

const logger = new Logger();
const initStart = process.hrtime.bigint();

// OUTER SCOPE: pays cost once per cold start
const ddb = new DynamoDBClient({});
logger.info('outer init done', {
  ms: Number(process.hrtime.bigint() - initStart) / 1e6,
});

export const handler = async (event: any) => {
  const invokeStart = process.hrtime.bigint();
  // INNER SCOPE (BAD): would recreate every invoke
  // const ddb = new DynamoDBClient({});
  await ddb.config.credentials();
  logger.info('invoke done', {
    ms: Number(process.hrtime.bigint() - invokeStart) / 1e6,
    coldStart: !(global as any).warm,
  });
  (global as any).warm = true;
  return { statusCode: 200 };
};
```

Fire 10 cold starts by forcing new versions:
```bash
for i in {1..10}; do
  aws lambda update-function-configuration \
    --function-name <fn> --environment "Variables={BUMP=$i}" >/dev/null
  aws lambda wait function-updated --function-name <fn>
  aws lambda invoke --function-name <fn> --payload '{}' /tmp/out >/dev/null
done

aws logs filter-log-events --log-group-name /aws/lambda/<fn> \
  --filter-pattern '{ $.message = "outer init done" }' \
  --query 'events[].message'
```

Typical: **Init 180-350ms**, **warm invoke <5ms**. Moving the client inside the handler adds ~40ms to every invoke.

## Step 3: Internal extension (Node) for lifecycle logging
> **Why:** Internal extensions share the runtime process — ideal for zero-overhead instrumentation that needs the same env as your handler (e.g. capturing unhandled rejections, wrapping SDK clients). External extensions are heavier but isolate failure. Start with internal unless you need a different language or crash isolation.

`src/internal-extension.mjs` (loaded via `NODE_OPTIONS=--require`):
```javascript
import { request } from 'http';
const base = `http://${process.env.AWS_LAMBDA_RUNTIME_API}/2020-01-01/extension`;

const register = () => new Promise((resolve) => {
  const req = request(`${base}/register`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Lambda-Extension-Name': 'internal-lifecycle',
    },
  }, (res) => resolve(res.headers['lambda-extension-identifier']));
  req.write(JSON.stringify({ events: ['INVOKE', 'SHUTDOWN'] }));
  req.end();
});

const nextEvent = (id) => new Promise((resolve) => {
  request(`${base}/event/next`, {
    headers: { 'Lambda-Extension-Identifier': id },
  }, (res) => {
    let body = '';
    res.on('data', (c) => body += c);
    res.on('end', () => resolve(JSON.parse(body)));
  }).end();
});

(async () => {
  const id = await register();
  while (true) {
    const evt = await nextEvent(id);
    console.log(JSON.stringify({ ext: 'lifecycle', type: evt.eventType }));
    if (evt.eventType === 'SHUTDOWN') process.exit(0);
  }
})();
```

Set `NODE_OPTIONS=--import ./internal-extension.mjs` on the function.

## Step 4: External extension (Go) for out-of-band log shipping
> **Why:** If your handler writes 10 MB of logs, CloudWatch charges $0.005 per invoke just for ingestion. An external extension running `tail -f` on `/var/log/lambda` can ship to S3/Kinesis/OpenSearch directly and skip CloudWatch entirely. It runs in a separate process so a panic won't take down the handler.

`extensions/logshipper/main.go`:
```go
package main

import (
  "bytes"
  "encoding/json"
  "net/http"
  "os"
)

var runtimeAPI = os.Getenv("AWS_LAMBDA_RUNTIME_API")

func register() string {
  body, _ := json.Marshal(map[string][]string{"events": {"INVOKE", "SHUTDOWN"}})
  req, _ := http.NewRequest("POST",
    "http://"+runtimeAPI+"/2020-01-01/extension/register",
    bytes.NewReader(body))
  req.Header.Set("Lambda-Extension-Name", "logshipper")
  resp, _ := http.DefaultClient.Do(req)
  return resp.Header.Get("Lambda-Extension-Identifier")
}

func next(id string) map[string]interface{} {
  req, _ := http.NewRequest("GET",
    "http://"+runtimeAPI+"/2020-01-01/extension/event/next", nil)
  req.Header.Set("Lambda-Extension-Identifier", id)
  resp, _ := http.DefaultClient.Do(req)
  var v map[string]interface{}
  json.NewDecoder(resp.Body).Decode(&v)
  return v
}

func main() {
  id := register()
  for {
    evt := next(id)
    // shipLogsToS3() ...
    if evt["eventType"] == "SHUTDOWN" {
      return
    }
  }
}
```

Build + package:
```bash
GOOS=linux GOARCH=arm64 go build -o extensions/logshipper main.go
chmod +x extensions/logshipper
zip -r extension.zip extensions/
aws lambda publish-layer-version --layer-name logshipper \
  --zip-file fileb://extension.zip \
  --compatible-architectures arm64
```

Attach the layer to your function.

## Step 5: Response streaming
> **Why:** A normal Lambda response buffers until the handler returns — bad for LLM tokens or large reports. `awslambda.streamifyResponse` lets you flush bytes incrementally. First byte can land in hundreds of ms instead of tens of seconds. Must be invoked via Function URL with `InvokeMode=RESPONSE_STREAM`.

`src/stream.ts`:
```typescript
import type { Handler } from 'aws-lambda';

declare const awslambda: any;

export const handler: Handler = awslambda.streamifyResponse(
  async (_event: any, responseStream: NodeJS.WritableStream) => {
    responseStream = awslambda.HttpResponseStream.from(responseStream, {
      statusCode: 200,
      headers: { 'Content-Type': 'text/event-stream' },
    });
    for (let i = 0; i < 20; i++) {
      responseStream.write(`data: chunk ${i}\n\n`);
      await new Promise((r) => setTimeout(r, 200));
    }
    responseStream.end();
  }
);
```

Expose via Function URL:
```typescript
fn.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.NONE,
  invokeMode: lambda.InvokeMode.RESPONSE_STREAM,
});
```

Measure TTFB:
```bash
curl -N -w '\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n' \
  https://<url>.lambda-url.us-east-1.on.aws/
# TTFB ~0.3s, Total ~4s  (buffered would show TTFB == Total)
```

## Step 6: Observe the shutdown phase
> **Why:** Shutdown is your only hook to flush buffers, close DB connections cleanly, or emit final metrics. It is best-effort — you get <500ms and it only fires when Lambda scales the microVM down (idle timeout is usually 5-15 min but undocumented). Don't rely on it for correctness; use it for hygiene.

The extension from Step 3 already logs SHUTDOWN events. Idle for 15+ minutes then inspect:
```bash
aws logs filter-log-events --log-group-name /aws/lambda/<fn> \
  --filter-pattern '"SHUTDOWN"' --query 'events[].message'
```

## Step 7: Memory vs CPU sweet-spot benchmark
> **Why:** CPU scales linearly with memory up to 1769 MB (= 1 vCPU) and continues adding vCPUs above that. A CPU-bound task at 512 MB may cost the same at 1792 MB because duration drops proportionally — but past that, you pay for unused capacity. Plot the curve for **your** workload; never default to 128 MB and never default to 10 GB.

`src/bench.ts`:
```typescript
export const handler = async () => {
  const start = Date.now();
  // CPU-bound: sha256 a 4 MB buffer 200 times
  const crypto = await import('crypto');
  const buf = Buffer.alloc(4 * 1024 * 1024, 'x');
  for (let i = 0; i < 200; i++) crypto.createHash('sha256').update(buf).digest();
  return { ms: Date.now() - start };
};
```

```bash
for MEM in 128 256 512 1024 1792 3008 5120 10240; do
  aws lambda update-function-configuration --function-name <fn> --memory-size $MEM >/dev/null
  aws lambda wait function-updated --function-name <fn>
  DUR=$(aws lambda invoke --function-name <fn> /tmp/o --query 'ExecutedVersion' --output text >/dev/null && \
        aws lambda invoke --function-name <fn> /tmp/o >/dev/null && cat /tmp/o | jq -r '.ms')
  COST=$(python -c "print($MEM/1024 * $DUR/1000 * 0.0000133334)")
  echo "$MEM MB  ${DUR}ms  \$$COST"
done
```

Typical curve (CPU-bound): duration halves from 128→1024 (cost flat), keeps falling to 1792 (cost flat), then flattens while cost rises. **Sweet spot: ~1792 MB** for CPU work.

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **Init duration >10s → INIT_FAILED** → top-level `await` hit a network call with no timeout. Wrap in `Promise.race` with a timeout.
- **Extension not registering: `403 Forbidden`** → extension binary not at `/opt/extensions/<name>` with execute permissions. Layer must have the exact path `extensions/`.
- **Response streaming returns buffered** → function URL `InvokeMode` still `BUFFERED`. Recreate the URL; can't change in place.
- **`aws lambda invoke` returns `202` immediately** → invocation type is `Event` (async) instead of `RequestResponse`. Omit `--invocation-type`.
- **Logs show "Task timed out after 3.00 seconds" on first call only** → init+invoke billed against the 3s timeout. Raise timeout or move work to provisioned concurrency.
- **Extension eats into 10 GB memory budget** → if your handler needs 9 GB and the extension needs 1 GB, function OOMs. Extensions share the function memory limit.
- **SHUTDOWN never fires during testing** → idle timer is 5-15 min and undocumented. Don't depend on shutdown for correctness — use try/finally in the handler.
