# Walkthrough — 04 CloudFormation Primer

## About this service
**AWS CloudFormation (CFN)** is AWS's native Infrastructure-as-Code service. You write YAML/JSON describing resources, submit it, and CFN creates/updates/deletes them as a unit called a **stack**. CDK compiles TO CloudFormation, so understanding CFN = understanding CDK's failure modes.

**Why it matters:** error messages you'll see as a CDK user (`ROLLBACK_COMPLETE`, `UPDATE_FAILED`, drift) come from CFN. You can't debug CDK without CFN literacy.

**When to use CFN directly:** legacy projects, tiny stacks, when your team doesn't know TypeScript/Python, AWS Quick Starts.
**When NOT to use CFN:** complex apps with lots of repetition or logic (use CDK). Cross-cloud (use Terraform).

## Estimated cost
- CloudFormation itself: **free** (you pay only for what it creates)
- S3 bucket used in this lesson: **~$0.02/month** (tiny)
- Total: **<$0.10/month** for this lesson

---

## Step 1: Raw CFN
> **Why:** Seeing YAML directly before letting CDK write it for you builds intuition. You'll recognize the structure when reading `cdk synth` output.

Create `bucket.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Learn CFN

Parameters:
  BucketName:
    Type: String

Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: !Ref BucketName
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: Purpose
          Value: learning

Outputs:
  BucketArn:
    Value: !GetAtt MyBucket.Arn
```

Deploy:
```bash
BUCKET="learn-cfn-$(aws sts get-caller-identity --query Account --output text)-$(date +%s)"
aws cloudformation deploy \
  --stack-name learn-cfn \
  --template-file bucket.yaml \
  --parameter-overrides BucketName=$BUCKET
```

## Step 2: Update + observe events
> **Why:** Every CFN change produces a timeline of events. Reading these is how you debug failed deploys. Intentionally updating lets you see the flow on a working stack first.

Add a tag:
```yaml
      Tags:
        - Key: Purpose
          Value: learning
        - Key: Owner
          Value: cdk-dev
```

Redeploy, then:
```bash
aws cloudformation describe-stack-events --stack-name learn-cfn \
  --max-items 10 \
  --query 'StackEvents[].[Timestamp,ResourceStatus,ResourceType,ResourceStatusReason]' \
  --output table
```

Look for `UPDATE_IN_PROGRESS → UPDATE_COMPLETE`.

## Step 3: Force a failure
> **Why:** Seeing a real rollback with your own eyes — including the stuck `ROLLBACK_COMPLETE` state — teaches you that recovery often means delete + recreate. This is the most common "why is CDK broken?" moment.

Set `BucketName: amazon` (already taken). Redeploy.

Expected events:
```
CREATE_FAILED    AWS::S3::Bucket    amazon already exists
ROLLBACK_IN_PROGRESS
ROLLBACK_COMPLETE
```

Stack is stuck in `ROLLBACK_COMPLETE`. Only option: `delete-stack`.

## Step 4: Drift detection
> **Why:** CFN doesn't prevent humans from clicking in the console and changing resources it manages. Drift detection flags those differences. In production, drift is a red flag — someone bypassed IaC.

1. **Console → CloudFormation → Stacks → learn-cfn → Stack actions → Detect drift**.
2. Add a manual tag to the bucket via S3 console.
3. Re-run drift detection → resource shows **MODIFIED**.

CLI:
```bash
aws cloudformation detect-stack-drift --stack-name learn-cfn
aws cloudformation describe-stack-resource-drifts --stack-name learn-cfn
```

## Step 5: DeletionPolicy test
> **Why:** Losing a production database because someone did `cdk destroy` is a disaster. `DeletionPolicy: Retain` keeps the resource when the stack goes. Learn it now.

```bash
aws cloudformation delete-stack --stack-name learn-cfn
aws s3 ls | grep $BUCKET     # bucket still exists — retained!
aws s3 rb s3://$BUCKET --force
```

## Step 6: CDK parity
> **Why:** Seeing `cdk synth` output next to hand-written CFN demystifies CDK. It's just generating templates for you.

```typescript
new s3.Bucket(this, 'MyBucket', {
  versioned: true,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});
```

```bash
cdk synth > cdk-bucket.yaml
diff bucket.yaml cdk-bucket.yaml
```

## Common Errors
- **`ROLLBACK_COMPLETE` can't update** → delete + recreate.
- **`The bucket you tried to delete is not empty`** → `aws s3 rm s3://$BUCKET --recursive` first.
- **Bucket name globally unique** — always add account ID or UUID.
