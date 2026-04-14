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
