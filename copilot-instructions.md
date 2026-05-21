This instruction is optimized for generating the `on-board.yml` GitHub Actions workflow with proper branching, PR automation, template processing, and onboarding orchestration.

---

# GitHub Copilot Instructions — Application Onboarding Workflow

## Objective

Generate a GitHub Actions workflow named:

```text
.github/workflows/on-board.yml
```

The workflow automates onboarding for:

1. Application CI setup
2. Application CD setup
3. core_infrastructure setup

The workflow must:

* Generate files from templates
* Create temporary onboarding branches
* Commit generated changes
* Raise Pull Requests only
* Never push directly to target branches

---

# Workflow Requirements

## Workflow Name

```yaml
name: Application Onboarding
```

## Trigger

Use:

```yaml
on:
  workflow_dispatch:
```

---

# Workflow Inputs

The workflow must support the following inputs.

```yaml
inputs:
  RepoName:
  AppName:
  Environment:
  APIBranchName:
  APITargetBranchName:
  UIBranchName:
  UITargetBranchName:
```

---

# Allowed Environment Values

Only allow:

* dev-i
* qa-i
* stage-i
* prod-i

Validation must fail for any other value.

Example validation logic:

```bash
case "$ENVIRONMENT" in
  dev-i|qa-i|stage-i|prod-i)
    echo "Valid environment"
    ;;
  *)
    echo "Invalid environment"
    exit 1
    ;;
esac
```

---

# Repository Structure

The workflow assumes the following template structure exists:

```text
template-files/

├── app-ci/
│   ├── .github/workflows/ci-api.yml
│   ├── bin/docker-entrypoint
│   └── Dockerfile
│
├── app-cd/
│   ├── .github/workflows/cd-dev.yml
│   ├── iac/aws/us-east-1/ui.yaml
│   ├── iac/aws/us-east-1/api.yaml
│   ├── ui-config.yaml
│   └── api-config.yaml
│
└── infra-cd/
    ├── iac/aws/us-east-1/ecr.yaml
    ├── iac/aws/us-east-1/iam-policy.yaml
    ├── iac/aws/us-east-1/iam-role.yaml
    ├── iac/aws/us-east-1/s3.yaml
    └── iac/aws/us-east-1/secret-manager.yaml
```

---

# Placeholder Replacement Rules

Templates may contain placeholders like: 

```text
__APP_NAME__ 
__CF_APP_NAME__ (cloudformation compatible)
__ENVIRONMENT__
__REPO_NAME__
__RUBY_VERSION__
```

IF NOT LIKELY, make the respective change on template files to placeholders dynamically

The workflow must replace placeholders dynamically using workflow inputs.

Use:

```bash
find . -type f -exec sed -i "s|__APP_NAME__|${APP_NAME}|g" {} +
```

Support replacement for:

* APP_NAME
* REPO_NAME
* ENVIRONMENT
* API_BRANCH
* UI_BRANCH

---

# General Workflow Rules

## Mandatory Rules

### DO

* Use Pull Requests for all onboarding changes
* Use temporary onboarding branches
* Use template-based file generation
* Support:

  * API-only applications
  * API + UI applications

### DO NOT

* Never push directly to target branches
* Never merge automatically
* Never delete files outside CD onboarding job

---

# Git Configuration

Each job must configure git user:

```bash
git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"
```

---

# Authentication

Use GitHub token:

```yaml
env:
  GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Use GitHub CLI for PR creation.

---

# Job 1 — Application CI Onboard

## Purpose

Setup CI pipeline files for application repositories.

---

## Job Name

```yaml
application-ci-onboard
```

---

## Branch Logic

### API onboarding branch

Create temporary branch:

```text
onboard/api/<AppName>
```

from:

```text
APIBranchName
```

### UI onboarding branch

Create temporary branch:

```text
onboard/ui/<AppName>
```

from:

```text
UIBranchName
```

Only create UI onboarding if `UIBranchName` is provided.

---

## Steps

### 1. Checkout repository

Clone target application repository.

### 2. Create onboarding branch

```bash
git checkout -b onboard/api/${APP_NAME}
```

### 3. Copy CI template files

Copy:

```text
template-files/app-ci/*
```

into repository root.

---

### 4. Replace placeholders

Get __RUBY_VERSION__ values from .ruby-version file in template directory.

Perform placeholder substitution.

---

### 5. Commit changes

Example:

```bash
git add .
git commit -m "feat(ci): onboard ${APP_NAME} CI pipeline"
```

---

### 6. Push onboarding branch

Push only onboarding branch.

---

### 7. Raise PR

Create PR against:

```text
APITargetBranchName
```

Example title:

```text
feat(ci): onboard <AppName> API pipeline
```

PR body should include:

* onboarding type
* environment
* generated files

---

# UI Onboarding Logic

If UI inputs exist:

* Repeat same onboarding flow
* Use UI branches
* Create separate PR

Target:

```text
UITargetBranchName
```

---

# Job 2 — Application CD Onboard

## Purpose

Setup CD deployment pipeline and infrastructure manifests.

---

## Job Name

```yaml
application-cd-onboard
```

Depends on:

```yaml
needs:
  - application-ci-onboard
```

---

# Environment Branch Rules

Use environment branch:

```text
dev-i
qa-i
stage-i
prod-i
```

---

# Environment Branch Creation Logic

If environment branch does not exist:

Create it from default branch.

Example:

```bash
git checkout -b ${ENVIRONMENT}
git push origin ${ENVIRONMENT}
```

---

# Onboarding Branch Naming

Must strictly follow:

```text
<environment>-on-board
```

Example:

```text
dev-i-on-board
```

---

# CD Onboarding Steps

## 1. Checkout environment branch

---

## 2. Create onboarding branch

```bash
git checkout -b ${ENVIRONMENT}-on-board
```

---

## 3. Remove existing files

Preview before deleting:

```bash
ls -la
```

Only in this job.

Allowed to remove files:

Example:

```bash
rm -rf ./* ./.*
```

---

## 4. Copy CD template files

Copy:

```text
template-files/app-cd/*
```

---

## 5. Replace placeholders

Perform dynamic replacements.

Preview after replacement:

```bash
ls -la
```
---

## 6. Commit changes

Example:

```bash
git commit -m "feat(cd): onboard ${APP_NAME} ${ENVIRONMENT} deployment"
```

---

## 7. Raise PR

Target branch:

```text
<environment>
```

Example:

```text
dev-i
```

PR title:

```text
feat(cd): onboard <AppName> into <environment>
```

---

# Job 3 — core_infrastructure Onboard

## Purpose

Provision infrastructure templates and CloudFormation stack definitions.

---

## Job Name

```yaml
infra-cd-onboard
```

Depends on:

```yaml
needs:
  - application-cd-onboard
```

---

# Repository

This job targets:

```text
core_infrastructure
```

repository.

---

# Branch Logic

Use environment branch.

If missing:

Create environment branch first.

---

# Infrastructure Onboarding Branch

Recommended branch:

```text
infra/<environment>/<AppName>
```

Example:

```text
infra/dev-i/orders-api
```

---

# Steps

## 1. Checkout core_infrastructure repository

---

## 2. Create onboarding branch

---

## 3. Copy infrastructure templates

Copy:

```text
template-files/infra-cd/*
```

---

## 4. Replace placeholders

Required replacements:

* APP_NAME
* ENVIRONMENT
* REPO_NAME

---

## 5. Append CloudFormation stack entries

The workflow must append stack references where required.

1. find the right target filename  which is same as template filename.
    
```text
template-files/infra-cd/ecr.yaml  --> core_infrastructure/iac/aws/us-east-1/ecr.yaml
```
2. Append at end of file in target file without overwriting existing entries. Use append logic only.  Never overwrite existing stack entries.

---

## 6. Commit changes

Example:

```bash
git commit -m "feat(infra): onboard ${APP_NAME} infrastructure"
```

---

## 7. Raise PR

Target branch:

```text
<environment>
```

PR title:

```text
feat(infra): onboard <AppName> infrastructure
```

---

# Required GitHub Actions

Use official actions only where possible.

Recommended:

```yaml
uses: actions/checkout@v4
```

---

# Required CLI Tools

Workflow may use:

* git
* gh
* sed
* rsync
* find

Avoid installing unnecessary dependencies.

---

# Error Handling

Workflow must fail if:

* invalid environment supplied
* template directory missing
* branch creation fails
* PR creation fails

Use:

```bash
set -euo pipefail
```

for all shell scripts.

---

# PR Creation Standards

Every PR must contain:

## Title

Conventional commit format:

```text
feat(ci):
feat(cd):
feat(infra):
```

## Body

Include:

* application name
* environment
* onboarding type
* generated resources
* validation checklist

---

# Support Matrix

| Scenario                    | Supported |
| --------------------------- | --------- |
| API only                    | Yes       |
| API + UI                    | Yes       |
| Existing environment branch | Yes       |
| Missing environment branch  | Yes       |
| Existing onboarding branch  | No        |
| Direct push to target       | No        |

---

# Security Guidelines

Never:

* expose secrets
* hardcode tokens
* disable branch protection
* auto merge PRs

Use:

```yaml
permissions:
  contents: write
  pull-requests: write
```

only.

---

# Expected Deliverable

Generate a production-ready GitHub Actions workflow that:

* is modular
* uses reusable shell functions where possible
* contains clear comments
* validates inputs
* supports onboarding automation end-to-end
* creates PRs instead of direct merges
* supports CI, CD, and infrastructure onboarding flows independently
