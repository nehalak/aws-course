# 04 — Step Functions

## Concept
Orchestrate Lambdas/AWS services with a state machine. Standard = durable long-running. Express = high-volume short.

## Exercises
1. **Linear flow**: Task → Task → Task. 3 Lambdas: validate → process → notify.
2. **Choice + parallel**: branch on input, parallel fan-out to 3 Lambdas, merge results.
3. **Map state**: iterate over array; process each item in parallel; limit concurrency to 10.
4. **Error handling**: `Retry` with exponential backoff (max 3), `Catch` → fallback state.
5. **Saga pattern**: 3 steps with compensating actions on failure (book → pay → confirm; undo on fail).
6. **Direct integrations**: call DynamoDB PutItem, SNS Publish, SQS SendMessage without Lambda.
7. **Callback pattern**: `.waitForTaskToken` → external system POSTs back to resume execution.

## Verification
- Visual graph in console shows all branches.
- Saga rollback triggers on induced failure.

## Gotchas
- Standard: $25/M state transitions. Express: $1/M + duration.
- ASL (Amazon States Language) — use CDK `sfn.Chain` for readability.
- Map state concurrency limits.

## Cleanup
```bash
cdk destroy
```
