# 01 — Zero Trust Patterns

## Concept
Never trust network; verify identity + device + context per request. Verified Access (corp apps), Verified Permissions (Cedar policies).

## Exercises
1. **Verified Access**: deploy a private web app behind AWS VA. Trust provider = Cognito or IdP. Access from browser with login.
2. **Access policies**: Cedar policy that allows only users with `department=eng` AND trusted device.
3. **Verified Permissions**: create a policy store; write Cedar policies for a doc-sharing app (`user can read doc if owner or shared-with`).
4. **Integrate with Cognito**: Identity source = Cognito user pool; authorize API with VP.
5. **Device trust**: integrate with Jamf/CrowdStrike connector (or mock data).

## Verification
- Access denied without proper attributes.
- Cedar policy evaluation returns expected allow/deny.

## Gotchas
- Cedar syntax has learning curve; use Cedar playground.
- VA pricing per app per hour.

## Cleanup
```bash
cdk destroy
```
