# lambda-deploy

Composite action: **build** a Lambda deployment ZIP (**Python** or **Node.js**), **assume** an IAM role with GitHub **OIDC**, and run **`aws lambda update-function-code`**.

Path: **`BullCredTech/gh-composite-actions/actions/lambda-deploy@<tag>`** (e.g. **`@v1.2.0`**).

Multiple Lambdas that share **one** build artifact → use **`actions/lambda-deploy-monorepo`** (**`function-names`**).

The action does **not** run `actions/checkout` — the calling workflow must check out the consuming repository first (same pattern as other actions in this collection).

## Prerequisites

- Job permissions for OIDC:

  ```yaml
  permissions:
    contents: read
    id-token: write
  ```

- Existing Lambda function in AWS that this role may update.
- IAM role trust + permissions for `sts:AssumeRoleWithWebIdentity` from GitHub Actions and `lambda:UpdateFunctionCode` (and any read APIs required by the CLI).

---

## Inputs

| Input | Required | Default | Description |
|--------|----------|---------|-------------|
| `aws-role-arn` | **yes** | — | IAM role ARN assumed via OIDC. |
| `function-name` | **yes** | — | `FunctionName` passed to `aws lambda update-function-code`. |
| `runtime` | **yes** | — | `python` or `node` (literal). |
| `aws-region` | no | `us-east-1` | AWS region. |
| `role-session-name` | no | `gha-deploy-lambda-code` | STS session name. |

### When `runtime` is `python`

| Input | Default | Description |
|--------|---------|-------------|
| `python-version` | `3.11` | `setup-python` version. |
| `lambda-directory` | `lambda` | Folder with `requirements.txt` and handler file (repo-relative). |
| `lambda-handler-file` | `handler.py` | File copied next to deps in build output. |
| `lambda-zip-path` | `lambda/deployment.zip` | Output ZIP relative to workspace (parent dirs created). |

Equivalent to **`pip install -t`** + **`cp handler`** + zip (same layout as **`bull-lambda-celcoin-contract-status`** workflows).

### When `runtime` is `node`

| Input | Default | Description |
|--------|---------|-------------|
| `node-version` | `20` | `setup-node` version; enables npm cache from lockfile where present. |
| `node-working-directory` | `.` | Directory containing **`package.json`**. |
| `node-install-command` | `npm ci` | Runs after `cd` into `node-working-directory`. |
| `node-build-command` | `npm run build:full` | Build step (override with **`npm run build`**, etc.). |
| `node-dist-directory` | `dist` | Folder zipped as the Lambda artifact (relative to `node-working-directory`). |
| `node-install-prod-deps-in-dist` | `false` | If `true`, copies **`package.json`** + **`package-lock.json`** into `dist/` and runs **`npm ci --omit=dev --prefix`** `dist/` before zipping (RDS-style Lambdas that need **`node_modules`** beside **`*.js`**). |
| `node-zip-path` | `deployment.zip` | Output ZIP relative to **`node-working-directory`**. |

---

## Outputs

| Output | Description |
|--------|-------------|
| `zip-path` | Path to the ZIP (relative to repository root), as passed to `fileb://` for **`update-function-code`**. Useful for downstream steps or artifacts. |

---

## Examples

### Python (`lambda/requirements.txt` + `lambda/handler.py`)

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: BullCredTech/gh-composite-actions/actions/lambda-deploy@v1.2.0
        with:
          aws-role-arn: arn:aws:iam::235527546773:role/GitHubActionsLambdaRole
          aws-region: us-east-1
          function-name: bull-lambda-staging-celcoin-contract-status
          runtime: python
```

### Node (default `npm run build:full` + zip `dist/`)

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: BullCredTech/gh-composite-actions/actions/lambda-deploy@v1.2.0
    with:
      aws-role-arn: arn:aws:iam::235527546773:role/GitHubActionsLambdaRole
      function-name: bull-lambda-dataprev-retry-staging
      runtime: node
```

### Node (`npm run build` + prod `node_modules` inside `dist/`)

Typical **`bull-lambda-trigger-proposal-status`**-style artifact:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: BullCredTech/gh-composite-actions/actions/lambda-deploy@v1.2.0
    with:
      aws-role-arn: arn:aws:iam::235527546773:role/GitHubActionsLambdaRole
      function-name: bull-trigger-proposal-status-staging
      runtime: node
      node-build-command: npm run build
      node-install-prod-deps-in-dist: 'true'
```

---

## Versioning

Pin to a **`v*`** tag from this repo (for example **`@v1`**) unless you intentionally follow **`main`**.
