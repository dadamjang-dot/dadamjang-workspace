# FO Release Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Finish visible FO routes and close authentication, platform, retry, recycling, compiler, and mutation-race defects.

**Architecture:** Reuse current queries, list primitives, auth return routing, and design-system components. Add no speculative endpoint or navigation layer; stateful safety stays in the feature hook that owns the operation.

**Tech Stack:** Expo Router, React Native, TypeScript, TanStack Query, LegendList, Expo Image, Reanimated, Jest, Maestro.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- Use the exact `callbackToken` backend contract from the spec.
- Use `@legendapp/list/react-native`, Expo Image, and Unistyles.
- Keep small fixed horizontal galleries on FlatList when virtualization replacement adds no value.
- No visible `onPress` may be a no-op.
- Use arrow functions and no comments.
- Write and observe each regression test failing before implementation.
- Commit only files in `dadamjang-fe` and use Conventional Commits.

---

### Task 1: Complete and harden the FO application

**Files:**
- Modify: `apps/dadamjang-fo/src/features/auth/auth-session.ts`
- Modify: `apps/dadamjang-fo/src/features/auth/kakao-api.ts`
- Modify: `apps/dadamjang-fo/src/features/auth/identity-api.ts`
- Modify: `apps/dadamjang-fo/src/app/(tabs)/index.tsx`
- Modify: `apps/dadamjang-fo/src/app/(tabs)/my.tsx`
- Modify: `apps/dadamjang-fo/src/app/(tabs)/shop.tsx`
- Modify: `apps/dadamjang-fo/src/shared/components/search-content/search-content.tsx`
- Modify: `apps/dadamjang-fo/src/app/compare.tsx`
- Modify: `apps/dadamjang-fo/src/app/cart.tsx`
- Modify: `apps/dadamjang-fo/src/app/orders.tsx`
- Modify: `apps/dadamjang-fo/src/app/product/[product-id].tsx`
- Modify: `apps/dadamjang-fo/src/features/cart/hooks.ts`
- Modify: `apps/dadamjang-fo/src/features/style/components/style-composer.tsx`
- Modify: `apps/dadamjang-fo/src/features/style/hooks.ts`
- Modify: `apps/dadamjang-fo/src/features/style/components/style-post-card.tsx`
- Modify: `apps/dadamjang-fo/src/shared/components/product-card/product-card.tsx`
- Modify: `apps/dadamjang-fo/src/features/style/components/style-post-detail.tsx`
- Modify: `apps/dadamjang-fo/src/shared/components/product-layout/product-layout.ios.tsx`
- Modify: `apps/dadamjang-fo/src/shared/components/product-header/product-header.ios.tsx`
- Modify: `apps/dadamjang-fo/src/app/_layout.tsx`
- Modify: `apps/dadamjang-fo/.maestro/ios-full.yaml`
- Modify or create: focused FO unit/integration tests covering each behavior

**Interfaces:**
- Consumes: callback deep-link query `callbackToken`; existing `resolveAuthReturnTo`, product/style/comparison/order/cart hooks, LegendList, platform icon descriptors.
- Produces: `completeKakaoLogin(flowId, callbackToken)` and `completeIdentityVerification(sessionId, callbackToken)`; stable checkout-attempt key lifecycle; per-style-post mutation serialization.

- [ ] **Step 1: Write failing callback-token session tests**

Update auth-session tests so callback URLs without `callbackToken` reject and these calls are required:

```typescript
expect(completeKakaoLogin).toHaveBeenCalledWith("flow-1", "callback-token");
expect(completeIdentityVerification).toHaveBeenCalledWith("identity-session", "callback-token");
```

Use literal callback URLs containing both IDs and `callbackToken`.

- [ ] **Step 2: Verify RED and implement the callback contract**

Run the focused auth-session tests, observe the missing-argument failure, then pass the token through deep-link parsing and GraphQL variables. Never persist callback tokens.

- [ ] **Step 3: Write failing route-completeness tests**

Render Home, authenticated My, Search with a keyword, and Compare with fixtures. Assert each exposes real navigation/data actions: Home routes to style/shop/cart; My shows account ID and routes to orders/cart plus logout; Search renders product results and retry/empty states; Compare renders matching summaries with remove and product-open actions. Assert no registered action has an empty handler.

- [ ] **Step 4: Implement visible routes using existing features**

Reuse current layout, query hooks, cards, and buttons. Correct comparison types to include the product shape already queried. Do not add a new backend endpoint or duplicate Shop filtering logic.

- [ ] **Step 5: Write failing authentication-gate tests**

For unauthenticated Cart, Orders, add-to-cart, follow, wish, and protected comparison actions, assert no protected query/mutation runs and the router receives the existing sanitized auth route with the exact original `returnTo` path.

- [ ] **Step 6: Implement one reusable FO auth-action gate**

Place the smallest reusable hook/helper in `features/auth`, reuse `resolveAuthReturnTo`, and enable protected queries only when authenticated. Preserve loading, offline, and error states instead of treating them as unauthenticated.

- [ ] **Step 7: Write failing platform-action and submit-dismissal tests**

Under Android test conditions, assert each product/style action has a Material/text/bundled icon and no Expo Image source beginning `sf:`. During style submission, assert close, route gesture, picker dismissal, image removal, and form mutation controls are disabled while upload/create is pending.

- [ ] **Step 8: Implement cross-platform actions and submission locking**

Reuse `ActionButtonGroup` icon descriptors where possible. Replace remaining runtime SF images with the existing platform icon abstraction or accessible text. Disable the route gesture for style compose and disable all composer dismissal/mutation paths while submitting.

- [ ] **Step 9: Write failing retry, recycling, compiler, and mutation-race tests**

Assert checkout retries reuse one UUID until success or cart mutation; every recycled product image receives its product ID as `recyclingKey`; shared values are accessed by `.get()`/`.set()`; and a second like toggle for the same style post cannot race the first or roll back newer state.

- [ ] **Step 10: Implement operation-owned state**

Keep the checkout key in the cart-actions hook, reset it on successful checkout or cart-changing mutation, add stable recycling keys, convert Reanimated accessors, and serialize mutations per `stylePostId` with a pending map or latest-revision guard. Avoid global locks.

- [ ] **Step 11: Correct Maestro and styling consistency**

Use `e2e.auth.email.input` and `E2E_USER_EMAIL`. Convert remaining React Native `StyleSheet` imports in touched mobile files/packages to `react-native-unistyles` without inline style objects.

- [ ] **Step 12: Run FO verification and commit**

Run focused tests, all FO tests, Android tests, typecheck, lint, autolinking, Android export, iOS export, and Maestro YAML validation. Commit with `fix(fo): complete routes and harden protected flows`.
