# Frontend Final Quality Remediation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the validated frontend authentication, endpoint, virtualization, accessibility, styling, measurement, and iOS plugin findings with minimal existing-pattern changes.

**Architecture:** Keep session ownership in the shared GraphQL client, UI state in existing auth reset handlers, and native controls on the current shared Button/Unistyles primitives. Test real LegendList behavior through the existing layout helper. Delete dead measurement tooling rather than rebuilding stale telemetry-by-source parsing.

**Tech Stack:** React Native 0.86, Expo 57, LegendList, Unistyles, TanStack Query, Jest/RNTL, Next/Vitest.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- No package additions, inline RN style objects, React Native `StyleSheet`, or speculative abstractions.
- Write each behavioral test first and observe the expected failure.
- Preserve existing test IDs and Maestro-visible labels.
- Treat logout server revocation as best effort but always complete local invalidation.
- Keep finite forms/filter bars on ScrollView; use real LegendList only for unbounded collections.

---

### Task 1: Revoke FO refresh credentials and always invalidate web sessions

**Files:**
- Modify: `dadamjang-fe/packages/graphql-client/src/graphql-client.ts`
- Modify: `dadamjang-fe/packages/graphql-client/src/index.ts`
- Modify: FO GraphQL client unit tests
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/hooks.ts`
- Modify: BO/Partner `src/shared/auth/session.ts`
- Modify: BO/Partner `tests/unit/session-invalidation.test.ts`

**Interfaces:**
- Produces: `logoutAuthSession(): Promise<boolean>`; `true` means backend revocation succeeded, while both outcomes locally reset the captured session.

- [ ] **Step 1: Write RED shared-client tests**

Using the real test transport/storage, assert logout sends `mutation Logout { logout }` with the refresh bearer token, never attempts access-token refresh, removes persisted/session tokens on success, and still removes them when transport fails.

- [ ] **Step 2: Write RED web tests**

Reject `requestGraphQl` and assert origin notification, storage broadcast, provider cache clearing, and login routing still occur.

- [ ] **Step 3: Verify RED**

Run focused FO GraphQL and BO/Partner session tests. Expected: no FO logout request and no web invalidation after rejection.

- [ ] **Step 4: Implement minimal logout behavior**

Add one shared-client method that captures the refresh token, issues the logout request directly with `Authorization: Bearer <refreshToken>`, maps success to `true` and request failure to `false`, and awaits local reset in all cases. Make `useSignOut` call it. Put both web `invalidateSession()` calls in `finally` while preserving the original result/error.

- [ ] **Step 5: Verify GREEN and commit**

Commit `fix(auth): revoke and invalidate logout sessions`.

### Task 2: Require an explicit correct FO API endpoint

**Files:**
- Modify: `dadamjang-fe/packages/graphql-client/src/graphql-client.ts`
- Modify: GraphQL client unit tests
- Modify: `dadamjang-fe/apps/dadamjang-fo/.env.example`
- Modify: `dadamjang-fe/apps/dadamjang-fo/README.md`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/setup.ts` only if default-client tests need an explicit test URL

- [ ] **Step 1: Write RED resolver tests**

Test a real exported/lazy endpoint resolver with a literal valid URL and with missing/blank input. Missing/blank must throw `EXPO_PUBLIC_API_URL is required`; custom clients with `url` must remain independent of the environment.

- [ ] **Step 2: Verify RED**

Expected: missing input silently resolves to port 3000.

- [ ] **Step 3: Remove the fallback**

Resolve `process.env.EXPO_PUBLIC_API_URL` lazily when the default client creates a request. Set test setup explicitly to `http://127.0.0.1:5500/graphql`. Document iOS simulator/local as `http://localhost:5500/graphql`, Android emulator as `http://10.0.2.2:5500/graphql`, and physical devices as the host LAN address.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(fo): require explicit GraphQL endpoint`.

### Task 3: Test real LegendList virtualization and image recycling

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/list-performance.test.tsx`
- Reuse: `dadamjang-fe/apps/dadamjang-fo/__tests__/helpers/layout-legend-list.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/style/components/style-post-detail.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/recycled-product-images.test.tsx`

- [ ] **Step 1: Remove the LegendList implementation mock and write RED behavior**

Keep transport/layout wrappers mocked only where external, render the actual list, call `layoutLegendList("<literal label>")`, and assert a large dataset does not eagerly mount every item while real end-reached pagination is driven through list events.

- [ ] **Step 2: Verify RED**

Expected: the old eager View mock assumption or missing real layout contract fails.

- [ ] **Step 3: Write gallery recycling RED test**

Give the fixture a non-empty `imageUrls` list and require `recyclingKey="style-1:<index>:<url>"` on each pager image.

- [ ] **Step 4: Implement the stable key**

Use post ID, index, and URL in the actual `expo-image` item. No list abstraction is added.

- [ ] **Step 5: Verify GREEN and commit**

Commit `test(fo): exercise real list recycling` and `fix(fo): key recycled gallery images`.

### Task 4: Make core native commerce actions accessible

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/cart.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/orders.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/product/[product-id].tsx`
- Modify: focused FO integration tests

- [ ] **Step 1: Write RED role/name/state assertions**

Assert retry controls are buttons named `다시 시도`; cart decrement/increment/remove have explicit action labels; checkout is a button exposing disabled state; order rows are buttons named by order number; and brand follow is a button exposing selected state.

- [ ] **Step 2: Verify RED**

Expected: raw Pressables are missing roles/labels/states.

- [ ] **Step 3: Reuse the shared Button or add exact native props**

Prefer `Button variant="bare"` for text/icon actions. Preserve existing styles/test IDs and pass `accessibilityState` explicitly where selected/disabled meaning exists.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(fo): expose commerce action semantics`.

### Task 5: Enforce Unistyles and exact iOS build-plugin behavior

**Files:**
- Modify: two `dadamjang-fe/packages/mobile/src/action-button-group/*.ios.tsx` files
- Modify: `dadamjang-fe/apps/dadamjang-fo/plugins/with-ios-build-settings.cjs`
- Delete: `dadamjang-fe/apps/dadamjang-fo/plugins/with-ios-build-settings.ts` if no runtime/build consumer exists
- Create/modify: focused config-plugin test that imports the actual CJS runtime artifact
- Modify: `dadamjang-fe/scripts/verify-web-release-policy.mjs`

- [ ] **Step 1: Write RED policy and fixture tests**

The production-source policy must reject `StyleSheet` imported from `react-native`. A PBX fixture containing a normal Sentry shell phase must gain exactly one `alwaysOutOfDate = 1;` and remove only duplicate `"-lc++"` flags through the actual CJS plugin helper/transform.

- [ ] **Step 2: Verify RED**

Expected: two styling hits and no match from the loaded CJS regex.

- [ ] **Step 3: Implement minimal fixes**

Import `View` from React Native and `StyleSheet` from Unistyles. Correct the CJS character class. Export only the smallest pure transform needed by the fixture if the Expo mod wrapper cannot be invoked directly. Remove the unused TypeScript twin instead of maintaining two implementations.

- [ ] **Step 4: Verify GREEN and commit**

Commit `fix(mobile): enforce native styling and plugin contracts`.

### Task 6: Remove dead source-measurement tooling and align docs

**Files:**
- Delete: `dadamjang-fe/scripts/measure-fo-problems.mjs`
- Delete: `dadamjang-fe/docs/measurements/fo-problems.md`
- Modify: `dadamjang-fe/package.json`
- Modify: `dadamjang-fe/README.md`

- [ ] **Step 1: Confirm no runtime/CI consumer**

Search all tracked files and verify the package script, README link, and measurement document are the only references.

- [ ] **Step 2: Delete the obsolete command and claims**

Do not rewrite source-grep telemetry around moved files; current performance correctness is covered by list, checkout, build, and runtime tests.

- [ ] **Step 3: Verify and commit**

Run frozen install, format check, policy, and diff check. Commit `chore(fo): remove stale measurement tooling`.

### Task 7: Full frontend verification and review

**Files:** none.

- [ ] Run clean frozen install, format, release policy, full audit, dedupe, all app/package typechecks, lint, BO/Partner FSD/unit/build/E2E, FO 180+ Jest tests, Android tests, autolinking/config, and Android/iOS exports.
- [ ] Parse all Maestro YAML files.
- [ ] Independently re-review P2-1 through P2-6 and P3-1, P3-2, P3-4; P1/P3 E2E fixes are reviewed under the mobile E2E plan.
