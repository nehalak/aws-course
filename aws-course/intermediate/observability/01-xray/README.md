# 01 — X-Ray

## Concept
Distributed tracing. Segments (service) + subsegments (calls). Service map shows topology.

## Exercises
1. **Instrument Lambda**: `@aws-lambda-powertools/tracer` — add segments around DynamoDB and HTTP calls.
2. **API GW → Lambda → DynamoDB** trace: end-to-end trace in X-Ray console.
3. **Sampling rules**: create custom rule: 100% of `/checkout`, 1% of `/health`.
4. **Analytics**: find p95 latency by service, errors by URL.
5. **Cross-service**: add second Lambda called by first via SDK. Trace context propagates.
6. **ADOT (OpenTelemetry)**: alternative — use OTel collector to send traces to X-Ray + Honeycomb.

## Verification
- Service map shows full request path.
- Sampling rules applied as expected.

## Gotchas
- X-Ray SDK v2 vs v3; OTel is future-proof.
- Trace header `X-Amzn-Trace-Id` must propagate across HTTP calls.

## Cleanup
```bash
cdk destroy
```
