# Backend Final Audit Remediation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the nine validated backend release findings without adding a new role schema, payment lifecycle, scheduler, or service decomposition.

**Architecture:** Reuse the existing admission limiter, transaction helpers, PostgreSQL advisory locks, and 32-bit GraphQL/database money boundary. Centralize only the buyer-capability rule and shared cart/money limits because multiple guards/services consume them. Keep cleanup request-driven and bounded; no daemon or new dependency is introduced.

**Tech Stack:** NestJS, TypeScript strict mode, GraphQL, PostgreSQL, Jest unit/integration tests.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- Tests must fail for the reported behavior before production edits.
- Use arrow functions and no source comments.
- Keep checkout non-reserving; update stale documentation instead of inventing payment settlement.
- `PARTNER` gains buyer capability; `ADMIN` does not.
- Keep GraphQL `Int` and PostgreSQL integer money storage, so every persisted/returned total must be at most `2_147_483_647`.
- Add no package and do not mechanically split `AdminService`.

---

### Task 1: Bound anonymous durable auth sessions

**Files:**
- Modify: `dadamjang-be/src/modules/fo-auth/kakao-flow.service.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/kakao-flow.repository.ts`
- Modify: `dadamjang-be/src/modules/identity-verification/identity-verification.service.ts`
- Modify: `dadamjang-be/src/modules/identity-verification/identity-verification.repository.ts`
- Modify: `dadamjang-be/src/modules/admission/admission-limiter.ts`
- Test: focused service specs and `dadamjang-be/test/fo-auth.integration-spec.ts`

**Interfaces:**
- Consumes: existing IP/device admission keys and flow expiry/status columns.
- Produces: admitted starts, at most one reusable current flow for the same device/purpose, and bounded deletion of expired/consumed rows before insertion.

- [ ] **Step 1: Write RED tests**

Add tests proving repeated same-device starts do not retain one row per request, a rejected admission performs no insert, and a cleanup call deletes no more than a literal batch size while removing expired/consumed rows.

- [ ] **Step 2: Verify RED**

Run the focused Kakao/identity unit and integration specs. Expected: duplicate row counts and missing limiter calls fail.

- [ ] **Step 3: Implement the minimum shared behavior**

Inject the existing `AdmissionLimiter`, assert both origin and device scopes before persistence, reuse a still-valid current flow where provider semantics permit, and run one repository-side bounded cleanup statement before a new insert. Add only the two concrete admission event names required by these starts.

- [ ] **Step 4: Verify GREEN and commit**

Run focused tests, then commit `fix(auth): bound anonymous verification sessions`.

### Task 2: Preserve buyer capability after partner approval

**Files:**
- Modify/Create: the smallest existing auth role helper location under `dadamjang-be/src/modules/auth/`
- Modify: `dadamjang-be/src/guards/roles.guard.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.service.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.service.ts`
- Test: auth unit specs and `dadamjang-be/test/admin.integration-spec.ts`

**Interfaces:**
- Produces: `hasBuyerCapability(role)` returning true only for `USER` and `PARTNER`.
- Consumes: FO portal/sign-in checks and `@Roles(USER)` buyer resolvers.

- [ ] **Step 1: Write RED tests**

Add one approval → refresh → cart/order access regression and one unit table proving `USER` and `PARTNER` satisfy buyer capability while `ADMIN` does not; preserve a partner-only guard test.

- [ ] **Step 2: Verify RED**

Expected: refreshed `PARTNER` is forbidden from buyer operations and FO re-sign-in.

- [ ] **Step 3: Implement the central capability**

Use the helper in FO portal/sign-in validation and make a requested `USER` role in `RolesGuard` mean buyer capability. Leave `@Roles(PARTNER)` and `@Roles(ADMIN)` exact.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(auth): preserve partner buyer capability`.

### Task 3: Enforce serializable cart totals and cart cardinality

**Files:**
- Modify/Create: shared cart limit/money invariant near `dadamjang-be/src/modules/cart/`
- Modify: `dadamjang-be/src/modules/cart/cart.service.ts`
- Modify: `dadamjang-be/src/modules/order/order.service.ts`
- Test: cart unit specs and order/cart integration specs

**Interfaces:**
- Produces: `MAX_GRAPHQL_MONEY = 2_147_483_647`, `MAX_CART_ITEMS = 100`, checked line and aggregate totals.

- [ ] **Step 1: Write RED boundary tests**

Prove price `2_147_483_647` × quantity `2` is rejected before persistence, aggregate overflow is rejected, exactly 100 distinct SKUs remain allowed, the 101st is rejected, and a rejected upsert leaves a readable unchanged cart.

- [ ] **Step 2: Verify RED**

Expected: overflow persists a poisoned row and 101 items are accepted.

- [ ] **Step 3: Implement under the existing cart lock**

For a new SKU, count/enforce `MAX_CART_ITEMS`; validate multiplication with division-safe integer checks and accumulate with `Number.isSafeInteger` plus the GraphQL/database maximum. Recheck current prices and the same aggregate invariant during checkout before order insert.

- [ ] **Step 4: Bound legacy reads**

Read at most 101 rows, reject an oversized legacy cart, and avoid silently returning an incomplete checkout/cart.

- [ ] **Step 5: Verify GREEN and commit**

Commit `fix(cart): enforce bounded serializable totals`.

### Task 4: Reject weak production peppers at startup

**Files:**
- Modify: `dadamjang-be/src/modules/app.module.ts`
- Modify: `dadamjang-be/src/modules/app.module.spec.ts`
- Modify: `dadamjang-be/.env.example`

- [ ] **Step 1: Write RED table cases**

Production config must reject missing, `replace-me`, short, and equal `EMAIL_CODE_PEPPER`/`IDENTITY_CI_PEPPER`; a fixture with distinct high-entropy values must pass.

- [ ] **Step 2: Verify RED**

Expected: current validator accepts the invalid cases.

- [ ] **Step 3: Extend the canonical validator**

Reuse the JWT secret validation shape, require both peppers before Nest initialization, and replace public example values with explicit non-secret generation instructions/placeholders that production validation rejects.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(config): reject weak production peppers`.

### Task 5: Serialize opposite style-like and category hierarchy mutations

**Files:**
- Modify: `dadamjang-be/src/modules/style-posts/style-posts.service.ts`
- Modify: the style-post repository transaction boundary if required
- Modify: `dadamjang-be/src/modules/admin/admin.service.ts`
- Test: `dadamjang-be/test/graphql.integration-spec.ts`
- Test: `dadamjang-be/test/admin.integration-spec.ts`

- [ ] **Step 1: Write barrier-controlled RED tests**

One test delays an earlier like while a later unlike begins and requires the final state to be unliked. One test concurrently reparents roots A→B and B→A and requires exactly one failure with an acyclic tree.

- [ ] **Step 2: Verify RED**

Expected: later unlike is lost and both category updates can commit.

- [ ] **Step 3: Add transaction-scoped advisory locks**

Acquire one stable lock key for `(stylePostId,userId)` before either like direction reads/writes. Acquire one hierarchy-wide category lock before cycle validation/update. Keep all reads and writes inside their existing transactions.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(data): serialize mutable relationship state`.

### Task 6: Bound public metadata and align operational documentation

**Files:**
- Modify: `dadamjang-be/src/modules/catalog/catalog.service.ts`
- Modify: relevant catalog repository query methods
- Test: catalog unit/integration specs
- Modify: `dadamjang-be/README.md`

- [ ] **Step 1: Write RED metadata tests**

Seed 101 active metadata rows and require the public legacy arrays to return no more than the documented 100-row ceiling through database-side limits.

- [ ] **Step 2: Verify RED**

Expected: all 101 rows are returned.

- [ ] **Step 3: Apply SQL limits**

Reuse the repository's existing 100-row legacy ceiling and deterministic ordering for categories, brands, colors, and sizes; do not add connection types in this compatibility remediation.

- [ ] **Step 4: Correct docs**

Document checkout as a non-reserving availability snapshot and Kakao callbacks as `flowId` plus an opaque, hashed-at-rest, device-bound, one-time `callbackToken`. Remove stock-decrement/oversell claims.

- [ ] **Step 5: Verify GREEN and commit**

Commit `fix(catalog): bound public metadata contracts` and `docs(auth): align runtime contracts` if the changes are kept as separate review units.

### Task 7: Full backend verification and review

**Files:** none.

- [ ] Run frozen install, TypeScript, lint, build, format check, diff check, and full audit.
- [ ] Start the disposable PostgreSQL service; run 32+ unit suites and all integration suites; run production migration smoke; always stop the test database and volume.
- [ ] Independently review every prior P2/P3 reproduction plus the three updated rulings.
