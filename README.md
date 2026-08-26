# CPS Centralised Reusable Workflows

Centralized repository for GitHub Actions reusable workflows used across CPS-Innovation repositories.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| [dotnet-build.yml](.github/workflows/dotnet-build.yml) | Build, test, and package .NET applications |
| [docker-build-acr.yml](.github/workflows/docker-build-acr.yml) | Build and push container images to Azure Container Registry |
| [docker-build-ecr.yml](.github/workflows/docker-build-ecr.yml) | Build and push container images to AWS Elastic Container Registry |
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

## Docker Build Workflows

Build container images with Buildah and push to ACR or ECR using reusable workflows.

- Full guide: [docs/docker-build.md](docs/docker-build.md)
- Example workflows:
  - [examples/docker-build-acr.yml](examples/docker-build-acr.yml)
  - [examples/docker-build-ecr.yml](examples/docker-build-ecr.yml)

### Quick Usage (ACR)

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  docker-build:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/docker-build-acr.yml@v1.3
    with:
      image_name: 'my-app'
      acr_name: 'myacrregistry'
      runner_label: 'cps-cent-runner-nonprod-docker'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Quick Usage (ECR)

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  docker-build:
    uses: CPS-Innovation/CPS-Centralised-Reusable-Workflows/.github/workflows/docker-build-ecr.yml@v1.3
    with:
      image_name: 'my-app'
      ecr_registry: '123456789012.dkr.ecr.eu-west-2.amazonaws.com'
      aws_region: 'eu-west-2'
      runner_label: 'cps-cent-runner-nonprod-docker'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

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
