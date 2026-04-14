# 02 — Multi-AZ with Load Balancers

## Concept
ALB = L7 (HTTP/S, path/host routing). NLB = L4 (TCP/UDP, static IP). Target groups = health + routing.

## Exercises
1. **ALB + 2 target groups**: CDK ALB in public subnets. TG-A = Fargate service, TG-B = EC2. Path `/api/*` → A, `/*` → B.
2. **Health checks**: point to `/health`. Stop one backend; watch `UnHealthyHostCount` alarm.
3. **Sticky sessions**: enable cookie stickiness on a TG. Curl and verify cookie.
4. **NLB**: create NLB for TCP pass-through to EC2 on port 5432. Note NLB has static IP per AZ.
5. **WebSocket**: deploy a WS echo server on Fargate behind ALB. Connect via `wscat`.
6. **HTTP to HTTPS redirect**: add listener on 80 that redirects to 443.

## Verification
- `/api/health` returns ALB → Fargate; `/` returns EC2.
- Stopped target marked unhealthy within 2 checks.

## Gotchas
- ALB can't do static IP; NLB can.
- Health check path must return 200 (not 302).
- Cross-zone balancing: ALB = free, NLB = $$$ per GB.

## Cleanup
```bash
cdk destroy
```
