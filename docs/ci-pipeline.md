# CI Pipeline

This repository uses two GitHub Actions workflows:

- `.github/workflows/golden-path-ci.yml` is the reusable workflow. It owns the standard validation and deployment jobs.
- `.github/workflows/todo-service-ci.yml` is the caller for this service. It runs the reusable workflow on pushes to `main` and on pull requests.

## Reusable Workflow

`golden-path-ci.yml` is triggered with `workflow_call`. It accepts the Node.js and Terraform versions plus flags that control infrastructure and deployment work. Actions are version-tagged, and the workflow does not use npm caching because this repository has no lock file.

### Jobs

| Job | When it runs | Purpose |
|---|---|---|
| `lint` | Every call | Installs dependencies with Node.js and runs ESLint for both the backend and frontend workspaces. This catches code-quality and syntax issues before merge. |
| `test` | Every call | Runs the backend Jest suite with coverage and writes the coverage totals to the GitHub job summary. Jest enforces the repository's coverage thresholds, including 80% lines and branches. |
| `security-scan` | When `run_terraform_plan` is `true` | Installs Checkov and scans `infra/`, failing on HIGH severity findings. This prevents infrastructure security regressions from reaching deployment. |
| `terraform-plan` | When `run_terraform_plan` is `true` | Authenticates to AWS with OIDC, installs the requested Terraform version, creates a plan for `infra/stacks/dev`, appends a plan summary, and uploads the plan artifact. This verifies that the infrastructure configuration can be planned. |
| `docker-build` | Pull requests | Builds the backend and frontend Dockerfiles without pushing images or using AWS credentials. This confirms that both service images remain buildable. |
| `terraform-apply` | When `run_terraform_apply` is `true` | Uses the Terraform plan artifact, authenticates with OIDC, applies the dev stack, and records the deployed service URL. |
| `build-and-push` | When `build_and_push` is `true` | Resolves the ECR repositories, builds and pushes both images with immutable and `latest` tags, then triggers an ECS deployment. |

The caller enables `run_terraform_plan` for both pull requests and pushes. It enables `run_terraform_apply` and `build_and_push` only for pushes to `main`.

## Adoption

A new service team needs a caller workflow in `.github/workflows/` that points to the reusable workflow. This is the minimum caller for validation and infrastructure planning:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

Set `run_terraform_apply` and `build_and_push` to `true` only for the deployment path on the protected default branch. The caller must keep `id-token: write` at its top-level `permissions` block; permissions declared only inside the reusable workflow do not grant the caller permission to mint an OIDC token.

## Required Checks

The reusable workflow provides these required checks:

- **Lint** validates ESLint rules in `packages/backend` and `packages/frontend`, catching issues that can be detected without running the application.
- **Test** runs Jest with coverage. The backend Jest configuration fails the job when coverage drops below the repository threshold, protecting behavior and test quality.
- **Security scan** runs Checkov against the Terraform configuration and blocks HIGH severity infrastructure findings.
- **Terraform plan** validates the dev stack by generating a plan and publishing it as both a job summary and an artifact. Reviewing a plan makes resource changes visible before apply.

The PR caller also runs `docker-build` after `lint` and `test`, so image packaging is checked before merge even though it is not one of the four core infrastructure status checks.

## OIDC Role Secret

The `terraform-plan` job obtains temporary AWS credentials with `aws-actions/configure-aws-credentials@v4`. It reads the role from the reusable workflow secret named `aws_role_arn`; the caller maps that input from the repository secret `AWS_ROLE_ARN`:

```yaml
secrets:
  aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

To configure it:

1. Create an AWS IAM role trusted by GitHub Actions through OIDC for the repository and its permitted branches or environments.
2. In the GitHub repository, open **Settings > Secrets and variables > Actions**.
3. Add a repository secret named `AWS_ROLE_ARN` containing the role ARN, for example `arn:aws:iam::123456789012:role/todo-service-github-actions`.
4. Keep the ARN out of workflow files. The reusable workflow must reference only `${{ secrets.aws_role_arn }}`.

The caller's `id-token: write` permission and the reusable job's `id-token: write` permission are both required for federation. The role should have only the AWS permissions needed to plan or apply the dev stack.
