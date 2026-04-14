# Walkthrough — 02 Lambda@Edge vs CloudFront Functions

## About this service
CloudFront runs two different edge-compute runtimes. **CloudFront Functions** are JS-only, sub-millisecond, ~2 MB memory, fire on **viewer-request/viewer-response** only, cannot make network calls or access the request body, and cost cents per million. **Lambda@Edge** is full Node.js/Python, fires on all four events (**viewer-request, origin-request, origin-response, viewer-response**), can call AWS APIs / HTTP, reads/mutates body, up to 10 GB memory at origin events — but costs ~10x more and takes ~5 min to propagate globally.

**Why it matters:** Anything you can push to the edge runs ~100x closer to users and never hits origin. Header injection, A/B cookies, token validation, URL normalization, on-the-fly image resizing — all standard edge workloads. Picking the wrong runtime means either paying 10x too much or writing code you can't ship (CF Function can't call STS; Lambda@Edge can't respond in 1 ms).
**When to use CF Functions:** header manipulation, simple rewrites, cookie math, JWT-less auth checks — anything pure-JS at viewer events.
**When to use Lambda@Edge:** URL rewrites to S3, image transforms, SSR hydration, AWS SDK calls, request-body mutation.
**When NOT to use either:** full application logic (use API Gateway + Lambda); long responses (>40s origin-response limit); big binary processing (send to S3/Lambda async).

## Estimated cost
- **CloudFront Functions: $0.10 per 1M invocations** (no compute charge)
- **Lambda@Edge: $0.60 per 1M requests + $0.00005001 per GB-second**
- Worked example — 1M requests/day = 30M/month:
  - CF Function: 30 x $0.10 = **$3/month**
  - Lambda@Edge (128 MB, 50ms avg): 30 x $0.60 + 30M x 0.128 x 0.050 x $0.00005001 = $18 + $0.96 = **~$19/month**
- **CloudFront data transfer out**: $0.085/GB first 10 TB (unchanged either way)
- Total for this lesson: **~$5/month** during exercises (low traffic; CF distribution itself is free to have)

---

## Step 1: Baseline CloudFront distribution in CDK
> **Why:** You need a distribution with both a CF Function and a Lambda@Edge association to compare behaviors. Origin is a simple S3 bucket serving JSON.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cf from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Construct } from 'constructs';

export class EdgeStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    // Lambda@Edge MUST be in us-east-1
    super(scope, id, { ...props, env: { region: 'us-east-1' } });

    const bucket = new s3.Bucket(this, 'Origin', {
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    // --- CloudFront Function (viewer-response) ---
    const secHeaders = new cf.Function(this, 'SecHeadersFn', {
      code: cf.FunctionCode.fromInline(`
function handler(event) {
  var r = event.response;
  r.headers['strict-transport-security'] = { value: 'max-age=63072000; includeSubDomains; preload' };
  r.headers['x-frame-options'] = { value: 'DENY' };
  r.headers['x-content-type-options'] = { value: 'nosniff' };
  r.headers['content-security-policy'] = { value: "default-src 'self'" };
  r.headers['referrer-policy'] = { value: 'strict-origin-when-cross-origin' };
  return r;
}`),
    });

    // --- CloudFront Function (viewer-request, A/B cookie) ---
    const abFn = new cf.Function(this, 'ABFn', {
      code: cf.FunctionCode.fromInline(`
function handler(event) {
  var req = event.request;
  var cookies = req.cookies || {};
  if (!cookies.variant) {
    var v = (Math.random() < 0.5) ? 'A' : 'B';
    req.headers['x-variant'] = { value: v };
    req.headers['x-set-cookie'] = { value: 'variant=' + v + '; Path=/; Max-Age=86400' };
  } else {
    req.headers['x-variant'] = { value: cookies.variant.value };
  }
  return req;
}`),
    });

    // --- Lambda@Edge (origin-request, URL rewrite) ---
    const rewriteFn = new cf.experimental.EdgeFunction(this, 'RewriteFn', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`
exports.handler = async (event) => {
  const req = event.Records[0].cf.request;
  // /users/123 -> /users/123.json
  const m = req.uri.match(/^\\/users\\/([a-z0-9-]+)$/i);
  if (m) req.uri = '/users/' + m[1] + '.json';
  return req;
};`),
    });

    new cf.Distribution(this, 'Dist', {
      defaultBehavior: {
        origin: new origins.S3BucketOrigin(bucket),
        viewerProtocolPolicy: cf.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        functionAssociations: [
          { function: abFn, eventType: cf.FunctionEventType.VIEWER_REQUEST },
          { function: secHeaders, eventType: cf.FunctionEventType.VIEWER_RESPONSE },
        ],
        edgeLambdas: [
          { functionVersion: rewriteFn.currentVersion, eventType: cf.LambdaEdgeEventType.ORIGIN_REQUEST },
        ],
      },
    });
  }
}
```

```bash
cdk deploy
# Wait ~5 min after deploy for edge propagation
```

## Step 2: Verify security headers
> **Why:** Security headers belong at viewer-response so they apply to cached AND uncached responses uniformly. `curl -I` confirms the CF Function fired.

```bash
curl -sI https://d123.cloudfront.net/ | grep -iE 'strict-transport|x-frame|csp|content-security'
# strict-transport-security: max-age=63072000; includeSubDomains; preload
# x-frame-options: DENY
# content-security-policy: default-src 'self'
```

## Step 3: Verify A/B cookie assignment
> **Why:** Cookie-based bucketing at the edge is free and cacheable when you shard the cache key by the cookie.

```bash
curl -sI https://d123.cloudfront.net/ | grep -i x-variant
# x-variant: A   (or B, 50/50)
```

To make this cacheable, add a cache policy keyed on `variant` cookie:

```typescript
cachePolicy: new cf.CachePolicy(this, 'ABCache', {
  cookieBehavior: cf.CacheCookieBehavior.allowList('variant'),
}),
```

## Step 4: Verify URL rewrite via Lambda@Edge
> **Why:** The rewrite runs at **origin-request**, which means one cached object per rewritten URL (good: cache key is rewritten URI). Putting this at viewer-request would blow the cache wide open.

```bash
aws s3 cp - s3://<origin-bucket>/users/123.json <<< '{"id":123,"name":"Alice"}'

curl -s https://d123.cloudfront.net/users/123
# {"id":123,"name":"Alice"}
```

## Step 5: Lambda@Edge origin-response — image resize proxy
> **Why:** Origin-response mutations let you transform once per origin hit and cache the result. Classic pattern: `?w=200` querystring → resize on the fly with Sharp → return smaller image.

Because `sharp` is native and L@E has no layers, package it with an `x64-linux` build:

```bash
mkdir resize && cd resize
npm init -y
npm install --arch=x64 --platform=linux --include=optional sharp
```

`index.js`:
```javascript
const sharp = require('sharp');
const https = require('https');

exports.handler = async (event) => {
  const resp = event.Records[0].cf.response;
  const req = event.Records[0].cf.request;
  const qs = new URLSearchParams(req.querystring);
  const w = parseInt(qs.get('w') || '0', 10);
  if (!w || resp.status !== '200') return resp;

  // Fetch original from origin (response body isn't passed through by default for binaries)
  const originUrl = `https://${req.origin.s3.domainName}${req.uri}`;
  const buf = await new Promise((res, rej) => {
    https.get(originUrl, r => {
      const chunks = [];
      r.on('data', c => chunks.push(c));
      r.on('end', () => res(Buffer.concat(chunks)));
      r.on('error', rej);
    });
  });

  const resized = await sharp(buf).resize({ width: w }).toBuffer();
  resp.body = resized.toString('base64');
  resp.bodyEncoding = 'base64';
  resp.headers['content-type'] = [{ key: 'Content-Type', value: 'image/jpeg' }];
  resp.headers['cache-control'] = [{ key: 'Cache-Control', value: 'public, max-age=31536000' }];
  return resp;
};
```

Associate with origin-response + add querystring to the cache key (`?w=200` must be in the key or every user collides on one cached variant).

## Step 6: Cost comparison — 1M req/day
> **Why:** Doing the math forces you to pick the right tool. Header injection for 30M req/mo at $19 (Edge) vs $3 (CF Fn) is a 6x waste.

| Workload (30M/month) | CF Function | Lambda@Edge (128 MB, 50ms) |
|---|---|---|
| Add 5 security headers | **$3** | $19 |
| A/B cookie assignment | **$3** | $19 |
| `/users/:id` → S3 key rewrite | not possible (origin event) | **$19** |
| Image resize (Sharp, 256 MB, 200ms) | not possible | **$99** (+CF egress) |

Rule of thumb: if you **can** do it as a CF Function, do it as a CF Function.

## Step 7: Decision doc — `cf-func-vs-edge.md`
> **Why:** Future you (and teammates) will reopen this question every quarter. Codify the decision tree once.

```markdown
# CF Functions vs Lambda@Edge

## Pick CloudFront Functions when ALL of:
- Event is viewer-request or viewer-response
- No network calls / AWS SDK needed
- No request/response body access needed
- Logic fits in < 10 KB compiled JS and < 1ms runtime

## Pick Lambda@Edge when ANY of:
- Event is origin-request or origin-response
- Need AWS SDK (Secrets Manager for JWT key, DDB lookup, etc.)
- Need to read/mutate body
- Need Python / larger memory

## Neither, go to origin when:
- Logic > 50 ms compute
- Needs persistent state (use DDB behind API GW)
- Needs WebSocket / long-poll
```

## Cleanup
```bash
cdk destroy
# NOTE: Lambda@Edge replicas delete asynchronously (~1 hr). If destroy fails
# with "replicated function", wait and retry.
```

## Common Errors
- **CDK deploys Lambda@Edge in wrong region** → `EdgeFunction` must be created in a stack pinned to `us-east-1` (set `env.region`). CDK errors out otherwise.
- **`The function cannot have environment variables`** → Lambda@Edge forbids env vars. Use SSM Parameter Store or bake config into code.
- **`The function execution role must be assumable by edgelambda.amazonaws.com`** → CDK's `EdgeFunction` sets this automatically; manual roles must add both `lambda.amazonaws.com` and `edgelambda.amazonaws.com`.
- **CF Function `Function size exceeds maximum`** → 10 KB limit. No bundling, no dependencies. Keep it hand-written JS.
- **CF Function `Cannot access request.body`** → bodies aren't available in CF Functions ever. Move to Lambda@Edge viewer-request.
- **`cdk destroy` hangs on replicated function** → wait up to 1 hr for global replicas to drain, then retry.
- **Image resize returns 502** → response body > 1 MB limit for viewer events; use origin-response (allowed up to 1 MB generated, larger via streaming).
