# Mobile E2E Contract Remediation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the parked AWS E2E API lifecycle, GraphQL URL, EAS version, and Maestro variables a deterministic release contract shared by infrastructure and both mobile workflows.

**Architecture:** Terraform remains the producer of exact environment values and a scale-to-zero ECS service. Each FE workflow gets a protected prepare job that assumes the E2E OIDC role, scales and waits for the API, runs the existing one-off reset task, then fans out to platform tests; one always-run cleanup job scales the service back to zero. Existing release-policy and Maestro tests become executable consumer contract gates.

**Tech Stack:** Terraform test, GitHub Actions, AWS CLI/ECS, Expo/EAS, Maestro, Jest, Node.js.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- Preserve `desired_count = 0` and `ignore_changes = [desired_count]`; CI owns temporary scale-up.
- Use `mobile-e2e` as the single concurrency group across smoke and full workflows.
- Keep fork pull requests unable to receive environment/OIDC credentials.
- Pin every third-party action by the already-approved immutable SHA.
- Never add an HTTP/GraphQL reset endpoint; use `pnpm e2e:reset` in a one-off ECS task.
- Keep Terraform apply, AWS mutation, EAS build, and secret mutation out of local verification.

---

### Task 1: Publish the exact mobile E2E endpoint and region

**Files:**
- Modify: `dadamjang-infra/terraform/e2e/outputs.tf`
- Modify: `dadamjang-infra/terraform/e2e/contracts.tftest.hcl`
- Modify: `dadamjang-infra/terraform/e2e/README.md`

**Interfaces:**
- Consumes: `var.api_hostname`, `var.aws_region`.
- Produces: `e2e_api_url = https://<host>/graphql`, `e2e_aws_region`, and documented FE variables `E2E_API_URL`, `E2E_AWS_REGION`.

- [ ] **Step 1: Write the failing Terraform assertions**

Add literal assertions inside `release_contracts`:

```hcl
assert {
  condition     = output.e2e_api_url == "https://api.e2e.example.test/graphql"
  error_message = "The mobile API URL must target the backend GraphQL route."
}

assert {
  condition     = output.e2e_aws_region == var.aws_region
  error_message = "The mobile workflow region must match the Terraform provider region."
}
```

- [ ] **Step 2: Verify RED**

Run: `terraform -chdir=dadamjang-infra/terraform/e2e test`

Expected: FAIL because `e2e_api_url` lacks `/graphql` and `e2e_aws_region` does not exist.

- [ ] **Step 3: Implement the outputs and documentation**

Set the URL output to `"https://${var.api_hostname}/graphql"`, add `e2e_aws_region = var.aws_region`, and add `E2E_AWS_REGION` to the README contract table.

- [ ] **Step 4: Verify GREEN**

Run: `terraform -chdir=dadamjang-infra/terraform/e2e fmt -check`

Run: `terraform -chdir=dadamjang-infra/terraform/e2e validate`

Run: `terraform -chdir=dadamjang-infra/terraform/e2e test`

Expected: all commands exit 0 and the E2E test remains 1/1.

- [ ] **Step 5: Commit**

Commit: `fix(e2e): publish exact mobile API contract`

### Task 2: Make workflow lifecycle and variables executable contracts

**Files:**
- Modify: `dadamjang-fe/scripts/verify-web-release-policy.mjs`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/maestro-contract.test.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/eas.json`

**Interfaces:**
- Consumes: both mobile workflow YAML files and `ios-full.yaml`.
- Produces: failures for missing OIDC setup, shared concurrency, scale/wait/reset/cleanup, exact EAS 23.0.0, or mismatched Maestro environment variables.

- [ ] **Step 1: Write failing release-policy checks**

For each mobile workflow, require the approved AWS credentials action SHA, `group: mobile-e2e`, `id-token: write`, `vars.E2E_AWS_REGION`, `vars.E2E_AWS_ROLE_ARN`, ECS scale-to-one, service stability wait, `aws ecs run-task`, stopped-task exit-code validation, and an `if: always()` scale-to-zero cleanup. Parse `eas.json` and require `cli.version === "23.0.0"`.

- [ ] **Step 2: Extend the Maestro variable test**

Read `../../../.github/workflows/mobile-e2e-full.yml`, derive the literal variables used by `ios-full.yaml`, and assert the workflow supplies `E2E_USER_EMAIL`, `E2E_USER_PASSWORD`, `E2E_PRODUCT_ID`, and `E2E_SKU_ID` with no `E2E_USER_ID`.

- [ ] **Step 3: Verify RED**

Run: `pnpm --dir dadamjang-fe verify:web-release-policy`

Run: `pnpm --dir dadamjang-fe/apps/dadamjang-fo test:unit -- --runTestsByPath __tests__/unit/maestro-contract.test.ts`

Expected: release policy fails on the absent lifecycle/EAS exact pin; Jest fails on the missing email and unused user ID.

- [ ] **Step 4: Pin the app contract**

Change `apps/dadamjang-fo/eas.json` to:

```json
"cli": {
  "version": "23.0.0",
  "appVersionSource": "remote"
}
```

- [ ] **Step 5: Leave workflow tests RED for Task 3**

Run both focused commands again. The EAS assertion should pass while lifecycle and Maestro workflow assertions still fail.

### Task 3: Implement prepare, platform, and always-run cleanup jobs

**Files:**
- Modify: `dadamjang-fe/.github/workflows/mobile-e2e-smoke.yml`
- Modify: `dadamjang-fe/.github/workflows/mobile-e2e-full.yml`

**Interfaces:**
- Consumes: `vars.E2E_AWS_ROLE_ARN`, `vars.E2E_AWS_REGION`, `vars.AWS_ECS_CLUSTER`, `vars.AWS_ECS_SERVICE`, `vars.AWS_ECS_TASK_DEFINITION`, `vars.AWS_PRIVATE_SUBNET_IDS`, `vars.AWS_API_SECURITY_GROUP_ID`, `vars.E2E_API_URL`.
- Produces: reset E2E fixtures before Maestro and desired count zero after every success/failure/cancellation path.

- [ ] **Step 1: Add protected prepare jobs**

Use `aws-actions/configure-aws-credentials@7474bc4690e29a8392af63c5b98e7449536d5c3a`, job-level `permissions: { contents: read, id-token: write }`, `environment: mobile-e2e`, the trusted same-repository condition for PR smoke, and the README's exact scale/wait/run-task/wait/exit-code commands.

- [ ] **Step 2: Serialize all mobile runs**

Set both workflow-level concurrency blocks to:

```yaml
concurrency:
  group: mobile-e2e
  cancel-in-progress: true
```

- [ ] **Step 3: Gate platform jobs on prepare**

Add `needs: prepare-e2e` to iOS/Android smoke and iOS full. Keep `EXPO_PUBLIC_API_URL: ${{ vars.E2E_API_URL }}` unchanged because Task 1 now supplies the exact GraphQL URL.

- [ ] **Step 4: Correct full-flow credentials**

Replace `E2E_USER_ID` with:

```yaml
E2E_USER_EMAIL: ${{ secrets.E2E_USER_EMAIL }}
```

- [ ] **Step 5: Add always-run cleanup jobs**

For smoke, depend on prepare, iOS, and Android; for full, depend on prepare and iOS. Configure OIDC again and run only the exact scale-to-zero command under `if: always()` plus the trusted-context condition where applicable.

- [ ] **Step 6: Verify GREEN**

Run: `pnpm --dir dadamjang-fe verify:web-release-policy`

Run: `pnpm --dir dadamjang-fe/apps/dadamjang-fo test:unit -- --runTestsByPath __tests__/unit/maestro-contract.test.ts`

Run: `ruby -ryaml -e 'Dir["dadamjang-fe/.github/workflows/mobile-e2e-*.yml"].each { |file| YAML.load_stream(File.read(file), aliases: true) }'`

Expected: all commands exit 0.

- [ ] **Step 7: Commit**

Commit: `fix(e2e): own mobile API lifecycle`

### Task 4: Integrated contract verification

**Files:** none.

- [ ] **Step 1: Run FE gates**

Run: `pnpm --dir dadamjang-fe format:check`

Run: `pnpm --dir dadamjang-fe verify:web-release-policy`

Run: `pnpm --dir dadamjang-fe fo:typecheck`

Run: `pnpm --dir dadamjang-fe fo:lint`

Run: `pnpm --dir dadamjang-fe fo:test`

- [ ] **Step 2: Run infra gates**

Run: `terraform -chdir=dadamjang-infra fmt -check -recursive`

Run: `terraform -chdir=dadamjang-infra/terraform/e2e validate`

Run: `terraform -chdir=dadamjang-infra/terraform/e2e test`

- [ ] **Step 3: Independent review**

Review the FE and infra commits together against P1-1, P1-2, P1-3, and P3-3. No AWS/EAS execution is part of local verification.
