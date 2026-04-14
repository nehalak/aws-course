# 04 — Cognito

## Concept
User Pool = user directory (sign-up, sign-in, MFA). Identity Pool = AWS credentials for users (federated).

## Exercises
1. **User Pool**: CDK pool with email verification, MFA optional, password policy. App client for a SPA.
2. **Hosted UI**: enable Cognito hosted login. Test sign-up/sign-in flow in browser. Inspect the JWT tokens.
3. **Federated IdP**: add Google as social IdP. Sign in with Google; see user in pool.
4. **Lambda triggers**: `PreSignUp` auto-confirm, `PostConfirmation` writes user to DynamoDB.
5. **Identity Pool**: map authenticated users to IAM role allowing `s3:GetObject` on their prefix.
6. **API Gateway integration**: HTTP API with Cognito JWT authorizer. Call `/me` with token → Lambda returns claims.

## Verification
- JWT decoded shows claims.
- Unauthenticated API call returns 401.

## Gotchas
- User Pool and Identity Pool are confusingly different.
- Custom domain for Hosted UI requires ACM cert in `us-east-1`.
- Migrating users out of Cognito is painful — plan up front.

## Cleanup
```bash
cdk destroy
```
