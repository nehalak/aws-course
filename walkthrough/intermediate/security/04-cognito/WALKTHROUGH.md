# Walkthrough — 04 Cognito

## About this service
**Cognito** is two products sharing a name. **User Pools** are a user directory — sign-up, sign-in, MFA, password reset, JWT issuance. Think "hosted auth server." **Identity Pools (aka Federated Identities)** exchange any JWT (Cognito, Google, SAML) for **temporary AWS IAM credentials** so a browser can call AWS directly with per-user scoping. You usually use User Pool for login and (optionally) Identity Pool for direct AWS access. Cognito supports **Hosted UI**, **social IdPs** (Google, Facebook, Apple, SAML, OIDC), **Lambda triggers** at every lifecycle step, and integrates with **API Gateway JWT authorizers** and **ALB authentication**.

**Why it matters:** rolling your own auth is a security nightmare. Cognito gives you SOC-2-grade identity in an afternoon: email verification, brute-force lockout, MFA, token refresh — all wired up.

**When to use:** B2C apps with email/social login, mobile apps, any app where you want AWS to handle password storage.
**When NOT to use:** enterprise SSO (use Entra ID / Okta directly with API Gateway OIDC authorizer). Also: Cognito migration is painful — if you might outgrow it, start with Auth0/Clerk.

## Estimated cost
- Monthly Active Users (MAUs): **first 10,000 free**, then **$0.0055/MAU** up to 100k
- MFA SMS: **$0.05 per message** (SMS to US)
- Hosted UI custom domain: **$0.00** (but requires ACM cert in us-east-1 — included elsewhere)
- Identity Pool: **free** (you pay only for the AWS actions the temp creds invoke)
- Total for this lesson (dev traffic): **~$0/month** (under free tier)

---

## Step 1: CDK User Pool with MFA + app client
> **Why:** The User Pool is the directory. MFA-optional means users can opt in; email verification blocks bot signups; the app client is what your SPA uses to authenticate (no client secret for browser clients).

`lib/cognito-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import { Construct } from 'constructs';

export class CognitoStack extends cdk.Stack {
  public readonly pool: cognito.UserPool;
  public readonly client: cognito.UserPoolClient;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.pool = new cognito.UserPool(this, 'Pool', {
      userPoolName: 'demo-pool',
      selfSignUpEnabled: true,
      signInAliases: { email: true },
      autoVerify: { email: true },
      standardAttributes: { email: { required: true, mutable: false } },
      passwordPolicy: {
        minLength: 12,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true,
      },
      mfa: cognito.Mfa.OPTIONAL,
      mfaSecondFactor: { sms: false, otp: true },
      accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    this.client = this.pool.addClient('SpaClient', {
      authFlows: { userSrp: true },
      oAuth: {
        flows: { authorizationCodeGrant: true },
        scopes: [cognito.OAuthScope.OPENID, cognito.OAuthScope.EMAIL],
        callbackUrls: ['http://localhost:5173/callback'],
        logoutUrls: ['http://localhost:5173'],
      },
      generateSecret: false,  // SPA = public client
    });

    this.pool.addDomain('Domain', {
      cognitoDomain: { domainPrefix: 'demo-' + this.account },
    });

    new cdk.CfnOutput(this, 'PoolId', { value: this.pool.userPoolId });
    new cdk.CfnOutput(this, 'ClientId', { value: this.client.userPoolClientId });
    new cdk.CfnOutput(this, 'HostedUi', {
      value: `https://demo-${this.account}.auth.${this.region}.amazoncognito.com/login?client_id=${this.client.userPoolClientId}&response_type=code&scope=openid+email&redirect_uri=http://localhost:5173/callback`,
    });
  }
}
```

## Step 2: Test Hosted UI + inspect JWT
> **Why:** Hosted UI is the fastest way to wire up login without writing forms. Inspecting the JWT reveals what claims your backend can trust.

Open the `HostedUi` output URL. Sign up → verify email → sign in. You'll land on `localhost:5173/callback?code=...`. Exchange the code:

```bash
curl -X POST https://demo-$ACCOUNT.auth.us-east-1.amazoncognito.com/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&client_id=$CLIENT_ID&code=$CODE&redirect_uri=http://localhost:5173/callback"
```

Expected response:
```json
{
  "id_token": "eyJraWQiOi...",
  "access_token": "eyJraWQiOi...",
  "refresh_token": "eyJjdHkiOi...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

Decode at [jwt.io](https://jwt.io) — id_token payload shows `email`, `cognito:username`, `aud`, `iss`, `exp`.

## Step 3: Add Google as federated IdP
> **Why:** 70% of consumer users prefer social sign-in. Cognito handles the OAuth dance; Google users land in the same User Pool so your app only cares about the JWT.

Get a Google OAuth client from Google Cloud Console. Then:
```typescript
new cognito.UserPoolIdentityProviderGoogle(this, 'Google', {
  userPool: this.pool,
  clientId: 'xxxxx.apps.googleusercontent.com',
  clientSecretValue: cdk.SecretValue.secretsManager('google-client-secret'),
  scopes: ['openid', 'email', 'profile'],
  attributeMapping: {
    email: cognito.ProviderAttribute.GOOGLE_EMAIL,
  },
});

// Add Google to the client's supported providers
this.client.node.addDependency(/* google idp */);
```

In Google console add the callback URL: `https://demo-<acct>.auth.us-east-1.amazoncognito.com/oauth2/idpresponse`.

## Step 4: Lambda triggers (PreSignUp + PostConfirmation)
> **Why:** Triggers are how you customize Cognito without running your own auth server. Auto-confirm internal emails; drop a DynamoDB row when a user confirms.

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as ddb from 'aws-cdk-lib/aws-dynamodb';

const users = new ddb.Table(this, 'Users', {
  partitionKey: { name: 'userId', type: ddb.AttributeType.STRING },
  billingMode: ddb.BillingMode.PAY_PER_REQUEST,
});

const preSignUp = new lambda.Function(this, 'PreSignUp', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => {
      if (event.request.userAttributes.email.endsWith('@mycompany.com')) {
        event.response.autoConfirmUser = true;
        event.response.autoVerifyEmail = true;
      }
      return event;
    };
  `),
});

const postConfirm = new lambda.Function(this, 'PostConfirm', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  environment: { TABLE: users.tableName },
  code: lambda.Code.fromInline(`
    const { DynamoDBClient, PutItemCommand } = require('@aws-sdk/client-dynamodb');
    const ddb = new DynamoDBClient({});
    exports.handler = async (event) => {
      await ddb.send(new PutItemCommand({
        TableName: process.env.TABLE,
        Item: {
          userId: { S: event.userName },
          email:  { S: event.request.userAttributes.email },
          created:{ S: new Date().toISOString() },
        },
      }));
      return event;
    };
  `),
});
users.grantWriteData(postConfirm);

this.pool.addTrigger(cognito.UserPoolOperation.PRE_SIGN_UP, preSignUp);
this.pool.addTrigger(cognito.UserPoolOperation.POST_CONFIRMATION, postConfirm);
```

## Step 5: Identity Pool → IAM role per user
> **Why:** Map the JWT to AWS credentials. A `$aws:cognito-identity.amazonaws.com:sub` condition scopes S3 to `/users/<their-sub>/*` — every user gets a private prefix with zero backend code.

```typescript
import * as idp from 'aws-cdk-lib/aws-cognito-identitypool-alpha';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3 from 'aws-cdk-lib/aws-s3';

const bucket = new s3.Bucket(this, 'UserFiles', { removalPolicy: cdk.RemovalPolicy.DESTROY });

const idPool = new idp.IdentityPool(this, 'IdPool', {
  authenticationProviders: {
    userPools: [new idp.UserPoolAuthenticationProvider({
      userPool: this.pool,
      userPoolClient: this.client,
    })],
  },
});

idPool.authenticatedRole.addToPrincipalPolicy(new iam.PolicyStatement({
  actions: ['s3:GetObject', 's3:PutObject'],
  resources: [`${bucket.bucketArn}/users/\${cognito-identity.amazonaws.com:sub}/*`],
}));
```

## Step 6: API Gateway with JWT authorizer
> **Why:** Validate JWT at the gateway — cheap, fast, no Lambda needed to check tokens. Lambda only runs for authorized requests.

```typescript
import * as apigw from 'aws-cdk-lib/aws-apigatewayv2';
import * as apiauth from 'aws-cdk-lib/aws-apigatewayv2-authorizers';
import * as apiint from 'aws-cdk-lib/aws-apigatewayv2-integrations';

const meFn = new lambda.Function(this, 'MeFn', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => ({
      statusCode: 200,
      body: JSON.stringify(event.requestContext.authorizer.jwt.claims),
    });
  `),
});

const api = new apigw.HttpApi(this, 'Api');
api.addRoutes({
  path: '/me',
  methods: [apigw.HttpMethod.GET],
  integration: new apiint.HttpLambdaIntegration('Int', meFn),
  authorizer: new apiauth.HttpJwtAuthorizer('JwtAuth',
    `https://cognito-idp.${this.region}.amazonaws.com/${this.pool.userPoolId}`,
    { jwtAudience: [this.client.userPoolClientId] }),
});
```

Test:
```bash
curl https://$API_ID.execute-api.us-east-1.amazonaws.com/me
# {"message":"Unauthorized"} -- 401

curl -H "Authorization: Bearer $ID_TOKEN" https://$API_ID.execute-api.us-east-1.amazonaws.com/me
# {"sub":"...","email":"you@example.com","aud":"...","iss":"...","exp":...}
```

## Cleanup
```bash
cdk destroy
```

## Common Errors
- **`NotAuthorizedException: Incorrect username or password`** — app client has wrong auth flow (enable `ALLOW_USER_SRP_AUTH`).
- **Social login returns `redirect_mismatch`** — callback URL in Google console must be the `/oauth2/idpresponse` URL, not your app URL.
- **401 on API Gateway with a valid token** — audience mismatch: `aud` claim must equal the client ID, and issuer must be the exact User Pool URL.
- **Can't change `email` attribute later** — `mutable: false`. Decide up front.
- **Hosted UI custom domain fails** — requires ACM cert in `us-east-1` and DNS `A ALIAS` to the Cognito domain.
- **Lambda trigger errors block sign-up** — any exception from the trigger fails the user action. Log and swallow non-critical errors.
