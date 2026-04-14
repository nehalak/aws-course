# 02 — Lambda Internals

## Concept
Firecracker microVMs. Init → Invoke → Shutdown phases. Extensions for observability. Response streaming.

## Exercises
1. **Measure cold start phases**: Powertools logs showing Init vs Invoke durations across 10 cold starts. Plot.
2. **Tune init cost**: move SDK clients + DB connection to handler-outer-scope vs inner scope — measure.
3. **Lambda extensions**: write a minimal internal extension (Node) logging lifecycle events.
4. **External extension** (Go) to ship logs out-of-band.
5. **Response streaming**: use `awslambda.streamifyResponse` to stream a large payload; measure TTFB.
6. **Shutdown phase**: log "shutdown" via extension; invoke → wait 15 min idle → observe.
7. **Memory vs CPU plot**: benchmark same CPU task at 128MB..10GB; plot cost-vs-duration curve. Find sweet spot.

## Verification
- Init < 200ms for small Node function.
- Streaming response shows TTFB << full duration.

## Gotchas
- Shutdown not guaranteed (<500ms).
- Extensions count against 10GB memory total.

## Cleanup
```bash
cdk destroy
```
