# AWS Mastery Course (CDK — TypeScript)

A hands-on curriculum. Every lesson is a folder with a README containing:
- **Concept** — what and why
- **Prereqs** — lessons/tools required
- **Exercises** — practical tasks to do with CDK + AWS Console
- **Verification** — how to prove it works
- **Gotchas** — real-world pitfalls
- **Cleanup** — `cdk destroy` + manual steps

## Setup (do this once)

```bash
npm install -g aws-cdk typescript
aws configure           # create an IAM user "cdk-dev" with AdministratorAccess (learning only)
cdk bootstrap aws://ACCOUNT_ID/us-east-1
```

For each lesson:
```bash
mkdir <lesson>/cdk && cd <lesson>/cdk
cdk init app --language=typescript
```

## Structure

- `basic/` — foundations, single-service mastery
- `intermediate/` — production patterns, integrations
- `advanced/` — multi-region, deep internals, capstones

## Rules
1. Read AWS docs for the service before writing CDK.
2. Deploy → break it → observe → fix → destroy.
3. Track spend daily in Cost Explorer.
4. Never commit credentials; use `~/.aws/credentials`.

## Study Order

### 1. Basic

**1.1 core-concepts**
- 1.1.1 `basic/core-concepts/01-aws-account-setup`
- 1.1.2 `basic/core-concepts/02-regions-azs-edge`
- 1.1.3 `basic/core-concepts/03-cdk-bootstrap`
- 1.1.4 `basic/core-concepts/04-cloudformation-primer`

**1.2 infra**
- 1.2.1 `basic/infra/01-vpc-basics`
- 1.2.2 `basic/infra/02-ec2-fundamentals`
- 1.2.3 `basic/infra/03-s3-basics`
- 1.2.4 `basic/infra/04-ebs-efs`
- 1.2.5 `basic/infra/05-route53-basics`

**1.3 compute**
- 1.3.1 `basic/compute/01-lambda-hello`
- 1.3.2 `basic/compute/02-ecs-fargate-intro`

**1.4 data**
- 1.4.1 `basic/data/01-rds-basics`
- 1.4.2 `basic/data/02-dynamodb-basics`

**1.5 security**
- 1.5.1 `basic/security/01-iam-deep`
- 1.5.2 `basic/security/02-security-groups-nacls`
- 1.5.3 `basic/security/03-kms-basics`
- 1.5.4 `basic/security/04-secrets-manager-vs-ssm`

**1.6 observability**
- 1.6.1 `basic/observability/01-cloudwatch-logs-metrics`
- 1.6.2 `basic/observability/02-cloudtrail`

---

### 2. Intermediate

**2.1 infra**
- 2.1.1 `intermediate/infra/01-vpc-advanced`
- 2.1.2 `intermediate/infra/02-multi-az-patterns`
- 2.1.3 `intermediate/infra/03-autoscaling`
- 2.1.4 `intermediate/infra/04-cloudfront-waf`
- 2.1.5 `intermediate/infra/05-api-gateway`
- 2.1.6 `intermediate/infra/06-eventbridge-sns-sqs`

**2.2 compute**
- 2.2.1 `intermediate/compute/01-lambda-advanced`
- 2.2.2 `intermediate/compute/02-ecs-patterns`
- 2.2.3 `intermediate/compute/03-eks-intro`
- 2.2.4 `intermediate/compute/04-step-functions`
- 2.2.5 `intermediate/compute/05-batch`

**2.3 data**
- 2.3.1 `intermediate/data/01-dynamodb-advanced`
- 2.3.2 `intermediate/data/02-rds-advanced`
- 2.3.3 `intermediate/data/03-elasticache`
- 2.3.4 `intermediate/data/04-s3-advanced`
- 2.3.5 `intermediate/data/05-kinesis-msk`
- 2.3.6 `intermediate/data/06-opensearch`

**2.4 security**
- 2.4.1 `intermediate/security/01-iam-advanced`
- 2.4.2 `intermediate/security/02-secrets-rotation`
- 2.4.3 `intermediate/security/03-certificate-manager`
- 2.4.4 `intermediate/security/04-cognito`
- 2.4.5 `intermediate/security/05-guardduty-securityhub`
- 2.4.6 `intermediate/security/06-macie-inspector`

**2.5 devops**
- 2.5.1 `intermediate/devops/01-cdk-patterns`
- 2.5.2 `intermediate/devops/02-cdk-pipelines`
- 2.5.3 `intermediate/devops/03-codebuild-codedeploy`
- 2.5.4 `intermediate/devops/04-ecr`

**2.6 observability**
- 2.6.1 `intermediate/observability/01-xray`
- 2.6.2 `intermediate/observability/02-cloudwatch-advanced`
- 2.6.3 `intermediate/observability/03-aws-config`

---

### 3. Advanced

**3.1 infra**
- 3.1.1 `advanced/infra/01-multi-account-landing-zone`
- 3.1.2 `advanced/infra/02-multi-region-architectures`
- 3.1.3 `advanced/infra/03-hybrid-networking`
- 3.1.4 `advanced/infra/04-privatelink`
- 3.1.5 `advanced/infra/05-global-accelerator`

**3.2 compute**
- 3.2.1 `advanced/compute/01-eks-production`
- 3.2.2 `advanced/compute/02-lambda-internals`
- 3.2.3 `advanced/compute/03-ec2-deep`
- 3.2.4 `advanced/compute/04-app-runner-lightsail`

**3.3 data**
- 3.3.1 `advanced/data/01-data-lake`
- 3.3.2 `advanced/data/02-analytics-stack`
- 3.3.3 `advanced/data/03-streaming-architectures`
- 3.3.4 `advanced/data/04-dynamodb-internals`
- 3.3.5 `advanced/data/05-timestream-quantum-neptune`
- 3.3.6 `advanced/data/06-sagemaker-bedrock`

**3.4 security**
- 3.4.1 `advanced/security/01-zero-trust-patterns`
- 3.4.2 `advanced/security/02-encryption-deep`
- 3.4.3 `advanced/security/03-detective-incident-response`
- 3.4.4 `advanced/security/04-compliance`
- 3.4.5 `advanced/security/05-network-firewall-shield`

**3.5 devops**
- 3.5.1 `advanced/devops/01-cdk-mastery`
- 3.5.2 `advanced/devops/02-policy-as-code`
- 3.5.3 `advanced/devops/03-disaster-recovery`
- 3.5.4 `advanced/devops/04-finops`

**3.6 edge-iot**
- 3.6.1 `advanced/edge-iot/01-iot-core`
- 3.6.2 `advanced/edge-iot/02-lambda-edge-cloudfront-functions`
- 3.6.3 `advanced/edge-iot/03-wavelength-outposts-local-zones`

**3.7 capstones**
- 3.7.1 `advanced/capstones/01-saas-multi-tenant`
- 3.7.2 `advanced/capstones/02-event-driven-platform`
- 3.7.3 `advanced/capstones/03-ml-platform`
- 3.7.4 `advanced/capstones/04-global-web-app`
- 3.7.5 `advanced/capstones/05-data-platform`
