# Docker Build Reusable Workflows

Use these workflows to build container images with Buildah and push to either Azure Container Registry (ACR) or AWS Elastic Container Registry (ECR).

## Available Workflows

| Workflow | Registry | File |
|----------|----------|------|
| Build and push to ACR | Azure Container Registry | `.github/workflows/docker-build-acr.yml` |
| Build and push to ECR | AWS Elastic Container Registry | `.github/workflows/docker-build-ecr.yml` |

## Why Use These Workflows

- No Docker daemon required (Buildah + Skopeo)
- Works on self-hosted runners where privileged Docker is not available
- Standardized build and push across CPS repositories
- OIDC-based auth for Azure and AWS (no long-lived passwords)

## Common Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image_name` | Yes | - | Image repository name without registry prefix |
| `dockerfile` | No | `Dockerfile` | Path to Dockerfile |
| `context` | No | `.` | Build context path |
| `image_tag` | No | `github.run_id` | Image tag to publish |
| `environment` | No | `non-prod` | GitHub Environment name |
| `runner_label` | Yes | - | Runner label to execute on |

## ACR Workflow

### Extra Input

| Input | Required | Description |
|-------|----------|-------------|
| `acr_name` | Yes | ACR name without `.azurecr.io` |

### Required Secrets

| Secret | Description |
|--------|-------------|
| `AZURE_CLIENT_ID` | Azure app client ID |
| `AZURE_TENANT_ID` | Azure tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

### Usage Example

```yaml
jobs:
  docker-build:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/docker-build-acr.yml@v1.3
    with:
      image_name: my-app
      acr_name: myacrregistry
      runner_label: cps-cent-runner-nonprod-docker
      dockerfile: Dockerfile
      context: .
      image_tag: ${{ github.sha }}
      environment: non-prod
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## ECR Workflow

### Extra Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `ecr_registry` | Yes | Full ECR registry URL |
| `aws_region` | Yes | AWS region |

### Required Secret

| Secret | Description |
|--------|-------------|
| `AWS_ROLE_ARN` | IAM role ARN to assume via GitHub OIDC |

### Usage Example

```yaml
jobs:
  docker-build:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/docker-build-ecr.yml@v1.3
    with:
      image_name: my-app
      ecr_registry: 123456789012.dkr.ecr.eu-west-2.amazonaws.com
      aws_region: eu-west-2
      runner_label: cps-cent-runner-nonprod-docker
      dockerfile: Dockerfile
      context: .
      image_tag: ${{ github.sha }}
      environment: non-prod
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

## What Gets Published

Both workflows push:
- `${image_tag}` (or `${{ github.run_id }}` if not provided)
- `latest`

## Examples

See runnable examples:
- `examples/docker-build-acr.yml`
- `examples/docker-build-ecr.yml`

## Notes

- Ensure the runner has `buildah`, `skopeo`, and cloud CLIs installed.
- Prefer tagged references such as `@v1.3` over `@main`.
