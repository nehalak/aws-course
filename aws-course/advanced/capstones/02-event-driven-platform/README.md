# Capstone 02 — Event-Driven Platform

## Goal
Build an order-processing platform with EventBridge as central bus, Step Functions orchestration, SQS buffering.

## Requirements
- EventBridge custom bus `orders`
- 5 services as Fargate tasks publishing events (OrderCreated, PaymentAuthorized, InventoryReserved, OrderShipped, OrderCancelled)
- Step Functions saga for order fulfillment with compensations
- SQS FIFO queue for email notifications with DLQ
- Archive + replay on bus for debugging
- Schema registry for events
- EventBridge Pipes: DynamoDB stream → transform → SQS target

## Deliverables
- CDK app
- `architecture.md` with sequence diagrams
- Load test 1k orders/sec; observe e2e latency
- Chaos test: kill payment service mid-saga; verify compensation

## Verification
- Saga completes happy path.
- Compensation invoked + inventory released on payment failure.

## Gotchas
- EventBridge at-least-once; dedupe downstream.
- Schema evolution: additive only.

## Cleanup
```bash
cdk destroy
```
