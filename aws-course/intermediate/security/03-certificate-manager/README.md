# 03 — Certificate Manager (ACM)

## Concept
Free public TLS certs for AWS services. Private CA for internal mTLS. Only works on ALB, CloudFront, API GW, NLB (passthrough).

## Exercises
1. **Public cert**: request for `*.<yourdomain>`. DNS-validate by adding CNAME in Route 53 (auto if same account).
2. **Attach to ALB**: add HTTPS listener with ACM cert. Redirect HTTP → HTTPS.
3. **CloudFront cert**: must be in `us-east-1`. Request there; attach to distribution.
4. **Private CA**: create ACM PCA (expensive — $400/mo; use SHORT_LIVED mode $50/mo). Issue a cert for `internal.mycompany`.
5. **mTLS on ALB**: attach trust store; require client cert. Test with `curl --cert`.

## Verification
- `curl https://<yourdomain>` shows valid TLS.
- mTLS fails without `--cert`.

## Gotchas
- CloudFront cert MUST be `us-east-1`.
- ACM public cert auto-renews only if DNS records stay put.
- Private CA is expensive — turn off after exercise.

## Cleanup
```bash
cdk destroy
# delete private CA (30-day restore window)
```
