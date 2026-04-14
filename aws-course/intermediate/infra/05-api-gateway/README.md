# 05 — API Gateway

## Concept
REST API = feature-rich, $3.50/M. HTTP API = simpler/cheaper ($1/M). WebSocket API for realtime.

## Exercises
1. **HTTP API → Lambda**: CDK HTTP API with 2 routes: `GET /items`, `POST /items`. Integrate with Lambda.
2. **Lambda authorizer** (JWT): verify a bearer token. Reject 401 without token.
3. **REST API with usage plan**: API key + quota 1000/day, rate 10 req/sec.
4. **Request validation**: JSON schema for `POST /items` body. Reject invalid requests before Lambda.
5. **CORS**: enable CORS for browser origin. Test from a local HTML page.
6. **WebSocket API**: `$connect`, `$disconnect`, `sendMessage` routes wired to Lambda. Store `connectionId` in DynamoDB; broadcast to all.
7. **Custom domain**: map `api.<yourdomain>` to HTTP API via ACM cert.

## Verification
- `curl` with key succeeds; without → 403.
- WS echo works via `wscat -c <url>`.

## Gotchas
- REST API caching = per-stage $$$.
- `$default` stage is easy trap — use `dev`, `prod` stages.
- Lambda proxy integration: handler must return `{statusCode, body, headers}`.

## Cleanup
```bash
cdk destroy
```
