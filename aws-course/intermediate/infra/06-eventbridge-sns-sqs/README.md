# 06 — EventBridge, SNS, SQS

## Concept
SNS = pub/sub fan-out (push). SQS = queue (pull). EventBridge = event bus with pattern matching + scheduling.

## Exercises
1. **SQS standard + Lambda consumer**: producer Lambda sends messages; consumer reads batches of 10. Add DLQ after 3 failures.
2. **FIFO queue**: same but with `MessageGroupId` ordering. Verify order preservation.
3. **SNS → SQS fan-out**: topic with 2 SQS subscriptions + 1 email subscription. Publish once → all three receive.
4. **SNS filter policy**: subscription filters on `eventType = "order.created"`. Publish with/without matching attribute.
5. **EventBridge default bus**: rule matching `"source": ["aws.ec2"]` → Lambda. Start an EC2; observe Lambda fire.
6. **Custom bus**: publish custom events (`PutEvents`); route by pattern. Add archive + replay.
7. **Scheduler**: EventBridge Scheduler that runs a Lambda every Mon 9am.

## Verification
- DLQ collects poison messages after retry exhaustion.
- FIFO consumer sees order per group.

## Gotchas
- SQS visibility timeout must exceed Lambda timeout.
- EventBridge event size max 256KB.
- FIFO throughput: 300 msg/sec per group without high throughput mode.

## Cleanup
```bash
cdk destroy
```
