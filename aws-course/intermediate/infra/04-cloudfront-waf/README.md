# 04 — CloudFront + WAF

## Concept
CDN at AWS edge PoPs. Caches at origins. WAF = managed/custom rules at L7.

## Exercises
1. **Static site**: S3 bucket (block public) + CloudFront with OAC (Origin Access Control). Upload `index.html`.
2. **Cache policies**: create a custom cache policy that includes `Authorization` header. Compare TTLs.
3. **Behaviors**: add second origin (ALB) for `/api/*` path — API bypasses cache.
4. **Signed URLs**: create key pair, generate signed URL valid 5 min.
5. **WAF**: attach WebACL with AWS Managed Rules (Core Rule Set). Also a custom rule: rate-limit 100 req/5min per IP.
6. **Geo block**: deny access from one country. Test via VPN.
7. **Lambda@Edge**: minimal viewer-request function that adds security headers.

## Verification
- `curl` hits CloudFront (see `x-cache: Hit from cloudfront`).
- WAF rate limit triggers 403 after threshold.

## Gotchas
- CloudFront deploys take 5-15 min per update.
- WAF Managed Rules = $1/rule group/mo + requests.
- OAC replaces deprecated OAI.

## Cleanup
```bash
cdk destroy  # disable distribution first if it hangs
```
