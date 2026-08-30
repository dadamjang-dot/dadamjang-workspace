# Infrastructure Test Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the brittle custom HCL parser while preserving every release-contract guarantee.

**Architecture:** Terraform semantics move to native `terraform test` with mock providers. Ruby remains a direct parser only for YAML and a behavior runner for Docker, shell, checksum, and cross-artifact contracts.

**Tech Stack:** Terraform, HCL tests, Ruby, GitHub Actions, Docker Compose.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- Preserve all 26 existing release-contract behaviors or replace each with an equally strong native assertion.
- No regex or brace-count parser may interpret HCL.
- Do not run Terraform apply or mutate cloud state.
- Commit only files in `dadamjang-infra` and use Conventional Commits.

---

### Task 1: Replace HCL text parsing with native Terraform tests

**Files:**
- Modify: `tests/release-contracts.rb`
- Modify or create: environment `.tftest.hcl` files under `staging/` and `e2e/`
- Modify: `.github/workflows/infra-ci.yml` only if test commands change
- Test: Ruby release contracts, Terraform tests, validates, Compose/Shell/YAML checks

**Interfaces:**
- Consumes: existing Terraform resources, locals, variables, outputs, and mock-provider support.
- Produces: native assertions for task environment, runtime secret keys, IAM scope, immutable image identity, migration ordering, ALB/ECS health, and release provenance.

- [ ] **Step 1: Inventory each HCL-derived Ruby assertion**

Create a one-to-one table in the task report naming the current Ruby check and its native Terraform test replacement. Checks that inspect YAML, Dockerfile, shell execution, or downloaded checksums remain Ruby and must be identified separately.

- [ ] **Step 2: Add native tests and verify RED where coverage is absent**

Write output/resource assertions using Terraform mock providers. Introduce one deliberate local mutation at a time to prove each new assertion fails, then restore it before implementation continues.

- [ ] **Step 3: Remove the HCL parser and migrated checks**

Delete `active_hcl`, `hcl_block_from`, `hcl_block`, `hcl_arguments`, `task_environment`, `runtime_keys`, and every regex check that interprets HCL. Keep direct YAML parsing and executable behavior checks.

- [ ] **Step 4: Run all infrastructure verification and commit**

Run `terraform fmt -check -recursive`, staging/e2e init-backend=false validate, all `terraform test` files, Ruby release contracts, Docker Compose config, Shell syntax/ShellCheck when available, YAML parse, and `git diff --check`. Commit with `test(infra): replace custom hcl parsing`.
