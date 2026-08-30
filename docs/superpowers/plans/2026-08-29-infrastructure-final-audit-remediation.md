# Infrastructure Final Audit Remediation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore real remote state, deterministic plan inputs, complete release provenance, restart-safe image retention, least-privilege workflows, and native E2E IAM coverage without applying infrastructure.

**Architecture:** Keep GitHub Actions as the local Terraform execution environment and HCP remote backend as state-only storage. Make the existing workflow plan-only because no safe post-plan approval boundary currently exists. Generate source provenance from the full Terraform root and retain every tagged release image automatically.

**Tech Stack:** Terraform 1.15.7, HCP remote backend, AWS ECS/ECR/IAM, GitHub Actions, Ruby contract tests, Bash.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- Do not run Terraform apply, AWS mutation, deployment, publication, or secret changes.
- Preserve staging/e2e state isolation and current immutable action SHAs.
- Use native Terraform assertions for HCL semantics and Ruby only for workflow/cross-artifact contracts.
- Keep workflows fail-closed and use arrow-free language rules only where TypeScript applies.

---

### Task 1: Select the real remote backend and deterministic plan inputs

**Files:**
- Modify: `dadamjang-infra/terraform/staging/terraform.tf`
- Modify: `dadamjang-infra/terraform/e2e/terraform.tf`
- Modify: `dadamjang-infra/.github/workflows/terraform-apply.yml`
- Modify: `dadamjang-infra/tests/release-contracts.rb`
- Modify: `dadamjang-infra/README.md`

- [ ] **Step 1: Write RED contract checks**

Require both roots to initialize without the `Missing backend configuration` warning when supplied a temporary partial remote backend config. Require the plan workflow to expose non-empty protected `TF_VAR_acm_certificate_arn` and `TF_VAR_api_hostname` and contain no `terraform apply` command or apply dispatch mode.

- [ ] **Step 2: Verify RED**

Run Ruby contracts and an isolated `TF_DATA_DIR` init. Expected: missing backend warning and absent variable/plan-only checks fail.

- [ ] **Step 3: Implement backend declarations**

Add an empty `backend "remote" {}` inside both existing `terraform` blocks. Keep organization/workspace values in the current protected partial configuration.

- [ ] **Step 4: Make workflow plan-only**

Remove the apply input/step. Export `TF_VAR_acm_certificate_arn: ${{ vars.ACM_CERTIFICATE_ARN }}` and `TF_VAR_api_hostname: ${{ vars.API_HOSTNAME }}` and fail before init when either is blank. Document both variables and local execution with remote state.

- [ ] **Step 5: Verify GREEN and commit**

Run isolated init with `-backend=false` for local validation plus native/Ruby tests. Commit `fix(terraform): restore remote state contract`.

### Task 2: Hash the complete Terraform release root

**Files:**
- Modify: `dadamjang-infra/terraform/staging/outputs.tf`
- Modify: `dadamjang-infra/terraform/staging/contracts.tftest.hcl`
- Modify: `dadamjang-infra/scripts/prepare-ecs-task-definition.sh`
- Modify: `dadamjang-infra/tests/release-contracts.rb`

- [ ] **Step 1: Write RED cross-artifact test**

Discover every `terraform/staging/*.tf` basename and `.terraform.lock.hcl`; assert the output and shell validation cover the exact discovered set. The test must fail when `iam.tf`, `network.tf`, `providers.tf`, or `terraform.tf` is omitted.

- [ ] **Step 2: Verify RED**

Expected: four Terraform files are missing from the current literal map.

- [ ] **Step 3: Replace duplicated allow-lists**

Build the Terraform map from `fileset(path.module, "*.tf")` plus `.terraform.lock.hcl`. In Bash, discover the same local set and compare exact sorted keys and SHA-256 values from state. Remove literal four-file lists.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(release): hash complete terraform source`.

### Task 3: Preserve tagged release images

**Files:**
- Modify: `dadamjang-infra/terraform/staging/application.tf`
- Modify: `dadamjang-infra/terraform/e2e/application.tf`
- Modify: both native contract test files

- [ ] **Step 1: Write RED native assertions**

Decode each ECR lifecycle policy and require every expiration rule to use `tagStatus = "untagged"`; reject `tagStatus = "any"` and count-based expiry of tagged artifacts.

- [ ] **Step 2: Verify RED**

Expected: staging and e2e current `any` policies fail.

- [ ] **Step 3: Implement untagged-only cleanup**

Expire only untagged images after a fixed age. Do not add an automated tagged cleanup until it can exclude all active/rollback task-definition digests.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(registry): retain tagged release images`.

### Task 4: Harden workflow provenance and trigger coverage

**Files:**
- Modify: `dadamjang-infra/.github/workflows/api-deploy.yml`
- Modify: `dadamjang-infra/.github/workflows/infra-ci.yml`
- Modify: `dadamjang-infra/tests/release-contracts.rb`
- Modify: `dadamjang-infra/README.md`

- [ ] **Step 1: Write RED workflow contracts**

Require workflow-level `contents: read` only, deploy-job `id-token: write`, test-job no OIDC, repository-dispatch refs as mandatory full 40-hex SHAs, manual refs handled separately, and `.env.example` in both infra-CI path filters and required-path tests.

- [ ] **Step 2: Verify RED**

Expected: broad OIDC, `main` fallback, and missing path trigger fail.

- [ ] **Step 3: Implement exact permissions/ref resolution/triggers**

Move OIDC permission to deploy, add an early event-specific ref validation step, remove the repository-dispatch fallback to main, and add `.env.example` path coverage.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(ci): harden release provenance inputs`.

### Task 5: Make mobile E2E IAM a native contract

**Files:**
- Modify: `dadamjang-infra/terraform/e2e/iam.tf`
- Modify: `dadamjang-infra/terraform/e2e/contracts.tftest.hcl`

- [ ] **Step 1: Write RED native assertions**

Assert the exact OIDC audience, environment-scoped subject, repository condition, service-only UpdateService/DescribeServices resources, task-family/cluster-constrained RunTask, exact DescribeTasks task ARN, and two-role ECS-only PassRole contract.

- [ ] **Step 2: Verify RED**

Expected: current mocked policy-document outputs provide no inspectable policy semantics.

- [ ] **Step 3: Move exact policy objects into locals**

Define the trust and permission documents as Terraform-native objects/JSON consumed by the IAM resources, following the staging pattern. Keep no duplicated policy implementation.

- [ ] **Step 4: Verify GREEN and commit**

Commit `test(e2e): enforce mobile OIDC policy`.

### Task 6: Full infrastructure verification and review

**Files:** none.

- [ ] Run recursive fmt; isolated init/validate/test for staging and e2e; Ruby 17+ contracts; Compose config; Bash syntax; YAML parse; diff check; conflict/secret scan.
- [ ] Verify the backend warning is absent with a controlled partial remote configuration, without contacting real state.
- [ ] Independently re-review P1, five P2s, four P3s, and the 26/26 parser-replacement inventory.
