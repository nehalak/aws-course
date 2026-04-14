# Walkthrough — 01 IoT Core

## About this service
**AWS IoT Core** is a managed MQTT broker with a device registry, X.509 certificate auth, a **rules engine** (SQL over topic streams), **device shadows** (a JSON twin that represents the desired/reported state of a device), **jobs** (remote command/firmware rollouts), and **fleet indexing** (queryable registry). **Greengrass** is the optional edge runtime that lets you run Lambda, containers, and ML models on the device itself and sync with IoT Core opportunistically.

**Why it matters:** IoT fleets are auth-heavy, stateful, and bandwidth-constrained. IoT Core handles mTLS for millions of devices, decouples producers from consumers via topics, and lets you fan out telemetry into Kinesis/SNS/DynamoDB via one SQL rule instead of writing glue.
**When to use:** fleets of devices speaking MQTT/HTTPS, firmware OTA, remote config via shadow, low-power sensors.
**When NOT to use:** a handful of always-online servers (just use SQS/Kinesis directly); ultra-low-latency local control loops (use Greengrass + local pub/sub, not cloud round-trip).

## Estimated cost
- **Connectivity: $0.08 per million minutes** of connection (1 device always-on for a month = 43,200 min = $0.0035)
- **Messaging: $1.00 per million messages** (<=5KB each)
- **Rules engine: $0.15 per million rules triggered** + $0.18 per million actions executed
- **Device Shadow / Registry: $1.25 per million operations**
- **Greengrass: $0.16/device/month** (core devices, first 10,000)
- Downstream: SNS ($0.50/M), DynamoDB on-demand (~$1.25/M writes), Kinesis shard ($0.015/hr = ~$11/mo)
- Total for this lesson: **~$12/month** (mostly the Kinesis shard; drop it and you're at ~$1/month)

---

## Step 1: Bootstrap CDK stack + IoT Thing with cert
> **Why:** Every device needs an identity. The pattern is: create a Thing (logical record), create an X.509 cert keypair, attach a Policy to the cert that scopes what topics it can publish/subscribe on, then attach the cert to the Thing. CDK provisions all four in one go.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as iot from 'aws-cdk-lib/aws-iot';
import { Construct } from 'constructs';

export class IotCoreStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const thing = new iot.CfnThing(this, 'Device1', { thingName: 'd1' });

    // Policy scoped tightly by clientId + topic
    const policy = new iot.CfnPolicy(this, 'DevicePolicy', {
      policyName: 'd1-policy',
      policyDocument: {
        Version: '2012-10-17',
        Statement: [
          { Effect: 'Allow', Action: ['iot:Connect'],
            Resource: `arn:aws:iot:${this.region}:${this.account}:client/\${iot:ClientId}`,
            Condition: { 'Bool': { 'iot:Connection.Thing.IsAttached': 'true' } } },
          { Effect: 'Allow', Action: ['iot:Publish'],
            Resource: [
              `arn:aws:iot:${this.region}:${this.account}:topic/devices/\${iot:ClientId}/telemetry`,
              `arn:aws:iot:${this.region}:${this.account}:topic/$aws/things/\${iot:ClientId}/shadow/update`,
            ] },
          { Effect: 'Allow', Action: ['iot:Subscribe', 'iot:Receive'],
            Resource: [
              `arn:aws:iot:${this.region}:${this.account}:topicfilter/$aws/things/\${iot:ClientId}/shadow/update/delta`,
              `arn:aws:iot:${this.region}:${this.account}:topic/$aws/things/\${iot:ClientId}/shadow/update/delta`,
            ] },
        ],
      },
    });

    new cdk.CfnOutput(this, 'Endpoint', {
      value: `Run: aws iot describe-endpoint --endpoint-type iot:Data-ATS`,
    });
  }
}
```

Create the certificate outside CDK (it returns private key material which shouldn't land in CFN outputs):

```bash
aws iot create-keys-and-certificate --set-as-active \
  --certificate-pem-outfile d1.cert.pem \
  --public-key-outfile d1.public.key \
  --private-key-outfile d1.private.key \
  --query 'certificateArn' --output text > d1.cert.arn

aws iot attach-policy --policy-name d1-policy --target "$(cat d1.cert.arn)"
aws iot attach-thing-principal --thing-name d1 --principal "$(cat d1.cert.arn)"

# Amazon Root CA
curl -o AmazonRootCA1.pem https://www.amazontrust.com/repository/AmazonRootCA1.pem
```

## Step 2: Simulate the device with mosquitto_pub
> **Why:** Before writing firmware, prove the identity + topic policy works with a commodity MQTT client. `mosquitto_pub` connects with mTLS exactly the way an embedded client would.

```bash
ENDPOINT=$(aws iot describe-endpoint --endpoint-type iot:Data-ATS --query endpointAddress --output text)

mosquitto_pub \
  -h "$ENDPOINT" -p 8883 \
  -i d1 \
  --cafile AmazonRootCA1.pem \
  --cert d1.cert.pem \
  --key d1.private.key \
  -t 'devices/d1/telemetry' \
  -m '{"temp":92,"hum":54,"ts":1718553600}' -d
```

In another terminal, subscribe with the AWS CLI / MQTT test client (IoT console → MQTT test client → subscribe `devices/+/telemetry`). You should see the payload.

## Step 3: Device Shadow — desired/reported sync
> **Why:** Shadows are how you tell an offline device "new config please" without caring when it reconnects. Your app writes `desired`; when the device comes online it pulls a `delta` and applies, then publishes `reported`.

Write desired state via REST:

```bash
aws iot-data update-thing-shadow --thing-name d1 \
  --payload '{"state":{"desired":{"sampling_hz":5}}}' \
  /dev/stdout
```

Device subscribes and listens for delta:

```bash
mosquitto_sub -h "$ENDPOINT" -p 8883 -i d1 \
  --cafile AmazonRootCA1.pem --cert d1.cert.pem --key d1.private.key \
  -t '$aws/things/d1/shadow/update/delta' -v
# {"version":3,"state":{"sampling_hz":5},...}
```

Device acknowledges by reporting:

```bash
mosquitto_pub -h "$ENDPOINT" -p 8883 -i d1 \
  --cafile AmazonRootCA1.pem --cert d1.cert.pem --key d1.private.key \
  -t '$aws/things/d1/shadow/update' \
  -m '{"state":{"reported":{"sampling_hz":5}}}'
```

## Step 4: Rules engine — fan out hot telemetry
> **Why:** Telemetry is noisy. You want to store everything cheaply (DynamoDB), archive for analytics (Kinesis → Firehose → S3), and page a human only when a threshold trips (SNS). One SQL rule does all three — no Lambda needed.

```typescript
import * as sns from 'aws-cdk-lib/aws-sns';
import * as ddb from 'aws-cdk-lib/aws-dynamodb';
import * as kinesis from 'aws-cdk-lib/aws-kinesis';
import * as iam from 'aws-cdk-lib/aws-iam';

const topic = new sns.Topic(this, 'AlertTopic');
const table = new ddb.Table(this, 'Telemetry', {
  partitionKey: { name: 'thingName', type: ddb.AttributeType.STRING },
  sortKey: { name: 'ts', type: ddb.AttributeType.NUMBER },
  billingMode: ddb.BillingMode.PAY_PER_REQUEST,
});
const stream = new kinesis.Stream(this, 'Stream', { shardCount: 1 });

const role = new iam.Role(this, 'RuleRole', {
  assumedBy: new iam.ServicePrincipal('iot.amazonaws.com'),
});
topic.grantPublish(role);
table.grantWriteData(role);
stream.grantWrite(role);

new iot.CfnTopicRule(this, 'HotTempRule', {
  ruleName: 'hot_temp',
  topicRulePayload: {
    sql: "SELECT *, topic(2) AS thingName FROM 'devices/+/telemetry' WHERE temp > 80",
    awsIotSqlVersion: '2016-03-23',
    ruleDisabled: false,
    actions: [
      { sns: { targetArn: topic.topicArn, roleArn: role.roleArn } },
      { dynamoDBv2: { putItem: { tableName: table.tableName }, roleArn: role.roleArn } },
      { kinesis: { streamName: stream.streamName, partitionKey: '${thingName}', roleArn: role.roleArn } },
    ],
  },
});
```

Publish `temp=92` from Step 2 — verify SNS email, DDB item, Kinesis record all appear.

## Step 5: Jobs — OTA firmware rollout
> **Why:** You don't want to SSH to 10,000 devices. A Job document lives in S3, targets a Thing Group, and the device-side agent pulls it, applies it, reports status. Rolling rollout + abort criteria are built in.

```bash
# Group and tag devices
aws iot create-thing-group --thing-group-name production
aws iot add-thing-to-thing-group --thing-group-name production --thing-name d1

# Job doc
cat > firmware-job.json <<'EOF'
{ "operation": "install", "url": "https://example.com/firmware/1.2.3.bin", "checksum": "sha256:abc..." }
EOF
aws s3 cp firmware-job.json s3://my-iot-bucket/jobs/firmware-1.2.3.json

aws iot create-job \
  --job-id fw-1-2-3 \
  --targets "arn:aws:iot:us-east-1:111111111111:thinggroup/production" \
  --document-source "https://my-iot-bucket.s3.amazonaws.com/jobs/firmware-1.2.3.json" \
  --target-selection CONTINUOUS \
  --job-executions-rollout-config 'maximumPerMinute=50' \
  --abort-config 'criteriaList=[{failureType=FAILED,action=CANCEL,thresholdPercentage=10,minNumberOfExecutedThings=20}]'
```

Device side subscribes to `$aws/things/d1/jobs/notify-next` and updates status via `$aws/things/d1/jobs/<jobId>/update`.

## Step 6: Greengrass core on EC2 (edge simulation)
> **Why:** Greengrass runs on the device (a gateway, a Pi, a factory PLC host). It gives you local pub/sub, local Lambda execution, stream management with store-and-forward when cloud is unreachable. An EC2 instance is a fine stand-in for the class.

```bash
# On an Amazon Linux 2023 EC2 (t3.small, IAM role with GreengrassV2TokenExchangeRole)
sudo dnf install -y java-17-amazon-corretto
curl -s https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-nucleus-latest.zip -o gg.zip
unzip gg.zip -d GreengrassInstaller

sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
  -jar ./GreengrassInstaller/lib/Greengrass.jar \
  --aws-region us-east-1 \
  --thing-name ggc-core-1 --thing-group-name ggc-group \
  --component-default-user ggc_user:ggc_group \
  --provision true --setup-system-service true --deploy-dev-tools true
```

Deploy a local Lambda component via the console: Component → Create → upload zipped Python handler that subscribes to local topic `local/heartbeat` and prints. Verify with `gg-cli`:

```bash
sudo /greengrass/v2/bin/greengrass-cli component list
```

## Step 7: Fleet Indexing — query the registry
> **Why:** "How many devices on firmware <1.2.3 in region EU?" The registry isn't a database by default — enable fleet indexing and you get a Lucene-style query.

```bash
aws iot update-indexing-configuration \
  --thing-indexing-configuration 'thingIndexingMode=REGISTRY_AND_SHADOW,thingConnectivityIndexingMode=STATUS'

aws iot search-index --query-string 'shadow.reported.fw_version:1.2.2'
```

## Cleanup
```bash
# Detach & delete cert
aws iot detach-policy --policy-name d1-policy --target "$(cat d1.cert.arn)"
aws iot detach-thing-principal --thing-name d1 --principal "$(cat d1.cert.arn)"
CERT_ID=$(cut -d/ -f2 < d1.cert.arn)
aws iot update-certificate --certificate-id "$CERT_ID" --new-status INACTIVE
aws iot delete-certificate --certificate-id "$CERT_ID"

aws iot delete-job --job-id fw-1-2-3 --force
aws iot delete-thing-group --thing-group-name production
cdk destroy
# Greengrass: systemctl stop greengrass; terminate EC2
```

## Common Errors
- **`CLIENT_ERROR` / connection refused immediately** → policy denies Connect. Check `iot:ClientId` placeholder matches the MQTT `-i` client ID.
- **Shadow delta never arrives** → device subscribed before connecting, or QoS 0 + reconnect lost retained delta. Subscribe after `CONNACK`, use QoS 1.
- **Rule fires but DDB row missing** → role lacks `dynamodb:PutItem`; check the IoT rule's CloudWatch logs (`AWSIotLogsV2`).
- **Greengrass install fails with TES error** → EC2 instance role missing `GreengrassV2TokenExchangeRoleAccess`.
- **`ResourceAlreadyExists` on second cert attach** → a cert can attach to only one policy in the same statement; detach first.
