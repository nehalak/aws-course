# Walkthrough — 04 Secrets Manager vs SSM Parameter Store

## About these services
Both store sensitive data. Different trade-offs:

**Secrets Manager**: built for secrets (DB passwords, API keys). Supports **automatic rotation** (Lambda rotator), cross-region replication, JSON secret values, generated passwords. Costs money.

**SSM Parameter Store**: started as config store, can do secrets via **SecureString** type. No rotation. Simpler. **Standard tier free**.

**Why it matters:** mixing these up creates drift. "Where's the DB password?" with no answer = outage.

**When to use Secrets Manager:** DB passwords, API keys that need rotation, values shared cross-region, anything where audit + rotation is required.
**When to use SSM Parameter Store:** app config (URLs, feature flags, ARNs), values >4KB (use Advanced tier), cost-sensitive setups with many simple configs.
**When NOT to use either:** large binary data (use S3 with KMS), per-request values (use STS/temporary credentials).

## Estimated cost
- **Secrets Manager: $0.40/secret/month** + $0.05 per 10k API calls
- **SSM Standard: free** (up to 10k parameters, 4KB each)
- **SSM Advanced: $0.05/parameter/month** (up to 8KB, policy features)
- Total for this lesson: **~$0.40/month** (one secret)

---

## Step 1: SSM parameter
> **Why:** Parameter Store Standard is free and fine for non-sensitive config. Reference in CDK stacks without hardcoding values. Perfect for env-specific URLs, ARNs, feature flags.

```typescript
import * as ssm from 'aws-cdk-lib/aws-ssm';

new ssm.StringParameter(this, 'ApiUrl', {
  parameterName: '/myapp/dev/api-url',
  stringValue: 'https://api.dev.example.com',
});
```

```bash
aws ssm get-parameter --name /myapp/dev/api-url
```

## Step 2: SecureString
> **Why:** SecureString = KMS-encrypted value. Distinction matters: `get-parameter` returns ciphertext unless you pass `--with-decryption`. Prevents accidentally logging raw secrets in CLI output.

```bash
aws ssm put-parameter --name /myapp/dev/db-password \
  --value 'SuperSecret!' --type SecureString --overwrite

aws ssm get-parameter --name /myapp/dev/db-password              # ciphertext
aws ssm get-parameter --name /myapp/dev/db-password --with-decryption   # plaintext
```

## Step 3: Secrets Manager
> **Why:** Generated passwords are the killer feature — no developer sees the actual password. Combined with Secret references in CDK, passwords never hit source control or terminal history.

```typescript
import * as sm from 'aws-cdk-lib/aws-secretsmanager';

const secret = new sm.Secret(this, 'DbPassword', {
  secretName: 'myapp/dev/db-password-sm',
  generateSecretString: {
    passwordLength: 32,
    excludeCharacters: '/@" \'',
  },
});
```

## Step 4: Rotation (preview)
> **Why:** Rotation = the single biggest win of Secrets Manager over SSM. Attaching a Lambda rotator + enabling 30-day rotation means even if a password leaks, it's stale within 30 days.

Wired fully in `data/01-rds-basics` lesson.

## Step 5: Cost compare
> **Why:** The 10x cost difference at scale matters. Making the call case-by-case — "does this need rotation?" — is correct engineering.

```markdown
# decision.md
50 app configs × $0 (SSM Standard) = $0/mo
10 DB passwords × $0.40 (Secrets Mgr) = $4/mo
```

## Step 6: Reference in Lambda
> **Why:** Passing the ARN (not value) to environment variables means secret values never appear in CFN templates, CloudTrail logs, or `aws lambda get-function` output. Resolve at runtime.

```typescript
const fn = new NodejsFunction(this, 'App', {
  environment: { DB_SECRET_ARN: secret.secretArn },
});
secret.grantRead(fn);
```

In handler:
```javascript
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
const sm = new SecretsManagerClient();
const r = await sm.send(new GetSecretValueCommand({ SecretId: process.env.DB_SECRET_ARN }));
const secret = JSON.parse(r.SecretString);
```

## Cleanup
```bash
cdk destroy
aws secretsmanager delete-secret \
  --secret-id myapp/dev/db-password-sm \
  --force-delete-without-recovery
aws ssm delete-parameter --name /myapp/dev/db-password
aws ssm delete-parameter --name /myapp/dev/api-url
```

## Common Errors
- **`secret_value: Unable to fetch`** — role doesn't have `secretsmanager:GetSecretValue`.
- **Secret stuck in 7-day recovery** — use `--force-delete-without-recovery` to bypass.
