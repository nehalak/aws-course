# 01 — IoT Core

## Concept
MQTT broker + device registry + shadows + rules engine. Greengrass = edge runtime.

## Exercises
1. **Register a thing**: provision cert via CDK. Simulate device using `mosquitto_pub` with cert.
2. **Pub/Sub**: device publishes `devices/d1/telemetry`; another client subscribes.
3. **Device shadow**: set desired state via REST; device sync via MQTT `$shadow/update/delta`.
4. **Rules engine**: SQL rule `SELECT * FROM 'devices/+/telemetry' WHERE temp > 80` → SNS + DynamoDB + Kinesis.
5. **Jobs**: deploy a firmware-update job to a group of things.
6. **Greengrass**: run on EC2 (as edge). Deploy a Lambda component; run locally.
7. **Fleet indexing**: search for things by attribute.

## Verification
- Rule fires on matching telemetry.
- Shadow delta received by simulated device.

## Gotchas
- MQTT topic hierarchy matters for policy.
- Device cert policies — scope tightly by `iot:ClientId` / topic.

## Cleanup
```bash
cdk destroy
```
