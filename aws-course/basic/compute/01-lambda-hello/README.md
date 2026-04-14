# 01 — Lambda Hello

## Concept
Function as a Service. Run code without managing servers. Event-driven. Pay per 1ms execution.

## Exercises
1. **CDK Node Lambda**: simple handler that returns `{ statusCode: 200, body: "hello" }`. Use `NodejsFunction` (esbuild bundling).
2. **Invoke**:
   ```bash
   aws lambda invoke --function-name <name> --payload '{}' out.json
   cat out.json
   ```
3. **Environment variables**: pass `GREETING=hola` via env, read in handler.
4. **Cold start observation**: invoke once (cold), then rapidly 10x (warm). Compare `Init Duration` in CloudWatch Logs.
5. **Memory tuning**: set memory 128MB, 512MB, 1024MB. Measure billed duration for a CPU-bound task (loop 1M times).
6. **Timeout**: set 3 sec, make handler `setTimeout` 5 sec. Observe timeout error in logs.
7. **Python variant**: same exercise in Python runtime. Compare cold starts Node vs Python.

## Verification
- Log stream shows each invocation.
- Memory 1024 finishes CPU task ~4x faster than 128 (CPU scales with memory).

## Gotchas
- Default timeout 3 sec — too short for most things.
- Lambda logs cost CloudWatch $0.50/GB ingested.
- Free tier: 1M invocations + 400K GB-sec.

## Cleanup
```bash
cdk destroy
```
