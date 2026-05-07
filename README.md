# gh-composite-actions

Centralized GitHub Actions composite actions shared across BullCredTech service repos.

## Available actions

| Action | Path | Description |
|--------|------|-------------|
| [`build-push-ecr`](./actions/build-push-ecr) | `actions/build-push-ecr` | Assume an AWS IAM role via OIDC and build/push a Docker image to Amazon ECR with Buildx + GHA cache. |
| [`helm-deploy-eks`](./actions/helm-deploy-eks) | `actions/helm-deploy-eks` | Assume an AWS IAM role via OIDC, configure `kubectl`/`helm`, and run `helm upgrade --install` with rollout wait. |

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

See each action's README for full input/output reference and examples.

## Versioning

`.github/workflows/tag-on-push.yml` creates a semantic version tag on every push to `main`, using [conventional commits](https://www.conventionalcommits.org/):

- `fix:` → patch (e.g. `1.0.0` → `1.0.1`)
- `feat:` → minor (e.g. `1.0.0` → `1.1.0`)
- `feat!:` or `BREAKING CHANGE` → major (e.g. `1.0.0` → `2.0.0`)

If there are no version-bumping commits since the last tag, no new tag is created. Pin consumers to a tag (e.g. `@v1`) for stable behavior.
