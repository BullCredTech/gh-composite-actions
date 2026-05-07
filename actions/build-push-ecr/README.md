# build-push-ecr

Composite action: assume an AWS IAM role via GitHub OIDC, log in to Amazon ECR, build a Docker image with Buildx (with GHA cache), and push one or more tags. No long-lived AWS credentials.

**Reference:** `BullCredTech/gh-composite-actions/actions/build-push-ecr@main` (or pin to a tag, e.g. `@v1`).

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `aws-region` | no | `us-east-1` | AWS region of the ECR registry. |
| `aws-role-arn` | yes | - | IAM role ARN to assume via GitHub OIDC. |
| `role-session-name` | no | `gha-build-push-ecr` | STS session name. |
| `ecr-registry` | yes | - | ECR registry hostname (e.g. `123456789012.dkr.ecr.us-east-1.amazonaws.com`). |
| `ecr-repository` | yes | - | ECR repository name. |
| `image-tag` | yes | - | Primary image tag (e.g. commit SHA). |
| `extra-tags` | no | `""` | Newline-separated list of additional tags to push. |
| `context` | no | `.` | Docker build context. |
| `dockerfile` | no | `./Dockerfile` | Path to the Dockerfile. |
| `build-args` | no | `""` | Newline-separated list of build args (`KEY=VALUE`). |

## Outputs

| Output | Description |
|--------|-------------|
| `image-uri` | Fully-qualified URI of the pushed image with the primary tag. |

## Required permissions

```yaml
permissions:
  contents: read
  id-token: write
```

## Example

```yaml
- uses: actions/checkout@v4

- uses: BullCredTech/gh-composite-actions/actions/build-push-ecr@main
  with:
    aws-region: us-east-1
    aws-role-arn: arn:aws:iam::240037736937:role/GitHubActionsECRRole
    ecr-registry: 240037736937.dkr.ecr.us-east-1.amazonaws.com
    ecr-repository: my-service
    image-tag: ${{ github.sha }}
    extra-tags: |
      staging-latest
```
