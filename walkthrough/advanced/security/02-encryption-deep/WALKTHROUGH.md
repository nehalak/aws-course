# Walkthrough — 02 Encryption Deep

## About this service
**KMS** is the AWS key management service. Beyond the basics (`aws:kms` encryption), KMS has **key policies** (the root authorization), **grants** (temporary programmatic permissions), **multi-Region keys** (same key material in multiple regions), **custom key stores** (back KMS with your own HSM via CloudHSM or XKS), and **Nitro Enclaves** (isolated VM for processing plaintext secrets).

**Why it matters:** "Encrypted at rest" is table stakes. The interesting problems are: who can use the key? Under what conditions? What happens if KMS is breached (XKS)? How do you process secrets without ever writing plaintext to memory that a sidecar can read (Enclaves)?
**When to use:** Key policies + grants for every production workload; multi-Region keys for DR; CloudHSM for FIPS 140-2 L3 requirements or single-tenant compliance; XKS when your regulator requires the key material to physically leave AWS; Enclaves for PII processing, PCI DSS CDE, key signing.
**When NOT to use:** Don't reach for CloudHSM/XKS unless a compliance document forces you — they're expensive and operationally painful. Default to AWS-managed KMS keys or customer-managed KMS keys with good key policies.

## Estimated cost
- KMS customer-managed key: **$1/key/month** + **$0.03 per 10,000 requests**
- Multi-Region replica key: **$1/month per region**
- CloudHSM: **$1.45/hr per HSM** (~$1,058/month per HSM, 2 for HA = **~$2,116/month**)
- XKS: KMS price + your HSM/connector cost (external)
- Nitro Enclave: no extra charge, but requires `m5.xlarge` or larger metal/nitro instance (~$140/month for m5.xlarge on-demand)
- Total for this lesson (skipping CloudHSM provision): **~$5/month** in KMS; if you actually run CloudHSM for 2h = **~$3** one-time

---

## Step 1: Hardened key policy (VPC endpoint + MFA conditions)
> **Why:** The default KMS key policy is wide open to the account root. In production you scope down by adding conditions: only this service can use the key (`kms:ViaService`), only from this VPC endpoint (`aws:SourceVpce`), only MFA-authenticated calls (`aws:MultiFactorAuthPresent`). These are defense-in-depth layers over IAM.

```typescript
import * as kms from 'aws-cdk-lib/aws-kms';
import * as iam from 'aws-cdk-lib/aws-iam';

const key = new kms.Key(this, 'AppKey', {
  alias: 'alias/app-data',
  enableKeyRotation: true,
  policy: new iam.PolicyDocument({ statements: [
    new iam.PolicyStatement({
      sid: 'Root',
      principals: [new iam.AccountRootPrincipal()],
      actions: ['kms:*'],
      resources: ['*'],
    }),
    new iam.PolicyStatement({
      sid: 'UseViaS3Only',
      principals: [new iam.AnyPrincipal()],
      actions: ['kms:Encrypt', 'kms:Decrypt', 'kms:GenerateDataKey*'],
      resources: ['*'],
      conditions: {
        StringEquals: { 'kms:ViaService': `s3.${this.region}.amazonaws.com` },
        Bool: { 'aws:SecureTransport': 'true' },
      },
    }),
    new iam.PolicyStatement({
      sid: 'AdminRequiresMFA',
      principals: [new iam.ArnPrincipal(`arn:aws:iam::${this.account}:role/KmsAdmin`)],
      actions: ['kms:PutKeyPolicy', 'kms:ScheduleKeyDeletion', 'kms:DisableKey'],
      resources: ['*'],
      conditions: {
        Bool: { 'aws:MultiFactorAuthPresent': 'true' },
        NumericLessThan: { 'aws:MultiFactorAuthAge': '300' },
      },
    }),
  ] }),
});
```

Verify the ViaService restriction:
```bash
# Direct KMS call fails:
aws kms encrypt --key-id alias/app-data --plaintext SGVsbG8= 
# → AccessDenied: kms:ViaService condition not met

# Via S3 (it sets the ViaService header automatically) — works:
aws s3 cp hello.txt s3://bucket/ --sse aws:kms --sse-kms-key-id alias/app-data
```

## Step 2: Grants (short-lived delegated crypto)
> **Why:** Grants exist for workflows where you can't edit the key policy in time or the grantee is ephemeral. A Lambda warms up, asks for a grant to decrypt one file, uses it, and the grant is revoked. Grants have an operation allowlist and can carry a `GrantConstraints` encryption context.

```bash
GRANT=$(aws kms create-grant \
  --key-id alias/app-data \
  --grantee-principal arn:aws:iam::$ACCT:role/LambdaDecryptRole \
  --operations Decrypt \
  --constraints "EncryptionContextEquals={tenant=acme}" \
  --name decrypt-acme-only \
  --query 'GrantId' --output text)

# Lambda, assuming LambdaDecryptRole, can now decrypt only with context tenant=acme
aws kms decrypt --ciphertext-blob fileb://ct.bin \
  --encryption-context tenant=acme

# Revoke:
aws kms revoke-grant --key-id alias/app-data --grant-id $GRANT
```

After revoke, the same decrypt call returns `AccessDenied`.

## Step 3: Multi-Region keys (encrypt in us-east-1, decrypt in eu-west-1)
> **Why:** Cross-region DR is impossible with normal KMS keys — ciphertext encrypted with `key-us-east-1` can't be decrypted in eu-west-1. Multi-Region keys share key material (via secure replication) so a replica in eu-west-1 decrypts the same ciphertext. Critical for S3 cross-region replication, DynamoDB global tables, and warm-standby DR.

```typescript
// Primary in us-east-1
const primary = new kms.CfnKey(this, 'PrimaryKey', {
  multiRegion: true,
  enableKeyRotation: true,
  keyPolicy: { /* same as step 1 */ },
});

// Replica in eu-west-1 (different stack, different region)
new kms.CfnReplicaKey(euStack, 'Replica', {
  primaryKeyArn: primary.attrArn,
  keyPolicy: { /* same as primary */ },
});
```

Test:
```bash
# Encrypt in us-east-1
CT=$(aws --region us-east-1 kms encrypt \
  --key-id mrk-abc123 --plaintext "top secret" \
  --query CiphertextBlob --output text)

# Decrypt in eu-west-1 using the replica — works because key material is the same
aws --region eu-west-1 kms decrypt \
  --ciphertext-blob fileb://<(echo $CT | base64 -d) \
  --query Plaintext --output text | base64 -d
# → top secret
```

Note: IAM permissions are per-region. Grant `kms:Decrypt` on the replica ARN in the eu-west-1 account/role separately.

## Step 4: CloudHSM custom key store — design walkthrough
> **Why:** For FIPS 140-2 Level 3, single-tenant HSM, or regulators that demand "AWS must not have access to the key material," you run CloudHSM (dedicated HSM appliances in your VPC) and point KMS at it as a **custom key store**. KMS still does the API surface — your apps call KMS — but under the hood the crypto happens on your HSM. This is expensive and operationally heavy; walk through the design before spending money.

**Architecture:**
```
App  →  kms:Encrypt (alias/app-data)  →  KMS API
                                              │
                                              ▼ (via TLS to your HSM cluster)
                                       CloudHSM cluster (2+ HSMs in a VPC,
                                       multi-AZ, owned by your account)
                                              │
                                              ▼
                                       Key material stored on HSM hardware
```

**Provisioning steps (design; don't run unless you're ready for ~$3/hr):**
1. `aws cloudhsmv2 create-cluster --hsm-type hsm1.medium --subnet-ids subnet-a subnet-b`
2. Initialize the cluster: download the CSR, sign it with your own CA or self-sign, upload back.
3. `aws cloudhsmv2 create-hsm` at least twice (two HSMs for HA; losing a single HSM in a 1-node cluster destroys all keys).
4. Install the client on an EC2 in the same VPC, activate the cluster, set the Crypto Officer (CO) password.
5. Create a Crypto User (CU) — this is the identity KMS will use.
6. `aws kms create-custom-key-store --custom-key-store-name corp-hsm --cloud-hsm-cluster-id cluster-xyz --key-store-password <CU password> --trust-anchor-certificate file://customerCA.crt`
7. `aws kms connect-custom-key-store --custom-key-store-id cks-abc`
8. `aws kms create-key --custom-key-store-id cks-abc --origin AWS_CLOUDHSM` → now you have a KMS key backed by your HSM.

**Operational gotchas:**
- If you lose the CO/CU credentials, AWS cannot recover them. Back up in a corporate secrets vault.
- Scale to **≥2 HSMs across AZs** always. A single HSM failure wipes the cluster.
- Patching is your responsibility — you schedule the FW updates.
- Billing is per-HSM-hour the instant the HSM exists. `delete-hsm` before sleep/stop.

## Step 5: XKS (External Key Store) — paper design
> **Why:** XKS is the nuclear option: the key material lives in *your* HSM, *outside* AWS, and KMS calls out over HTTPS for every crypto operation. Required by some jurisdictions that prohibit cloud-resident keys. The trade-off is latency (~30ms per op vs ~1ms for native KMS) and you own a 99.999% SLA HSM + proxy.

**Components:**
```
KMS  ──HTTPS (mTLS)──>  XKS Proxy  ──PKCS#11──>  External HSM (Thales, Entrust, Futurex)
                        (you run)               (you own; on-prem or co-lo)
```

**Requirements you sign up for:**
- Highly available proxy: 99.999% over 7-day rolling window (~25 seconds/month downtime).
- Round-trip latency < 250ms p99 (KMS retries beyond this).
- Network path: either direct public internet with mTLS, or VPC endpoint service (PrivateLink) from the KMS side into your VPC, which then tunnels to on-prem via Direct Connect.
- Proxy implements the AWS KMS XKS proxy API spec (it's a documented REST spec; several vendors ship certified proxies).

**Provisioning sketch (don't deploy without a real HSM):**
1. Deploy vendor proxy (e.g., Thales CipherTrust XKS) in a VPC, in front of on-prem HSM.
2. Create a VPC endpoint service for the proxy.
3. `aws kms create-custom-key-store --custom-key-store-type EXTERNAL_KEY_STORE --xks-proxy-uri-endpoint https://xks.corp.example.com --xks-proxy-connectivity VPC_ENDPOINT_SERVICE --xks-proxy-vpc-endpoint-service-name com.amazonaws.vpce...`
4. `aws kms create-key --origin EXTERNAL_KEY_STORE --xks-key-id corp-key-42 --custom-key-store-id cks-xks`

**What breaks:** if the proxy is down, every `aws:kms` operation fails. EBS won't mount, S3 GETs fail, RDS connections fail. Keep your XKS proxy more reliable than AWS itself.

## Step 6: Nitro Enclave — confidential processing of a secret
> **Why:** Even with KMS, your Lambda/EC2 process has plaintext in RAM after `Decrypt`. A compromised kernel, debugger, or sidecar container can read it. Nitro Enclaves give you an isolated VM (no disk, no network, no SSH — only a `vsock` channel to the parent) attested by AWS so KMS only releases the plaintext to that specific enclave.

Launch an enclave-enabled EC2:
```typescript
const instance = new ec2.Instance(this, 'EnclaveHost', {
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.M5, ec2.InstanceSize.XLARGE),
  machineImage: ec2.MachineImage.latestAmazonLinux2023(),
  vpc,
  enclaveEnabled: true,
  role: enclaveRole, // see below
});
```

Key policy that restricts decrypt to the enclave's image hash (PCR0):
```json
{
  "Sid": "OnlyFromThisEnclave",
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::123456789012:role/EnclaveRole" },
  "Action": "kms:Decrypt",
  "Resource": "*",
  "Condition": {
    "StringEqualsIgnoreCase": {
      "kms:RecipientAttestation:ImageSha384":
        "8c4e2b5e... (PCR0 of your signed enclave image)"
    }
  }
}
```

Inside the enclave, `kmstool-enclave-cli decrypt --ciphertext ...` uses the vsock-proxied KMS call plus the attestation doc. Plaintext never appears in the parent instance.

## Cleanup
```bash
# KMS keys schedule for deletion (7-30 day window; use 7 for dev)
aws kms schedule-key-deletion --key-id <id> --pending-window-in-days 7
cdk destroy
# If CloudHSM was provisioned:
aws cloudhsmv2 delete-hsm --cluster-id cluster-xyz --hsm-id hsm-abc
aws cloudhsmv2 delete-cluster --cluster-id cluster-xyz
```

## Common Errors
- **`InvalidCiphertextException` across regions** → you used a single-region key; ciphertext only works in its origin region. Use MRK.
- **Key policy locks you out** → you removed the `AccountRoot` statement. Recovery: AWS Support ticket (they can help restore root access under narrow conditions, not always).
- **CloudHSM cluster shows `UNINITIALIZED`** → you haven't signed and uploaded the CSR yet. A cluster with HSMs but uninitialized still bills.
- **Enclave cannot reach KMS** → NitroEnclave only has vsock; you need the `kmstool-enclave` daemon on the parent forwarding to KMS. Also the parent instance role (not the enclave itself) calls KMS.
- **Grant revoke is eventually consistent** → allow ~5 min for a revoke to fully propagate.
- **Encryption context mismatch** → you encrypted with `{tenant:acme}` and decrypted with `{Tenant:acme}`. Context is case-sensitive.
