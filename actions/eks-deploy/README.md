# eks-deploy

Composite action: optionally **build and push** to Amazon ECR (always adds floating tag **`<environment>-latest`**, e.g. `staging-latest`), then **Helm upgrade --install** on EKS with `kubectl` rollout wait and optional failure diagnostics.

This replaces the former **`helm-deploy-eks`** action (renamed and extended). For **Helm-only** deploys (image already in ECR), set **`skip-ecr-build: true`** and pass **`image-repository`** + **`image-tag`**.

**Reference:** `BullCredTech/gh-composite-actions/actions/eks-deploy@main` (or pin to a tag after release).

The IAM role used for EKS must be allowed by an EKS Access Entry (or `aws-auth`) with RBAC on the target namespace.

## Inputs

### Mode

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `skip-ecr-build` | no | `false` | If `true`, skip ECR; require `image-repository` and `image-tag`. |

### AWS / ECR build (when `skip-ecr-build` is false)

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | no | `staging` | Used for floating tag `<environment>-latest`. |
| `aws-region` | no | `us-east-1` | AWS region. |
| `aws-role-arn` | no | `""` | When set, used for the ECR build step (overrides `ecr-role-arn` there) and as fallback for EKS when `eks-role-arn` is empty. |
| `ecr-role-arn` | no | `arn:aws:iam::240037736937:role/GitHubActionsECRRole` | Default BullCredTech ECR OIDC role (superseded for ECR when `aws-role-arn` is set). |
| `eks-role-arn` | no | `""` | Role for EKS/Helm; falls back to `aws-role-arn`. |
| `ecr-role-session-name` | no | `gha-eks-deploy-ecr` | STS session name for ECR. |
| `role-session-name` | no | `gha-eks-deploy` | STS session name for EKS. |
| `ecr-registry` | no | `240037736937.dkr.ecr.us-east-1.amazonaws.com` | ECR registry hostname when building. |
| `ecr-repository` | yes* | `""` | ECR repo name. *Required when building. |
| `image-tag` | no | `""` | Primary tag; when building, defaults to full `GITHUB_SHA` if empty. |
| `extra-image-tags` | no | `""` | Extra ECR tags (newline-separated), besides `<environment>-latest`. |
| `context` | no | `.` | Docker build context. |
| `dockerfile` | no | `./Dockerfile` | Dockerfile path. |
| `build-args` | no | `""` | Newline-separated `KEY=VALUE` build args. |
| `cache-from` | no | `type=gha` | Buildx `--cache-from`. |
| `cache-to` | no | `type=gha,mode=min` | Buildx `--cache-to`. |

### Helm / EKS

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `cluster-name` | yes | - | EKS cluster name. |
| `namespace` | yes | - | Kubernetes namespace. |
| `release` | yes | - | Helm release (must match Deployment name for rollout wait). |
| `chart-path` | yes | - | Helm chart directory. |
| `values-file` | yes | - | Values file path. |
| `helm-version` | no | `v3.15.4` | Helm CLI version. |
| `timeout` | no | `5m` | Helm and `kubectl rollout` timeout. |
| `extra-set` | no | `""` | Newline-separated extra `--set key=value`. |
| `create-namespace` | no | `true` | `--create-namespace`. |
| `atomic` | no | `true` | `--atomic`. |
| `wait` | no | `true` | `--wait` when not atomic. |
| `diagnostics-on-failure` | no | `true` | Print diagnostics on failure. |

When **not** skipping ECR build, **`image.repository`** and **`image.tag`** are set from `ecr-registry` / `ecr-repository` and the resolved primary tag.

When **`skip-ecr-build: true`**:

| Input | Required | Description |
|-------|----------|-------------|
| `image-repository` | yes | Full repo without tag (e.g. `123.dkr.ecr.region.amazonaws.com/my-app`). |
| `image-tag` | yes | Tag to deploy. |

## Outputs

| Output | Description |
|--------|-------------|
| `image-uri` | Primary image URI (from build) or `image-repository:image-tag` when skipping ECR. |

## Required permissions

```yaml
permissions:
  contents: read
  id-token: write
```

## Example: build, push (`staging-latest`), and deploy

```yaml
- uses: actions/checkout@v4

- uses: BullCredTech/gh-composite-actions/actions/eks-deploy@main
  with:
    environment: staging
    ecr-role-arn: arn:aws:iam::111:role/GitHubActionsECRRole
    eks-role-arn: arn:aws:iam::222:role/GitHubActionsEKSRole
    ecr-role-session-name: gha-my-service-ecr
    role-session-name: gha-my-service-deploy
    ecr-registry: 111.dkr.ecr.us-east-1.amazonaws.com
    ecr-repository: my-service
    image-tag: ${{ github.sha }}
    cluster-name: bull-staging
    namespace: my-app
    release: my-service
    chart-path: helm-chart
    values-file: helm-chart/values-staging.yaml
```

## Example: Helm-only (image already pushed)

```yaml
- uses: actions/checkout@v4

- uses: BullCredTech/gh-composite-actions/actions/eks-deploy@main
  with:
    skip-ecr-build: true
    aws-role-arn: arn:aws:iam::222:role/GitHubActionsEKSRole
    role-session-name: gha-my-service-deploy
    cluster-name: bull-staging
    namespace: my-app
    release: my-service
    chart-path: helm-chart
    values-file: helm-chart/values-staging.yaml
    image-repository: 111.dkr.ecr.us-east-1.amazonaws.com/my-service
    image-tag: ${{ needs.build-push.outputs.image_tag }}
```
