# 02 — Lambda@Edge vs CloudFront Functions

## Concept
CF Functions = sub-ms, JS only, viewer event, cheap. Lambda@Edge = Node/Python, 4 event types, full SDK, slower.

## Exercises
1. **CF Function** (viewer-request): A/B test cookie assignment.
2. **CF Function** (viewer-response): add security headers (HSTS, X-Frame-Options, CSP).
3. **Lambda@Edge origin-request**: URL rewriting (`/users/123` → S3 key `users/123.json`).
4. **Lambda@Edge origin-response**: mutate response body (image resize proxy with Sharp).
5. **Cost compare**: 1M req/day — calculate CF Functions vs Edge bill.
6. **Decision doc**: `cf-func-vs-edge.md` when each fits.

## Verification
- Security headers visible via `curl -I`.
- URL rewrite works end-to-end.

## Gotchas
- Lambda@Edge deploy takes ~5 min to propagate.
- Edge Lambda region = `us-east-1` only.
- CF Functions: no network calls allowed.

## Cleanup
```bash
cdk destroy
```
