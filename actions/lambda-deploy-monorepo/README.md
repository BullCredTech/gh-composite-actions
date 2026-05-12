# lambda-deploy-monorepo

Composite action for **monorepo / shared-artifact** Lambda deploys: build **one** ZIP (same layouts as **`lambda-deploy`** — Python or Node), assume AWS via **OIDC**, then **`aws lambda update-function-code`** for **every** name in **`function-names`**.

Path: **`BullCredTech/gh-composite-actions/actions/lambda-deploy-monorepo@<tag>`**.

Use **`actions/lambda-deploy`** when you only update **one** function — it avoids listing a single line in **`function-names`**.

Requires **`actions/checkout@v4`** first. OIDC permissions:

```yaml
permissions:
  contents: read
  id-token: write
```

## When to use

- Multiple Lambdas share the **same deployment package** (e.g. one **`dist/`** with different handler entry points configured in Terraform).
- You want **one** install/build and **many** **`update-function-code`** calls without copy-pasting the composite.

## Inputs

Shared with **`lambda-deploy`** (see that README for behavioral detail): **`aws-role-arn`**, **`runtime`** (`python` \| `node`), **`aws-region`**, **`role-session-name`**, and all Python / Node packaging options (`lambda-*`, `node-*`).

| Input | Required | Description |
|--------|----------|-------------|
| `function-names` | **yes** | Newline-separated Lambda **names** (`FunctionName`). Empty lines skipped. Must produce at least one name. |

## Outputs

| Output | Description |
|--------|-------------|
| `zip-path` | ZIP path relative to repo root (same semantics as **`lambda-deploy`**). |

## Examples

### Node — many functions from one `npm run build`

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ vars.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: BullCredTech/gh-composite-actions/actions/lambda-deploy-monorepo@main
        with:
          aws-role-arn: ${{ env.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION || 'us-east-1' }}
          runtime: node
          node-build-command: npm run build
          function-names: |
            my-app-staging-api-a
            my-app-staging-api-b
            my-app-staging-worker
```

### Python — shared package to two Lambdas

```yaml
- uses: BullCredTech/gh-composite-actions/actions/lambda-deploy-monorepo@main
  with:
    aws-role-arn: arn:aws:iam::ACCOUNT:role/GitHubActionsLambdaRole
    runtime: python
    lambda-directory: lambda
    function-names: |
      my-svc-staging-consumer
      my-svc-staging-retry
```

## Versioning

Examples use **`@main`** until a **`gh-composite-actions`** release ships that includes **`lambda-deploy-monorepo`**; afterwards pin **`@v*`** like other composites (e.g. **`@v1.3.0`**) unless you intentionally follow **`main`**.
