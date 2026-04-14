# Walkthrough — 01 Zero Trust Patterns

## About this service
**Zero Trust** means: never trust the network, verify every request by identity, device, and context. AWS offers two managed building blocks: **Verified Access (AVA)** replaces VPN for internal corporate apps (login at each request, evaluated against Cedar policies); **Verified Permissions (AVP)** is a managed Cedar policy engine for application-level authorization (e.g., "can this user read this document?").

**Why it matters:** Perimeter security (VPN, corporate IP allowlists) fails when laptops roam, contractors join, and attackers phish a single credential. Zero Trust moves the check to every request: identity + device posture + request attributes re-evaluated every time.
**When to use:** Internal web apps that previously sat behind a VPN, SaaS/multi-tenant apps needing fine-grained ABAC, document/record-level permissions.
**When NOT to use:** Public marketing sites (no identity), simple role-based APIs (Cognito + API Gateway authorizer is enough), latency-critical trading paths (Cedar eval adds ms).

## Estimated cost
- Verified Access: **$0.27/app/hr** (~$197/month per app) + **$0.02/GB** data processed
- Verified Permissions: **$150 per million authorization requests** (first 40M/mo @ $150, then tiered)
- Cognito user pool: first 50k MAU free, then $0.0055/MAU
- ALB (internal, used by AVA endpoint): ~$16/month + LCU
- Total for this lesson (1 app, dev traffic): **~$215/month** — destroy same day if just learning

---

## Step 1: Deploy a private web app behind Verified Access
> **Why:** AVA is the Zero Trust replacement for Client VPN. It front-ends an internal ALB, forces IdP login on every session, and evaluates a Cedar policy per request. You never expose the app to the public internet — AVA itself is the only public endpoint and it enforces auth before forwarding.

`lib/zero-trust-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as elb from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import * as ava from 'aws-cdk-lib/aws-ec2';
import { CfnVerifiedAccessInstance, CfnVerifiedAccessTrustProvider,
         CfnVerifiedAccessGroup, CfnVerifiedAccessEndpoint } from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export class ZeroTrustStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });

    // Internal ALB fronting a private app (e.g., internal dashboard)
    const alb = new elb.ApplicationLoadBalancer(this, 'InternalAlb', {
      vpc, internetFacing: false,
    });
    const listener = alb.addListener('Http', { port: 80, open: false });
    listener.addAction('Default', {
      action: elb.ListenerAction.fixedResponse(200, {
        contentType: 'text/html',
        messageBody: '<h1>Internal App — you are authenticated</h1>',
      }),
    });

    // Cognito user pool as identity source
    const userPool = new cognito.UserPool(this, 'Pool', {
      selfSignUpEnabled: false,
      signInAliases: { email: true },
      customAttributes: { department: new cognito.StringAttribute({ mutable: true }) },
    });
    const client = userPool.addClient('AvaClient', {
      oAuth: { flows: { authorizationCodeGrant: true },
               callbackUrls: ['https://placeholder.example.com/oauth/callback'] },
    });
    userPool.addDomain('Domain', { cognitoDomain: { domainPrefix: `ava-${this.account}` } });

    // Trust provider: Cognito OIDC
    const trustProvider = new CfnVerifiedAccessTrustProvider(this, 'TrustProvider', {
      trustProviderType: 'user',
      userTrustProviderType: 'oidc',
      policyReferenceName: 'cognito',
      oidcOptions: {
        issuer: `https://cognito-idp.${this.region}.amazonaws.com/${userPool.userPoolId}`,
        authorizationEndpoint: `https://ava-${this.account}.auth.${this.region}.amazoncognito.com/oauth2/authorize`,
        tokenEndpoint: `https://ava-${this.account}.auth.${this.region}.amazoncognito.com/oauth2/token`,
        userInfoEndpoint: `https://ava-${this.account}.auth.${this.region}.amazoncognito.com/oauth2/userInfo`,
        clientId: client.userPoolClientId,
        clientSecret: client.userPoolClientSecret.unsafeUnwrap(),
        scope: 'openid email profile',
      },
    });

    const avaInstance = new CfnVerifiedAccessInstance(this, 'AvaInstance', {
      description: 'Zero trust instance',
    });

    new cdk.CfnResource(this, 'AttachTrust', {
      type: 'AWS::EC2::VerifiedAccessTrustProviderAttachment',
      properties: {
        VerifiedAccessInstanceId: avaInstance.ref,
        VerifiedAccessTrustProviderId: trustProvider.ref,
      },
    });

    const group = new CfnVerifiedAccessGroup(this, 'AvaGroup', {
      verifiedAccessInstanceId: avaInstance.ref,
      policyDocument: `permit(principal, action, resource) when {
        context.cognito.email_verified == true
      };`,
    });

    // Endpoint = the public URL AVA gives you
    new CfnVerifiedAccessEndpoint(this, 'Endpoint', {
      applicationDomain: 'app.internal.example.com',
      attachmentType: 'vpc',
      domainCertificateArn: 'arn:aws:acm:us-east-1:123456789012:certificate/xxx', // replace
      endpointDomainPrefix: 'app',
      endpointType: 'load-balancer',
      loadBalancerOptions: {
        loadBalancerArn: alb.loadBalancerArn,
        port: 80, protocol: 'http',
        subnetIds: vpc.privateSubnets.map(s => s.subnetId),
      },
      securityGroupIds: [alb.connections.securityGroups[0].securityGroupId],
      verifiedAccessGroupId: group.ref,
    });
  }
}
```

Deploy:
```bash
cdk deploy ZeroTrustStack
```

Create a Cognito user:
```bash
aws cognito-idp admin-create-user --user-pool-id <pool> \
  --username alice@example.com \
  --user-attributes Name=email,Value=alice@example.com Name=custom:department,Value=eng \
  --temporary-password 'TempPass!23'
```

Browse to the AVA endpoint URL → redirected to Cognito → login → see the app.

## Step 2: Tighten with attribute + device Cedar policy
> **Why:** A bare "is logged in" check is barely Zero Trust. Real ZT uses attributes from the IdP (department, role) AND device posture (managed device, disk encrypted, OS patched). Cedar policies are evaluated per request — rotate the policy and access changes on the next click.

Update the group policy:
```cedar
permit(principal, action, resource)
when {
  context.cognito.email_verified == true &&
  context.cognito["custom:department"] == "eng" &&
  context.jamf.risk_score <= 25 &&
  context.jamf.disk_encrypted == true
};
```

Apply:
```bash
aws ec2 modify-verified-access-group \
  --verified-access-group-id <group-id> \
  --policy-document file://group-policy.cedar
```

Attach Jamf (or CrowdStrike) trust provider of type `device`. For learning, you can mock the context by using a second OIDC trust provider and asserting the attributes yourself.

Test: sign in as a user in `sales` department → 403. Add `department=eng` → works.

## Step 3: Verified Permissions policy store for a doc app
> **Why:** AVA gates access to an app; AVP gates actions inside an app (document-level). Instead of hand-rolling `if (doc.owner === user || doc.sharedWith.includes(user))` across your codebase, you define policies in Cedar, call `IsAuthorized`, and move policy changes out of deploys.

```typescript
import { CfnPolicyStore, CfnPolicy, CfnIdentitySource } from 'aws-cdk-lib/aws-verifiedpermissions';

const store = new CfnPolicyStore(this, 'DocsStore', {
  validationSettings: { mode: 'STRICT' },
  schema: {
    cedarJson: JSON.stringify({
      DocsApp: {
        entityTypes: {
          User:     { shape: { type: 'Record', attributes: { department: { type: 'String' } } } },
          Document: { shape: { type: 'Record', attributes: {
                        owner: { type: 'Entity', name: 'User' },
                        sharedWith: { type: 'Set', element: { type: 'Entity', name: 'User' } },
                      } } },
        },
        actions: {
          ReadDoc:   { appliesTo: { principalTypes: ['User'], resourceTypes: ['Document'] } },
          DeleteDoc: { appliesTo: { principalTypes: ['User'], resourceTypes: ['Document'] } },
        },
      },
    }),
  },
});

new CfnPolicy(this, 'OwnerCanRead', {
  policyStoreId: store.attrPolicyStoreId,
  definition: { static: { statement: `
    permit(principal, action == DocsApp::Action::"ReadDoc", resource)
    when { resource.owner == principal };`,
  } },
});

new CfnPolicy(this, 'SharedCanRead', {
  policyStoreId: store.attrPolicyStoreId,
  definition: { static: { statement: `
    permit(principal, action == DocsApp::Action::"ReadDoc", resource)
    when { resource.sharedWith.contains(principal) };`,
  } },
});

// Identity source = Cognito pool → principals come from JWTs
new CfnIdentitySource(this, 'Idp', {
  policyStoreId: store.attrPolicyStoreId,
  configuration: { cognitoUserPoolConfiguration: {
    userPoolArn: userPool.userPoolArn,
    clientIds: [client.userPoolClientId],
  } },
  principalEntityType: 'DocsApp::User',
});
```

## Step 4: Authorize an API call with AVP
> **Why:** The wiring from API → AVP is where ZT meets the application. Every request carries a JWT; your API hands JWT + action + resource to AVP; AVP says allow/deny. The app never implements ownership logic.

```typescript
// Lambda authorizer or in-handler check
import { VerifiedPermissionsClient, IsAuthorizedWithTokenCommand }
  from '@aws-sdk/client-verifiedpermissions';

const client = new VerifiedPermissionsClient({});

export const handler = async (evt: any) => {
  const res = await client.send(new IsAuthorizedWithTokenCommand({
    policyStoreId: process.env.POLICY_STORE_ID,
    identityToken: evt.headers.authorization.replace('Bearer ', ''),
    action: { actionType: 'DocsApp::Action', actionId: 'ReadDoc' },
    resource: { entityType: 'DocsApp::Document', entityId: evt.pathParameters.docId },
    entities: { entityList: [
      { identifier: { entityType: 'DocsApp::Document', entityId: evt.pathParameters.docId },
        attributes: {
          owner: { entityIdentifier: { entityType: 'DocsApp::User', entityId: 'alice' } },
          sharedWith: { set: [] },
        } },
    ] },
  }));
  if (res.decision !== 'ALLOW') return { statusCode: 403, body: 'Forbidden' };
  return { statusCode: 200, body: 'doc contents' };
};
```

## Step 5: Cedar playground iteration
> **Why:** Cedar has its own evaluation semantics — unknown entities become `false`, `in` traverses hierarchies, `has` checks attribute existence. The online playground (cedarpolicy.com/playground) lets you pin a policy + entities + request and watch allow/deny without deploying. Always iterate there before shipping.

Paste your policy, some entities (`User::"alice"` with `department="eng"`; a `Document::"42"` owned by alice), and a request `alice → ReadDoc → 42`. Confirm `ALLOW`. Flip `owner` to another user, no sharing → `DENY`.

## Cleanup
```bash
cdk destroy ZeroTrustStack
# Verified Access endpoints bill per hour — destroy when done.
# Verified Permissions is request-priced; idle policy stores are free.
```

## Common Errors
- **AVA endpoint returns 503** → ALB target group unhealthy, or security group doesn't allow AVA ENI traffic. AVA injects ENIs into your private subnets; the ALB SG must allow from those.
- **Cedar `unexpected token`** → `permit` and `forbid` are statements; each must end with `;`. Attributes with special chars need `["custom:department"]` bracket syntax.
- **AVP returns DENY but policy looks right** → you forgot to pass the resource's attributes in `entities`. Cedar can't evaluate `resource.owner` if you never sent it.
- **Cognito custom attribute missing** → custom attributes need `custom:` prefix in tokens AND must be added to the app client's read scope.
- **403 after long session** → OIDC token expired; AVA re-checks on every request, browser needs to refresh the token.
