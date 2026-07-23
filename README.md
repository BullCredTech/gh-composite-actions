# gh-composite-actions

Centralized GitHub Actions composite actions shared across BullCredTech service repos.

## Available actions

| Action | Path | Description |
|--------|------|-------------|
| [`build-push-ecr`](./actions/build-push-ecr) | `actions/build-push-ecr` | Assume an AWS IAM role via OIDC and build/push a Docker image to Amazon ECR with Buildx + GHA cache. |
| [`eks-deploy`](./actions/eks-deploy) | `actions/eks-deploy` | Build/push to ECR (optional; includes `<environment>-latest` floating tag) and Helm deploy to EKS with rollout wait; or Helm-only with `skip-ecr-build`. |
| [`lambda-deploy`](./actions/lambda-deploy) | `actions/lambda-deploy` | Build a Lambda ZIP (**Python pip** under `lambda/` or **Node** `npm` + `dist/`), OIDC **`configure-aws-credentials`**, and **`aws lambda update-function-code`** (one function). |
| [`lambda-deploy-monorepo`](./actions/lambda-deploy-monorepo) | `actions/lambda-deploy-monorepo` | Same packaging as **`lambda-deploy`**, then **`update-function-code`** for **many** Lambda names (**`function-names`**, newline-separated). |
| [`slack-pr-notify`](./actions/slack-pr-notify) | `actions/slack-pr-notify` | Ao abrir um PR (`on: pull_request`), posta no Slack (Incoming Webhook) uma notificação em ficha, com descrição de 1 linha em PT via Gemini (fallback = título) e menção ao time responsável. Só notifica PRs para `target-branches` (default `main`) e, por padrão, só depois que todos os checks passarem (`wait-for-checks`). |

## Usage

Reference an action from any workflow as:

```yaml
- uses: BullCredTech/gh-composite-actions/actions/<name>@<ref>
```

Where `<ref>` is `main` (always-latest) or a stable tag like `v1`. Each action requires the calling job to first run `actions/checkout@v4` if it needs files from the consumer repo (e.g. Dockerfile, Helm chart, values files).

OIDC-using actions need:

```yaml
permissions:
  contents: read
  id-token: write
```

See each action's README for full input/output reference, examples, and **BullCredTech defaults** (shared-services ECR registry and `GitHubActionsECRRole` for ECR OIDC—override when using another account).

## Versioning

`.github/workflows/tag-on-push.yml` creates a semantic version tag on every push to `main`, using [conventional commits](https://www.conventionalcommits.org/):

- `fix:` → patch (e.g. `1.0.0` → `1.0.1`)
- `feat:` → minor (e.g. `1.0.0` → `1.1.0`)
- `feat!:` or `BREAKING CHANGE` → major (e.g. `1.0.0` → `2.0.0`)

If there are no version-bumping commits since the last tag, no new tag is created. Pin consumers to a tag (e.g. `@v1`) for stable behavior.
