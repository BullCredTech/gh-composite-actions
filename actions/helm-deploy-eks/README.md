# helm-deploy-eks

Composite action: assume an AWS IAM role via GitHub OIDC, configure `kubectl` and `helm`, run `helm lint` + `helm upgrade --install` with `--atomic`, and wait for the Deployment rollout to complete.

**Reference:** `BullCredTech/gh-composite-actions/actions/helm-deploy-eks@main` (or pin to a tag, e.g. `@v1`).

> Note: the IAM role must be allowed by an EKS Access Entry (or `aws-auth` ConfigMap) on the target cluster, with sufficient RBAC on the target namespace (typically `AmazonEKSEditPolicy` scoped to the namespace).

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `aws-region` | no | `us-east-1` | AWS region of the EKS cluster. |
| `aws-role-arn` | yes | - | IAM role ARN to assume via GitHub OIDC. |
| `role-session-name` | no | `gha-helm-deploy-eks` | STS session name. |
| `cluster-name` | yes | - | EKS cluster name. |
| `namespace` | yes | - | Kubernetes namespace. |
| `release` | yes | - | Helm release name (must match the Deployment name produced by the chart for rollout wait). |
| `chart-path` | yes | - | Filesystem path to the Helm chart directory in the calling repo. |
| `values-file` | yes | - | Path to the environment-specific values file in the calling repo. |
| `image-repository` | yes | - | Container image repository (without tag). Used as `--set image.repository=...`. |
| `image-tag` | yes | - | Container image tag to deploy. Used as `--set image.tag=...`. |
| `helm-version` | no | `v3.15.4` | Helm CLI version to install. |
| `timeout` | no | `5m` | Helm `--timeout` and `kubectl rollout status --timeout` value. |
| `extra-set` | no | `""` | Newline-separated additional `--set key=value` pairs. |
| `create-namespace` | no | `"true"` | Whether to pass `--create-namespace` to `helm upgrade`. |

## Required permissions

```yaml
permissions:
  contents: read
  id-token: write
```

## Example

```yaml
- uses: actions/checkout@v4

- uses: BullCredTech/gh-composite-actions/actions/helm-deploy-eks@main
  with:
    aws-region: us-east-1
    aws-role-arn: arn:aws:iam::240037736937:role/GitHubActionsEKSRole
    cluster-name: bull-staging
    namespace: propostas
    release: bull-web-promotor-propostas
    chart-path: helm-chart
    values-file: helm-chart/values-staging.yaml
    image-repository: 240037736937.dkr.ecr.us-east-1.amazonaws.com/bull-web-promotor-propostas
    image-tag: ${{ github.sha }}
```
