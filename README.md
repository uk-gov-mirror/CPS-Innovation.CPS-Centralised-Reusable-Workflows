# CPS Centralised Reusable Workflows

Centralized repository for GitHub Actions reusable workflows used across CPS-Innovation repositories.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| [dotnet-build.yml](.github/workflows/dotnet-build.yml) | Build, test, and package .NET applications |
| [docker-build-acr-to-ecr.yml](.github/workflows/docker-build-acr-to-ecr.yml) | Build image in ACR and push it to ECR |
| [terraform-plan.yml](.github/workflows/terraform-plan.yml) | Terraform plan with change detection |
| [terraform-apply.yml](.github/workflows/terraform-apply.yml) | Terraform apply with approval gates |
| [terraform-destroy.yml](.github/workflows/terraform-destroy.yml) | Terraform destroy with approval gates |

---

## .NET Build Workflow

Build, test, and package .NET applications.

### Usage

```yaml
jobs:
  build:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/dotnet-build.yml@v1
    with:
      solution-file: 'src/MyApp.sln'
    secrets: inherit
```

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `runner` | No | `cps-cent-runner-nonprod` | Runner to use |
| `solution-file` | **Yes** | - | Path to .sln or .csproj file |
| `dotnet-version` | No | `8.0.x` | .NET SDK version (GitHub-hosted only) |
| `configuration` | No | `Release` | Build configuration |
| `run-tests` | No | `true` | Run unit tests |
| `use-github-packages` | No | `false` | Add GitHub Packages NuGet source |
| `timeout` | No | `30` | Job timeout in minutes |

### Example with GitHub Packages

```yaml
jobs:
  build:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/dotnet-build.yml@v1
    with:
      solution-file: 'src/MyApp.sln'
      use-github-packages: true
      run-tests: true
      timeout: 15
    secrets: inherit
```

---

## Docker Build Workflows (Removed — Did Not Work)

The ACR/ECR Buildah-based docker build reusable workflows were attempted and removed because they did not work. See [docs/docker-build.md](docs/docker-build.md) for the log of what was tried.

---

## Build in ACR and Push to ECR Workflow

Builds an image using `az acr build` and copies it to ECR with `crane`, avoiding the need for a local Docker daemon.

- Example workflow: [examples/docker-build-acr-to-ecr.yml](examples/docker-build-acr-to-ecr.yml)

### Usage

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  docker-build-acr-to-ecr:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/docker-build-acr-to-ecr.yml@v1
    with:
      acr-name: 'myacrregistry'
      acr-image-name: 'my-app'
      image-tag: ${{ github.sha }}
      aws-region: 'eu-west-2'
      ecr-registry: '123456789012.dkr.ecr.eu-west-2.amazonaws.com'
      ecr-image-name: 'my-app'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `runner` | No | `cps-cent-runner-nonprod` | Runner to use |
| `acr-name` | **Yes** | - | ACR name without `.azurecr.io` |
| `acr-image-name` | **Yes** | - | Image repository name in ACR |
| `image-tag` | No | `latest` | Tag applied to both ACR and ECR unless `ecr-image-tag` is set |
| `dockerfile-path` | No | `Dockerfile` | Path to Dockerfile |
| `build-context` | No | `.` | Docker build context path |
| `aws-region` | **Yes** | - | AWS region for ECR |
| `ecr-registry` | **Yes** | - | ECR registry hostname |
| `ecr-image-name` | **Yes** | - | Image repository name in ECR |
| `ecr-image-tag` | No | - (falls back to `image-tag`) | Optional ECR-specific tag |
| `crane-version` | No | `v0.19.1` | Version of `crane` to install |
| `timeout` | No | `30` | Job timeout in minutes |

### Secrets Required

| Secret | Description |
|--------|-------------|
| `AZURE_CLIENT_ID` | Azure AD app client ID |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |
| `AWS_ROLE_ARN` | IAM role ARN to assume via GitHub OIDC |

---

## Terraform Workflows

Infrastructure deployment with plan, apply, and destroy stages.

### Prerequisites

1. **Azure OIDC Authentication** - Configure federated credentials in your Azure AD app
2. **GitHub Secrets** - Add to your repository:
   - `AZURE_CLIENT_ID`
   - `AZURE_TENANT_ID`
   - `AZURE_SUBSCRIPTION_ID`
3. **GitHub Environment** - Create environments (e.g., `non-prod`, `prod`) with approval gates

### Inputs (Same for all Terraform workflows)

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `runner` | No | `cps-cent-runner-nonprod` | Runner to use |
| `working-directory` | No | `.` | Terraform working directory |
| `var-file` | **Yes** | - | Path to tfvars file |
| `environment` | **Yes** | - | Environment name / GitHub environment |
| `state-resource-group` | **Yes** | - | Resource group containing state storage |
| `state-storage-account` | **Yes** | - | Storage account name for tfstate |
| `state-container` | No | `tfstate` | Storage container name |
| `state-key` | **Yes** | - | State file name (e.g., `nonprod.tfstate`) |
| `timeout` | No | `30` | Job timeout in minutes |

### Secrets Required

| Secret | Description |
|--------|-------------|
| `AZURE_CLIENT_ID` | Azure AD app client ID |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

---

### Terraform Plan

Run terraform plan and detect changes.

```yaml
jobs:
  plan:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/terraform-plan.yml@v1
    with:
      var-file: 'environments/non-prod.tfvars'
      environment: 'nonprod'
      state-resource-group: 'rg-tfstate-nonprod'
      state-storage-account: 'sttfstatenonprod'
      state-container: 'tfstate'
      state-key: 'myapp-nonprod.tfstate'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Output:** `has_changes` - `true` if changes detected, `false` otherwise.

---

### Terraform Apply

Apply changes with environment approval gates.

```yaml
jobs:
  apply:
    needs: plan
    if: needs.plan.outputs.has_changes == 'true'
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/terraform-apply.yml@v1
    with:
      var-file: 'environments/non-prod.tfvars'
      environment: 'non-prod'  # GitHub environment with approvers
      state-resource-group: 'rg-tfstate-nonprod'
      state-storage-account: 'sttfstatenonprod'
      state-container: 'tfstate'
      state-key: 'myapp-nonprod.tfstate'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### Terraform Destroy

Destroy infrastructure (manual trigger recommended).

```yaml
jobs:
  destroy:
    if: github.event_name == 'workflow_dispatch'
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/terraform-destroy.yml@v1
    with:
      var-file: 'environments/non-prod.tfvars'
      environment: 'non-prod'
      state-resource-group: 'rg-tfstate-nonprod'
      state-storage-account: 'sttfstatenonprod'
      state-container: 'tfstate'
      state-key: 'myapp-nonprod.tfstate'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### Full Terraform Pipeline Example

Complete pipeline with plan, apply, and destroy:

```yaml
name: Terraform Deploy

on:
  push:
    branches: [main]
    paths-ignore:
      - '*.md'
  workflow_dispatch:
    inputs:
      action:
        description: 'Action to perform'
        type: choice
        default: 'plan'
        options:
          - plan
          - apply
          - destroy

jobs:
  plan:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/terraform-plan.yml@v1
    with:
      var-file: 'environments/non-prod.tfvars'
      environment: 'nonprod'
      state-resource-group: 'rg-tfstate-nonprod'
      state-storage-account: 'sttfstatenonprod'
      state-key: 'myapp-nonprod.tfstate'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  apply:
    needs: plan
    if: |
      needs.plan.outputs.has_changes == 'true' ||
      (github.event_name == 'workflow_dispatch' && github.event.inputs.action == 'apply')
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/terraform-apply.yml@v1
    with:
      var-file: 'environments/non-prod.tfvars'
      environment: 'non-prod'
      state-resource-group: 'rg-tfstate-nonprod'
      state-storage-account: 'sttfstatenonprod'
      state-key: 'myapp-nonprod.tfstate'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  destroy:
    if: github.event_name == 'workflow_dispatch' && github.event.inputs.action == 'destroy'
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/terraform-destroy.yml@v1
    with:
      var-file: 'environments/non-prod.tfvars'
      environment: 'non-prod'
      state-resource-group: 'rg-tfstate-nonprod'
      state-storage-account: 'sttfstatenonprod'
      state-key: 'myapp-nonprod.tfstate'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## Versioning

Always reference workflows by **tag** (not `main`) for stability:

```yaml
# Recommended - major version tag
uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/dotnet-build.yml@v1

# Pin to exact version
uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/dotnet-build.yml@v1.2.0

# NOT recommended for production
uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/dotnet-build.yml@main
```

---

## Runner Selection

| Runner | When to Use |
|--------|-------------|
| `cps-cent-runner-nonprod` | Default for non-prod environments |
| `cps-cent-runner-prod` | Production deployments |
| `ubuntu-latest` | Public repos or when self-hosted unavailable |

Self-hosted runners have pre-installed tools (.NET, Terraform, Azure CLI) - faster builds, no firewall issues.

---

## Support

- Issues: [GitHub Issues](https://github.com/CPS-Innovation/CPS-Centralised-Reusable-Workflows/issues)
- Teams: #devops-support channel
