# Backend Data Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close media durability, email proof atomicity, unbounded data work, SKU admission, and runtime migration defects.

**Architecture:** Keep state transitions atomic in repositories, make external deletion retryable, bound every user-controlled collection before hydration, and reuse existing catalog snapshots/projections rather than adding speculative caching infrastructure.

**Tech Stack:** NestJS, Drizzle ORM, PostgreSQL, GraphQL, Jest, Docker.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- No object that may have been deleted may return to READY or PREPARING.
- Maximum SKU count is exactly 100.
- Existing GraphQL fields remain backward compatible; bound legacy arrays to 100 newest records where a connection migration would break consumers.
- Use arrow functions and no comments.
- Write and observe each regression test failing before implementation.
- Commit only files in `dadamjang-be` and use Conventional Commits.

---

### Task 1: Make data transitions durable and bounded

**Files:**
- Modify: `src/modules/media/media.service.ts`
- Modify: `src/modules/media/media.repository.ts`
- Modify: `src/modules/email/email.service.ts`
- Modify: `src/modules/email/email.repository.ts`
- Modify: `src/modules/catalog/catalog.service.ts`
- Modify: `src/modules/style-posts/style-posts.service.ts`
- Modify: `src/modules/order/order.service.ts`
- Modify: `src/modules/wish/wish.service.ts`
- Modify: `src/modules/comparison/comparison.service.ts`
- Modify: `src/modules/partner/partner.service.ts`
- Modify: relevant repositories and GraphQL types used by these services
- Modify: `src/common/graphql/request-budget.ts`
- Modify: `package.json`
- Modify: `Dockerfile`
- Test: existing media, email, catalog, order, wish, comparison, partner, migration, and integration specs

**Interfaces:**
- Consumes: existing media garbage claim/CAS API, email verification records, catalog price evidence snapshots, GraphQL pagination inputs.
- Produces: retryable media deletion state; atomic idempotent email proof; maximum 100 results for legacy arrays; production-only `migrate:prod` command.

- [ ] **Step 1: Write the failing media deletion durability test**

Model `deleteObject` success followed by `completeGarbage` failure. Assert `releaseGarbage` is never called after deletion starts, a new reference remains rejected, and the next GC pass retries the idempotent delete then completes.

- [ ] **Step 2: Run the media spec and verify RED**

The failure must show the current catch path restores the previous state.

- [ ] **Step 3: Implement a monotonic deletion state**

After an external delete attempt, leave the claim in its deleting/garbage state on any ambiguous or DB failure. Allow stale claims to be reclaimed and complete through the existing CAS path. Do not add a second ledger.

- [ ] **Step 4: Write failing email proof failure/retry tests**

Assert that a proof-insert failure rolls back code consumption and that retry after response loss returns the same still-valid proof instead of consuming a second code.

- [ ] **Step 5: Implement one transactional idempotent email proof operation**

Move compare-and-set verification consumption and proof creation into one repository transaction keyed by verification ID. Preserve hashed-at-rest tokens; return the existing unexpired proof record on an idempotent retry using the current request identity.

- [ ] **Step 6: Write failing SKU boundary tests**

Use literal cases for 0, 1, 100, and 101 SKUs plus overlong code/option, negative or excessive price, and negative or excessive stock. The 101 case must fail before repository calls.

- [ ] **Step 7: Implement SKU and GraphQL variable bounds**

Validate at the service boundary, preserve database constraints, and extend request budgeting only for user-controlled array/cardinality arguments that are currently invisible to AST field counting.

- [ ] **Step 8: Write failing bounded-collection tests**

For orders, wish, comparison, and purchased-style history, make repositories return 101 fixture rows and assert the service/repository requests at most 100 newest records and never builds an unbounded `IN` list. Assert purchased products are deduplicated in SQL or the repository boundary.

- [ ] **Step 9: Bound collection hydration and expensive anonymous sorts**

Push hard limits, ordering, aggregation, and deduplication into SQL. Reuse existing price-evidence snapshots/projections for catalog sort. Apply an explicit resolver/database timeout or admission boundary to anonymous catalog/style ranking. Do not introduce a new cache or materialized view when the existing snapshot table can answer the query.

- [ ] **Step 10: Write and pass production migration smoke checks**

Add `migrate:prod` as `node dist/scripts/migrate.js`. Ensure the runtime image ships `dist/scripts/migrate.js`, production dependencies, migrations, and retired migrations. Build the final stage and invoke the command far enough to prove Node can load it without `ts-node` or source files.

- [ ] **Step 11: Run backend data verification and commit**

Run focused specs, full unit suite, full integration suite, typecheck, lint, build, audit, migration clean install, upgrade, and Docker smoke. Commit with `fix(data): make durable flows bounded and retryable`.
