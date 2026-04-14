# 02 — CDK Pipelines

## Concept
Self-mutating CI/CD built on CodePipeline. Pipeline redeploys itself when source changes.

## Exercises
1. **Pipeline from GitHub**: CDK Pipeline triggered on `main` push. Synth → Deploy to `dev` account/region.
2. **Multi-stage**: add `prod` stage with manual approval gate.
3. **Waves**: parallel deploy to `us-east-1` + `eu-west-1` in same stage.
4. **Test step**: add a `ShellStep` after deploy running integration tests (curl health endpoint, exit 1 on fail).
5. **Self-mutation**: commit a change to pipeline definition. Watch it update itself in first run.
6. **Cross-account**: bootstrap dev + prod accounts with trust. Deploy pipeline in tooling account, deploy app in others.

## Verification
- Commit to main triggers full pipeline.
- Failed test rolls back / blocks prod.

## Gotchas
- Needs GitHub connection (CodeStar) configured in console once.
- Cross-account bootstrap requires `--trust ACCOUNT`.
- Pipeline resource itself costs ~$1/mo.

## Cleanup
```bash
cdk destroy  # on each deployed stage
```
