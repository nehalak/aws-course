# Walkthrough — 04 CloudFront + WAF

## About this service
**CloudFront** is AWS's CDN — 600+ edge Points of Presence that cache your content close to users. Origins can be S3, ALB, API Gateway, MediaStore, or any HTTPS endpoint. **WAF (Web Application Firewall)** sits in front of CloudFront/ALB/API GW and inspects L7 traffic — it blocks SQL injection, bad bots, rate-limit abusers, and unwanted countries. **Lambda@Edge** and **CloudFront Functions** let you mutate requests/responses at the edge (auth, redirects, header shaping).

**Why it matters:** these are the two services that turn a website into something that performs globally and survives attacks. Misconfigure either and you either (a) cache private data to strangers or (b) block legitimate users.

**When to use CloudFront:** static sites, SPAs, image-heavy apps, API acceleration, DDoS absorption (Shield Standard is free and automatic in front of CF).
**When to use WAF:** anything public-facing. Core Rule Set is cheap insurance.
**When NOT to use CloudFront:** strictly internal apps (use ALB directly), or workloads where sub-10ms latency beats caching (rare for web).

## Estimated cost
- **CloudFront data out: $0.085/GB** (first 1TB), $0.01/10k HTTPS requests
- **CloudFront distribution: free** to exist; pay per use
- **WAF Web ACL: $5/month** + $1/rule/month + $0.60 per 1M requests
- **AWS Managed Rules (Core Rule Set): ~$1/rule group/month**
- **Lambda@Edge: $0.60/1M requests** + compute time, replicated globally
- Total for this lesson if idle: **~$10/month** (mostly WAF ACL + managed rules). Destroy after.

---

## Step 1: Scaffold + S3 origin with OAC
> **Why:** Origin Access Control (OAC) is the current pattern — deprecated OAI used static SigV2. OAC uses SigV4 and works with S3 SSE-KMS. Keep the bucket private; only CloudFront can read.

```bash
mkdir cf-waf && cd cf-waf
cdk init app --language=typescript
npm install aws-cdk-lib
```

`lib/cf-waf-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as s3deploy from 'aws-cdk-lib/aws-s3-deployment';
import * as cf from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';

export class CfWafStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, { ...props, env: { region: 'us-east-1' } });

    // --- S3 origin (private) ---
    const bucket = new s3.Bucket(this, 'Site', {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    // --- Step 2: custom cache policy ---
    const cachePolicy = new cf.CachePolicy(this, 'AuthCache', {
      defaultTtl: cdk.Duration.minutes(5),
      minTtl: cdk.Duration.seconds(0),
      maxTtl: cdk.Duration.hours(1),
      headerBehavior: cf.CacheHeaderBehavior.allowList('Authorization'),
      queryStringBehavior: cf.CacheQueryStringBehavior.all(),
      cookieBehavior: cf.CacheCookieBehavior.none(),
    });

    // --- Step 5: WAF ACL (must be us-east-1 for CloudFront) ---
    const acl = new wafv2.CfnWebACL(this, 'Acl', {
      scope: 'CLOUDFRONT',
      defaultAction: { allow: {} },
      visibilityConfig: { sampledRequestsEnabled: true, cloudWatchMetricsEnabled: true, metricName: 'acl' },
      rules: [
        {
          name: 'AWSCommonRules', priority: 0,
          overrideAction: { none: {} },
          statement: {
            managedRuleGroupStatement: {
              vendorName: 'AWS', name: 'AWSManagedRulesCommonRuleSet',
            },
          },
          visibilityConfig: { sampledRequestsEnabled: true, cloudWatchMetricsEnabled: true, metricName: 'common' },
        },
        {
          name: 'RateLimit', priority: 1,
          action: { block: {} },
          statement: {
            rateBasedStatement: { limit: 100, aggregateKeyType: 'IP' },
          },
          visibilityConfig: { sampledRequestsEnabled: true, cloudWatchMetricsEnabled: true, metricName: 'rate' },
        },
        {
          name: 'GeoBlock', priority: 2,
          action: { block: {} },
          statement: { geoMatchStatement: { countryCodes: ['KP'] } },
          visibilityConfig: { sampledRequestsEnabled: true, cloudWatchMetricsEnabled: true, metricName: 'geo' },
        },
      ],
    });

    // --- Step 7: CloudFront Function for security headers ---
    const secHeaders = new cf.Function(this, 'SecHeaders', {
      code: cf.FunctionCode.fromInline(`
function handler(event) {
  var res = event.response;
  res.headers['strict-transport-security'] = { value: 'max-age=31536000; includeSubDomains' };
  res.headers['x-content-type-options'] = { value: 'nosniff' };
  res.headers['x-frame-options'] = { value: 'DENY' };
  return res;
}`),
      eventType: cf.FunctionEventType.VIEWER_RESPONSE,
    });

    // --- Distribution ---
    const dist = new cf.Distribution(this, 'Dist', {
      defaultRootObject: 'index.html',
      defaultBehavior: {
        origin: origins.S3BucketOrigin.withOriginAccessControl(bucket),
        viewerProtocolPolicy: cf.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cachePolicy: cachePolicy,
        functionAssociations: [{ function: secHeaders, eventType: cf.FunctionEventType.VIEWER_RESPONSE }],
      },
      webAclId: acl.attrArn,
      priceClass: cf.PriceClass.PRICE_CLASS_100,
    });

    // --- Step 1 content ---
    new s3deploy.BucketDeployment(this, 'Upload', {
      sources: [s3deploy.Source.data('index.html', '<h1>hello from cloudfront</h1>')],
      destinationBucket: bucket,
      distribution: dist,
      distributionPaths: ['/*'],
    });

    new cdk.CfnOutput(this, 'Url', { value: `https://${dist.distributionDomainName}` });
  }
}
```

```bash
cdk deploy
```

Wait 5–15 min for distribution to deploy (`Deployed` state, not `InProgress`).

Test:
```bash
curl -I https://<dist>.cloudfront.net/
# HTTP/2 200
# x-cache: Miss from cloudfront        (first hit)
# strict-transport-security: max-age=31536000; includeSubDomains
curl -I https://<dist>.cloudfront.net/
# x-cache: Hit from cloudfront          (second hit — cached)
```

## Step 2: Custom cache policy
> **Why:** The default "CachingOptimized" policy strips `Authorization`. If you cache auth-gated content, either (a) bypass cache on auth paths or (b) include `Authorization` in the cache key. Getting this wrong = leaking one user's data to another.

Already configured (`AuthCache` above). Verify the policy:
```bash
aws cloudfront get-cache-policy --id <policy-id>
```

## Step 3: Add ALB origin for /api/*
> **Why:** Mixed-origin distributions are common — static assets from S3, dynamic API from an ALB, same hostname. Let the behavior for `/api/*` bypass cache so API responses aren't shared between users.

Add to the stack (assumes you have an ALB from Lesson 02):
```typescript
dist.addBehavior('/api/*', new origins.LoadBalancerV2Origin(alb, {
  protocolPolicy: cf.OriginProtocolPolicy.HTTP_ONLY,
}), {
  cachePolicy: cf.CachePolicy.CACHING_DISABLED,
  originRequestPolicy: cf.OriginRequestPolicy.ALL_VIEWER,
  viewerProtocolPolicy: cf.ViewerProtocolPolicy.HTTPS_ONLY,
});
```

## Step 4: Signed URLs
> **Why:** Private content (premium videos, paid downloads) needs time-limited access. Signed URLs let you hand out "valid for 5 minutes" links without making the bucket public.

Generate a CloudFront key pair (console: CloudFront → Key Management → Public keys), then:

```bash
aws cloudfront sign \
  --url https://<dist>.cloudfront.net/premium/video.mp4 \
  --key-pair-id K2EXAMPLE \
  --private-key file://private_key.pem \
  --date-less-than "$(date -u -d '+5 min' '+%Y-%m-%dT%H:%M:%SZ')"
```

Output:
```
https://<dist>.cloudfront.net/premium/video.mp4?Expires=...&Signature=...&Key-Pair-Id=K2EXAMPLE
```

Attach the trusted key group to the behavior to enforce signing.

## Step 5: WAF — managed rules + rate limit + geo
> **Why:** Core Rule Set covers OWASP Top 10 patterns (SQLi, XSS, LFI). The rate-based rule is the cheapest effective DDoS mitigation at L7. Geo block stops traffic from countries you don't serve.

Already in the ACL above. Test the rate limit:
```bash
for i in $(seq 1 200); do curl -s -o /dev/null -w "%{http_code}\n" https://<dist>.cloudfront.net/; done | sort | uniq -c
```

Expected after ~100 requests in 5 min:
```
 100 200
 100 403
```

Inspect sampled requests in the WAF console → Sampled requests, or:
```bash
aws wafv2 get-sampled-requests --web-acl-arn <arn> --rule-metric-name rate \
  --scope CLOUDFRONT --max-items 20 \
  --time-window "StartTime=$(date -u -d '-1 hour' +%s),EndTime=$(date -u +%s)"
```

## Step 6: Geo block
> **Why:** Compliance and fraud both often require "no traffic from X." The `geoMatchStatement` does this at the edge — cheaper than letting it hit your origin.

Already configured (`KP` blocked). Test via VPN or override in a custom rule for your own country while testing.

## Step 7: Security headers via CloudFront Function
> **Why:** CF Functions run at every edge (not regional), sub-millisecond, $0.10/1M requests — ideal for tiny mutations. Lambda@Edge is the heavyweight alternative (full Node/Python runtime, more expensive).

Already attached. Verify:
```bash
curl -I https://<dist>.cloudfront.net/ | grep -iE 'strict-transport|x-frame|x-content'
```

## Cleanup
> **Why:** WAF ACL + rules bill monthly regardless of traffic. Distribution must be disabled before deletion (CDK handles this but it takes 10+ min).

```bash
cdk destroy
```

If it hangs: console → CloudFront → disable distribution → wait → retry destroy.

## Common Errors
- **403 AccessDenied from S3 via CloudFront** → OAC principal policy missing on bucket; re-run `cdk deploy` or add bucket policy manually.
- **Distribution update takes 20 min** → normal. Every config change propagates to 600 edges.
- **WAF rule "us-east-1 required"** → CloudFront-scoped ACLs must be in us-east-1. CDK prop `env.region` above.
- **`x-cache: Miss from cloudfront` always** → cache key includes something per-request (cookie, query string). Check cache policy.
- **Signed URL returns 403** → clock skew or wrong key-pair-id. Use UTC and the `K...` ID, not the KMS one.
- **CloudFront Function > 10 KB** → use Lambda@Edge instead.
