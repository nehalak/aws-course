# Walkthrough — 05 API Gateway

## About this service
**API Gateway** is AWS's managed front door for APIs. Three variants:
- **HTTP API** — newer, cheaper ($1/M requests), faster, 70% of features. Default choice.
- **REST API** — older, feature-complete: API keys, usage plans, request validation, caching, WAF integration, private APIs. $3.50/M.
- **WebSocket API** — persistent bidirectional connections backed by Lambda.

Integrations include Lambda, HTTP(S) backends (via VPC Link for private ALB), Step Functions, and direct AWS service calls.

**Why it matters:** API Gateway is the canonical "put a URL on a Lambda" tool. It also handles the painful parts — TLS termination, throttling, authorizers, CORS — so your Lambda stays focused on business logic.

**When to use HTTP API:** Lambda-backed APIs, JWT auth, mobile/web backends.
**When to use REST API:** need API keys + usage plans, request/response transformations, per-method caching, or private APIs inside a VPC.
**When to use WebSocket API:** chat, live dashboards, collaborative editing — anything needing server push.
**When NOT to use:** high-traffic internal APIs (ALB + ECS is cheaper past ~10M req/day); gRPC (API GW doesn't speak HTTP/2 to backends well — use ALB).

## Estimated cost
- **HTTP API: $1.00/M requests**
- **REST API: $3.50/M requests** + **caching from ~$0.02/hour** per GB
- **WebSocket API: $1.00/M messages** + $0.25/M connection-minutes
- **Lambda: $0.20/M invocations** + compute time
- **Custom domain: free**; ACM cert: free
- Total for this lesson at 100K requests/month: **~$2/month**. Basically free.

---

## Step 1: Scaffold + HTTP API with Lambda
> **Why:** HTTP API is the modern starting point. Two routes + one Lambda gets you the full CRUD-stub pattern in about 40 lines of CDK.

```bash
mkdir api-gw && cd api-gw
cdk init app --language=typescript
npm install aws-cdk-lib
```

`lib/api-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigw2 from 'aws-cdk-lib/aws-apigatewayv2';
import * as integ from 'aws-cdk-lib/aws-apigatewayv2-integrations';
import * as auth from 'aws-cdk-lib/aws-apigatewayv2-authorizers';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class ApiStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const table = new dynamodb.Table(this, 'Items', {
      partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    const itemsFn = new lambda.Function(this, 'ItemsFn', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      environment: { TABLE: table.tableName },
      code: lambda.Code.fromInline(`
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, ScanCommand, PutCommand } = require('@aws-sdk/lib-dynamodb');
const doc = DynamoDBDocumentClient.from(new DynamoDBClient({}));
exports.handler = async (e) => {
  const method = e.requestContext.http.method;
  if (method === 'GET') {
    const r = await doc.send(new ScanCommand({ TableName: process.env.TABLE }));
    return { statusCode: 200, body: JSON.stringify(r.Items) };
  }
  if (method === 'POST') {
    const body = JSON.parse(e.body || '{}');
    await doc.send(new PutCommand({ TableName: process.env.TABLE, Item: { id: String(Date.now()), ...body } }));
    return { statusCode: 201, body: 'ok' };
  }
  return { statusCode: 405, body: '' };
};`),
    });
    table.grantReadWriteData(itemsFn);

    const api = new apigw2.HttpApi(this, 'HttpApi', {
      corsPreflight: {
        allowOrigins: ['*'], allowMethods: [apigw2.CorsHttpMethod.ANY],
        allowHeaders: ['Content-Type', 'Authorization'],
      },
    });

    const itemsInt = new integ.HttpLambdaIntegration('ItemsInt', itemsFn);
    api.addRoutes({ path: '/items', methods: [apigw2.HttpMethod.GET, apigw2.HttpMethod.POST], integration: itemsInt });

    new cdk.CfnOutput(this, 'ApiUrl', { value: api.apiEndpoint });
  }
}
```

Deploy:
```bash
cdk deploy
```

Test:
```bash
curl -X POST https://<api>.execute-api.us-east-1.amazonaws.com/items -d '{"name":"widget"}' -H 'Content-Type: application/json'
# ok
curl https://<api>.execute-api.us-east-1.amazonaws.com/items
# [{"id":"1710000000000","name":"widget"}]
```

## Step 2: JWT Lambda authorizer
> **Why:** Most real APIs need auth. A custom Lambda authorizer lets you validate any token format — JWT, Paseto, opaque session tokens. API Gateway caches the result per token for up to 1 hour.

Add to the stack:

```typescript
const authFn = new lambda.Function(this, 'Authorizer', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
exports.handler = async (e) => {
  const token = (e.headers.authorization || '').replace(/^Bearer\\s+/i, '');
  // demo: accept anything that decodes base64 to {"sub":"..."} ; real code would verify JWT signature
  try {
    const payload = JSON.parse(Buffer.from(token.split('.')[1] || '', 'base64').toString() || '{}');
    if (!payload.sub) return { isAuthorized: false };
    return { isAuthorized: true, context: { sub: payload.sub } };
  } catch { return { isAuthorized: false }; }
};`),
});

const authorizer = new auth.HttpLambdaAuthorizer('JwtAuth', authFn, {
  responseTypes: [auth.HttpLambdaResponseType.SIMPLE],
  resultsCacheTtl: cdk.Duration.minutes(5),
  identitySource: ['$request.header.Authorization'],
});

api.addRoutes({
  path: '/secure',
  methods: [apigw2.HttpMethod.GET],
  integration: itemsInt,
  authorizer,
});
```

Test:
```bash
curl -I https://<api>/secure
# HTTP/2 401
curl -I https://<api>/secure -H "Authorization: Bearer eyJhbGc.eyJzdWIiOiJ1MSJ9.sig"
# HTTP/2 200
```

## Step 3: REST API with usage plan + API key
> **Why:** HTTP API can't do API keys natively. For partner APIs where you throttle per customer, REST API + usage plan is still the canonical answer.

```typescript
import * as apigw from 'aws-cdk-lib/aws-apigateway';

const rest = new apigw.RestApi(this, 'Rest', { deployOptions: { stageName: 'prod' } });
const items = rest.root.addResource('items');
items.addMethod('GET', new apigw.LambdaIntegration(itemsFn), { apiKeyRequired: true });

const key = rest.addApiKey('DemoKey');
const plan = rest.addUsagePlan('Plan', {
  throttle: { rateLimit: 10, burstLimit: 20 },
  quota: { limit: 1000, period: apigw.Period.DAY },
});
plan.addApiKey(key);
plan.addApiStage({ stage: rest.deploymentStage });

new cdk.CfnOutput(this, 'RestUrl', { value: rest.url });
```

Test:
```bash
KEY=$(aws apigateway get-api-key --api-key <id> --include-value --query value --output text)
curl -H "x-api-key: $KEY" https://<rest>/prod/items     # 200
curl                       https://<rest>/prod/items     # 403 Forbidden
# Hammer it to hit the rate limit:
for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code} " -H "x-api-key: $KEY" https://<rest>/prod/items; done
# 200 200 200 ... 429 429 429
```

## Step 4: Request validation with JSON Schema
> **Why:** Rejecting bad input at the edge saves a Lambda invocation. The gateway can validate body + params against a JSON Schema before your code runs.

```typescript
const model = rest.addModel('ItemModel', {
  contentType: 'application/json',
  schema: {
    type: apigw.JsonSchemaType.OBJECT,
    required: ['name'],
    properties: {
      name:  { type: apigw.JsonSchemaType.STRING, minLength: 1, maxLength: 100 },
      price: { type: apigw.JsonSchemaType.NUMBER, minimum: 0 },
    },
  },
});
items.addMethod('POST', new apigw.LambdaIntegration(itemsFn), {
  apiKeyRequired: true,
  requestValidatorOptions: { validateRequestBody: true },
  requestModels: { 'application/json': model },
});
```

Test:
```bash
curl -X POST https://<rest>/prod/items -H "x-api-key: $KEY" -H 'Content-Type: application/json' -d '{}'
# {"message": "Invalid request body"}
```

## Step 5: CORS
> **Why:** Browsers block cross-origin requests unless the server says OK via CORS headers. Configuring at the gateway (not in every Lambda) is the clean way.

Already done for the HTTP API in Step 1 (`corsPreflight`). Test:
```bash
curl -i -X OPTIONS https://<api>/items \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: POST"
# HTTP/2 204
# access-control-allow-origin: *
# access-control-allow-methods: *
```

## Step 6: WebSocket API (broadcast chat)
> **Why:** WebSocket API holds connections so you can push data. The canonical pattern: store `connectionId` in DynamoDB on `$connect`, iterate on `sendMessage` to fan out.

```typescript
import * as apigw2ws from 'aws-cdk-lib/aws-apigatewayv2';

const connections = new dynamodb.Table(this, 'Conns', {
  partitionKey: { name: 'connectionId', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

const wsFn = new lambda.Function(this, 'WsFn', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  environment: { TABLE: connections.tableName },
  code: lambda.Code.fromInline(`
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, DeleteCommand, ScanCommand } = require('@aws-sdk/lib-dynamodb');
const { ApiGatewayManagementApiClient, PostToConnectionCommand } = require('@aws-sdk/client-apigatewaymanagementapi');
const doc = DynamoDBDocumentClient.from(new DynamoDBClient({}));
exports.handler = async (e) => {
  const { routeKey, connectionId, domainName, stage } = e.requestContext;
  if (routeKey === '$connect') {
    await doc.send(new PutCommand({ TableName: process.env.TABLE, Item: { connectionId } }));
    return { statusCode: 200 };
  }
  if (routeKey === '$disconnect') {
    await doc.send(new DeleteCommand({ TableName: process.env.TABLE, Key: { connectionId } }));
    return { statusCode: 200 };
  }
  // sendMessage
  const conns = await doc.send(new ScanCommand({ TableName: process.env.TABLE }));
  const client = new ApiGatewayManagementApiClient({ endpoint: \`https://\${domainName}/\${stage}\` });
  const msg = JSON.parse(e.body).message;
  await Promise.all(conns.Items.map(c =>
    client.send(new PostToConnectionCommand({ ConnectionId: c.connectionId, Data: Buffer.from(msg) }))
      .catch(() => doc.send(new DeleteCommand({ TableName: process.env.TABLE, Key: { connectionId: c.connectionId } })))
  ));
  return { statusCode: 200 };
};`),
});
connections.grantReadWriteData(wsFn);

const wsApi = new apigw2ws.WebSocketApi(this, 'WsApi', {
  connectRouteOptions:    { integration: new integ.WebSocketLambdaIntegration('C', wsFn) },
  disconnectRouteOptions: { integration: new integ.WebSocketLambdaIntegration('D', wsFn) },
  defaultRouteOptions:    { integration: new integ.WebSocketLambdaIntegration('M', wsFn) },
});
const wsStage = new apigw2ws.WebSocketStage(this, 'WsStage', { webSocketApi: wsApi, stageName: 'prod', autoDeploy: true });
wsApi.grantManageConnections(wsFn);

new cdk.CfnOutput(this, 'WsUrl', { value: wsStage.url });
```

Test with two terminals:
```bash
wscat -c wss://<ws>.execute-api.us-east-1.amazonaws.com/prod
> {"action":"sendMessage","message":"hi"}
< hi
```

## Step 7: Custom domain
> **Why:** `<random>.execute-api.amazonaws.com` isn't brandable. Map a real domain via ACM + API Gateway domain name.

```typescript
import * as acm from 'aws-cdk-lib/aws-certificatemanager';

const cert = acm.Certificate.fromCertificateArn(this, 'Cert', 'arn:aws:acm:us-east-1:...:certificate/...');
const domain = new apigw2.DomainName(this, 'Domain', { domainName: 'api.example.com', certificate: cert });
new apigw2.ApiMapping(this, 'Mapping', { api, domainName: domain });
new cdk.CfnOutput(this, 'TargetCloudFrontDomain', { value: domain.regionalDomainName });
// then point api.example.com CNAME → domain.regionalDomainName in Route 53
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **502 Bad Gateway from HTTP API** → Lambda returned non-conforming response. Proxy shape must be `{ statusCode, headers, body }`.
- **CORS fails in browser but curl works** → browsers send preflight OPTIONS; make sure gateway handles OPTIONS (HTTP API does automatically if `corsPreflight` is set).
- **Authorizer allows everything** → cache TTL hiding bugs. Set `resultsCacheTtl: Duration.seconds(0)` while debugging.
- **WebSocket 403 on connect** → missing `grantManageConnections` on the Lambda or wrong endpoint URL.
- **Usage-plan throttling doesn't trigger** → API key not associated with plan, or request missing `x-api-key` header.
- **"$default stage" surprise deploys** → using `apigw2.HttpApi` without explicit stage auto-creates `$default`; lock to named stage for prod.
