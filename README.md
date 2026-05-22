# Private Terraform Modules GitHub Action

This GitHub Action configures Git authentication so Terraform can download private Terraform modules hosted in GitHub repositories.

It automatically injects GitHub credentials into Git operations during `terraform init`.

---

# Why This Action Exists

Terraform modules can be sourced from GitHub repositories:

```hcl
module "vpc" {
  source   = "git@github.com:kemeattang/terraform_modules.git//aws/iam-role?ref=v1.0.0"  # this isn't private but get what i am putting down 
}
```

If the repository is private, Terraform cannot download the module unless authentication is configured.

This action solves that problem by configuring Git authentication automatically inside GitHub Actions workflows.

---



---

# Creating the GitHub Token

This action requires a GitHub token with access to the private Terraform module repositories.

---

## Step 1 — Create a Personal Access Token

Go to:

```text
GitHub → Profile Picture → Settings → Developer Settings → Personal Access Tokens
```

Choose:

- Fine-grained personal access token (recommended)
OR
- Classic personal access token

---

## Step 2 — Configure Token Permissions

### Fine-Grained Token (Recommended)

#### Repository Access

Grant access only to the repositories containing Terraform modules.

#### Repository Permissions

Set:

```text
Contents: Read-only
```

This is usually all Terraform requires.

Generate and copy the token.

---

# Creating the GitHub Secret

You now need to store the token securely in GitHub Secrets.

---

## Option 1 — Repository Secret

Use this when only one repository needs access.

Go to the repository running Terraform.

Navigate to:

```text
Settings → Secrets and variables → Actions
```

Click:

```text
New repository secret
```

Create the following secret:

| Name | Value |
|---|---|
| TERRAFORM_MODULES_TOKEN | <your-github-token> |

Click:

```text
Add secret
```

---

## Option 2 — Organization Secret

Use this when multiple repositories need access to the same Terraform modules.

Go to:

```text
Organization Settings → Secrets and variables → Actions
```

Click:

```text
New organization secret
```

Create:

| Name | Value |
|---|---|
| TERRAFORM_MODULES_TOKEN | <your-github-token> |

Choose which repositories can access the secret.

Save the secret.

---

# Using the Action

Create a workflow file:

```text
.github/workflows/terraform.yml
```

Example workflow:

```yaml
name: Terraform

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  terraform:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure private Terraform module access
        uses: ./.github/actions/private-terraform-modules
        with:
          org: my-org
          token: ${{ secrets.TERRAFORM_MODULES_TOKEN }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan
```

---

# Example Terraform Module

```hcl
module "networking" {
  source = "git::https://github.com/my-org/terraform-modules.git//networking?ref=v1.2.0"
}
```

During:

```bash
terraform init
```

Terraform uses Git internally to clone the module repository.

This action configures Git so authentication works automatically.

---

# How It Works

The action executes:

```bash
git config --global url."https://git:TOKEN@github.com/ORG".insteadOf "https://github.com/ORG"
```

Git automatically rewrites:

```text
https://github.com/my-org/private-module
```

into:

```text
https://git:TOKEN@github.com/my-org/private-module
```

allowing access to private repositories.

---

# Security Recommendations

## Recommended

- Store tokens only in GitHub Secrets
- Use read-only permissions
- Restrict repository access where possible
- Rotate tokens periodically

## Avoid

- Hardcoding tokens in workflows
- Printing tokens in logs
- Committing tokens into repositories

---

# Troubleshooting

---

## Terraform Cannot Find the Module Repository

Verify:

- Repository exists
- Repository is private
- Token has access
- `org` input matches the GitHub organization

Example:

```yaml
with:
  org: my-org
```

must match:

```text
https://github.com/my-org/private-module
```

---

## Authentication Failed

Verify the secret exists:

```yaml
${{ secrets.TERRAFORM_MODULES_TOKEN }}
```

Verify token permissions include:

```text
Contents: Read-only
```

---

## Terraform Prompts for Username/Password

Ensure Terraform module sources use HTTPS.

Correct:

```hcl
source = "git::https://github.com/my-org/repo.git"
```

Incorrect:

```hcl
source = "git::git@github.com:my-org/repo.git"
```

This action supports HTTPS Git sources only.

---

# Inputs

| Input | Required | Description |
|---|---|---|
| org | Yes | GitHub organization containing Terraform module repositories |
| token | Yes | GitHub token with access to private repositories |

---

# Example Usage

```yaml
- name: Configure private Terraform modules
  uses: ./.github/actions/private-terraform-modules
  with:
    org: my-org
    token: ${{ secrets.TERRAFORM_MODULES_TOKEN }}
```
