# 02 — DynamoDB Basics

## Concept
Managed NoSQL key-value + document. Single-digit ms latency. Partition key determines physical placement.

## Exercises
1. **CDK table** `Users`: PK = `userId` (string), billing = `PAY_PER_REQUEST`, encryption AWS-managed.
2. **CRUD via CLI**:
   ```bash
   aws dynamodb put-item --table-name Users --item '{"userId":{"S":"u1"},"name":{"S":"Alice"}}'
   aws dynamodb get-item --table-name Users --key '{"userId":{"S":"u1"}}'
   aws dynamodb query --table-name Users --key-condition-expression "userId = :u" --expression-attribute-values '{":u":{"S":"u1"}}'
   ```
3. **Composite key**: new table `Orders` PK=`userId` SK=`orderId`. Query all orders for a user.
4. **Scan vs Query**: populate 1000 items. Time a `scan` vs a `query` by PK. Understand why scan is bad.
5. **Provisioned vs on-demand**: switch billing mode; understand RCU/WCU math.
6. **TTL**: enable TTL on `expiresAt` attribute. Insert item with expiry 5 min out. Wait, observe deletion (up to 48h in practice).

## Verification
- Query returns items. Scan returns all.
- Item with TTL eventually disappears.

## Gotchas
- Scan reads entire table — use only as last resort.
- Hot partition = throttling even with plenty of capacity.
- On-demand is 7x more expensive per request than provisioned at scale.

## Cleanup
```bash
cdk destroy
```
