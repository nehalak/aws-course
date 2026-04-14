# Walkthrough — 03 Certificate Manager (ACM)

## About this service
**ACM** issues and manages TLS/SSL certificates. **Public certs** are free and auto-renew forever — the catch is they only attach to AWS-integrated services (ALB, NLB TLS passthrough, CloudFront, API Gateway, App Runner). You can't export the private key. **ACM Private CA (PCA)** issues internal certs for mTLS, service-to-service auth, IoT. Validation is DNS (recommended — auto-renews if the CNAME stays put) or email. CloudFront has a quirk: its cert must live in `us-east-1` regardless of where the distribution serves from.

**Why it matters:** HTTPS is non-negotiable. ACM makes "free TLS that auto-renews" the default. Letting certs expire = full outage; ACM removes that class of incident.

**When to use:** any HTTPS endpoint on ALB/CloudFront/API GW; internal mTLS between services.
**When NOT to use:** cert for a non-AWS server (you can't export ACM public certs — use Let's Encrypt). Private CA for anything where Let's Encrypt + public DNS works: PCA is expensive.

## Estimated cost
- ACM public certs: **$0.00** (free, forever)
- ACM Private CA (general-purpose): **$400/month** per CA + $0.75 per cert issued
- ACM Private CA (short-lived, ≤7-day certs): **$50/month** per CA + $0.058 per cert
- ALB hours: **$16/month** + LCU usage
- Total for this lesson (public only): **~$16/month** (ALB)
- Total with Private CA short-lived: **~$66/month** — **delete the CA after the exercise**

---

## Step 1: Request a public cert (DNS-validated)
> **Why:** DNS validation is the only option worth using. Email validation breaks renewal the moment the inbox owner leaves. With Route 53 in the same account, CDK adds the CNAME for you and validation is transparent.

`lib/acm-stack.ts`:
```typescript
import * as cdk from 'aws-cdk-lib';
import * as acm from 'aws-cdk-lib/aws-certificatemanager';
import * as route53 from 'aws-cdk-lib/aws-route53';
import { Construct } from 'constructs';

export class AcmStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const zone = route53.HostedZone.fromLookup(this, 'Zone', {
      domainName: 'example.com',
    });

    const cert = new acm.Certificate(this, 'SiteCert', {
      domainName: 'app.example.com',
      subjectAlternativeNames: ['*.app.example.com'],
      validation: acm.CertificateValidation.fromDns(zone),
    });

    new cdk.CfnOutput(this, 'CertArn', { value: cert.certificateArn });
  }
}
```

Deploy; CDK adds the validation CNAMEs and waits for `ISSUED`:
```bash
cdk deploy
aws acm describe-certificate --certificate-arn $CERT_ARN \
  --query "Certificate.Status"
# "ISSUED"
```

## Step 2: Attach to an ALB + redirect HTTP→HTTPS
> **Why:** HTTPS listener needs a cert. The redirect listener ensures no insecure traffic ever reaches your app — a 301 to the HTTPS port before the request ever hits a target.

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as elb from 'aws-cdk-lib/aws-elasticloadbalancingv2';

const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2 });
const alb = new elb.ApplicationLoadBalancer(this, 'Alb', {
  vpc,
  internetFacing: true,
});

// HTTPS listener with the ACM cert
const httpsListener = alb.addListener('Https', {
  port: 443,
  certificates: [elb.ListenerCertificate.fromCertificateManager(cert)],
  sslPolicy: elb.SslPolicy.RECOMMENDED_TLS,
  defaultAction: elb.ListenerAction.fixedResponse(200, {
    contentType: 'text/plain',
    messageBody: 'hello TLS',
  }),
});

// HTTP redirect to HTTPS
alb.addListener('HttpRedirect', {
  port: 80,
  defaultAction: elb.ListenerAction.redirect({
    protocol: 'HTTPS',
    port: '443',
    permanent: true,
  }),
});

new route53.ARecord(this, 'Alias', {
  zone,
  recordName: 'app',
  target: route53.RecordTarget.fromAlias(new targets.LoadBalancerTarget(alb)),
});
```

Verify:
```bash
curl -I http://app.example.com
# HTTP/1.1 301 Moved Permanently
# Location: https://app.example.com:443/

curl https://app.example.com
# hello TLS

openssl s_client -connect app.example.com:443 -servername app.example.com < /dev/null 2>&1 \
  | grep "subject="
# subject=CN = app.example.com
```

## Step 3: CloudFront cert — must be us-east-1
> **Why:** CloudFront is global but reads certs from a single region: `us-east-1`. Deploying from Frankfurt and forgetting this is the #1 ACM gotcha. CDK's `DnsValidatedCertificate` (or a cross-region stack) solves it.

```typescript
// In a stack pinned to us-east-1, OR use cross-region helper:
const cfCert = new acm.DnsValidatedCertificate(this, 'CfCert', {
  domainName: 'cdn.example.com',
  hostedZone: zone,
  region: 'us-east-1',  // cross-region issuance
});

new cloudfront.Distribution(this, 'Dist', {
  defaultBehavior: { origin: new origins.HttpOrigin('backend.example.com') },
  domainNames: ['cdn.example.com'],
  certificate: cfCert,
});
```

## Step 4: Private CA (short-lived mode) for mTLS
> **Why:** Public CAs don't issue certs for `internal.mycompany.local`. Private CA does. Use **short-lived mode** ($50/mo vs $400/mo) for dev. Short-lived certs (≤7 days) remove most revocation headaches too.

```typescript
import * as acmpca from 'aws-cdk-lib/aws-acmpca';

const pca = new acmpca.CfnCertificateAuthority(this, 'InternalCA', {
  type: 'ROOT',
  usageMode: 'SHORT_LIVED_CERTIFICATE',  // $50/mo tier
  keyAlgorithm: 'RSA_2048',
  signingAlgorithm: 'SHA256WITHRSA',
  subject: {
    country: 'US',
    organization: 'MyCompany',
    commonName: 'MyCompany Internal Root',
  },
});
```

After deploy, install the root cert on itself (bootstrap):
```bash
aws acm-pca issue-certificate \
  --certificate-authority-arn $PCA_ARN \
  --csr fileb://<(aws acm-pca get-certificate-authority-csr --certificate-authority-arn $PCA_ARN --output text) \
  --signing-algorithm SHA256WITHRSA \
  --template-arn arn:aws:acm-pca:::template/RootCACertificate/V1 \
  --validity Value=3650,Type=DAYS
# Then aws acm-pca import-certificate-authority-certificate ...
```

## Step 5: mTLS on ALB with client cert
> **Why:** mTLS proves both sides of the connection. Classic for internal B2B APIs and partner integrations. ALB now supports it natively with a trust store of CA bundles.

```bash
# Create a trust store backed by S3 CA bundle
aws s3 cp ca-bundle.pem s3://my-trust-bucket/ca.pem

aws elbv2 create-trust-store \
  --name internal-ca-trust \
  --ca-certificates-bundle-s3-bucket my-trust-bucket \
  --ca-certificates-bundle-s3-key ca.pem

# Modify HTTPS listener to require mTLS
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --mutual-authentication Mode=verify,TrustStoreArn=$TRUST_STORE_ARN
```

Test:
```bash
curl https://app.example.com          # HTTP 495 (cert required)
curl --cert client.pem --key client.key https://app.example.com  # works
```

## Cleanup
```bash
cdk destroy
# Private CA: must be disabled + scheduled for deletion (7-30 day window)
aws acm-pca update-certificate-authority --certificate-authority-arn $PCA_ARN --status DISABLED
aws acm-pca delete-certificate-authority --certificate-authority-arn $PCA_ARN \
  --permanent-deletion-time-in-days 7
```

## Common Errors
- **Cert stuck in `PENDING_VALIDATION`** → DNS CNAME missing or in the wrong zone. Check `aws acm describe-certificate` for the expected record.
- **CloudFront: "certificate must be in us-east-1"** → cert was issued in another region. Reissue in us-east-1.
- **ALB ignores cert update** → listeners cache; force a deploy or re-attach via console.
- **Private CA $400 bill** → you created GENERAL_PURPOSE instead of SHORT_LIVED_CERTIFICATE. Delete and recreate.
- **mTLS 495 with a valid cert** → cert was signed by a CA not in the trust store bundle, or bundle uses wrong PEM format.
