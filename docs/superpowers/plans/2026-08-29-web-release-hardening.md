# Web Release Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close BO/Partner session, proxy, image, upload, accessibility, type, dependency, formatting, and CI supply-chain findings.

**Architecture:** Use browser-native cross-tab signaling, bounded streaming fetch, Next Image allowlists, existing Playwright axe support, and exact dependency/action pins. Shared product status remains canonical in the existing domain package.

**Tech Stack:** Next.js, React, TanStack Query, GraphQL Request, Playwright, Vitest, pnpm, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- Use exact CI commits, EAS version, Maestro version, and checksum from the spec.
- Add no runtime package.
- Store no token or user data in localStorage.
- Use arrow functions and no comments.
- Write and observe each regression test failing before implementation.
- Commit only files in `dadamjang-fe` and use Conventional Commits.

---

### Task 1: Harden web runtime and supply chain

**Files:**
- Modify: `.github/workflows/frontend-static.yml`
- Modify: `.github/workflows/mobile-e2e-smoke.yml`
- Modify: `.github/workflows/mobile-e2e-full.yml`
- Modify: `apps/dadamjang-bo/src/_app/providers/app-providers.tsx`
- Modify: `apps/dadamjang-partner/src/_app/providers/app-providers.tsx`
- Modify: BO and Partner shared auth/session modules and shell logout handlers
- Modify: BO and Partner `src/_app/api-routes/graphql.ts`
- Modify: `apps/dadamjang-bo/next.config.ts`
- Modify: `apps/dadamjang-partner/next.config.ts`
- Modify: `apps/dadamjang-partner/src/shared/api/upload-file.ts`
- Modify: `apps/dadamjang-partner/src/_pages/products/ui/products-page.tsx`
- Modify: `apps/dadamjang-partner/src/_pages/product-editor/ui/product-editor-page.tsx`
- Modify: `apps/dadamjang-partner/src/_app/styles/globals.css`
- Modify: `packages/domain/src/product/product-status.ts`
- Modify: BO and Partner API contract types that represent product status
- Modify: `pnpm-workspace.yaml`, root/app `package.json` files, `pnpm-lock.yaml`
- Modify or create: focused Vitest and Playwright tests

**Interfaces:**
- Consumes: native `storage` event, request `AbortSignal`, Web Streams reader, `DADAMJANG_IMAGE_ORIGINS`.
- Produces: versioned session invalidation signal; 10-second/1-MiB proxy boundary; canonical `ProductApprovalStatus` including `DRAFT`; root `format:check`.

- [ ] **Step 1: Write failing multi-tab session tests**

Create two logical window/channel fixtures. Dispatch logout in one, then assert both QueryClients clear once and both routers replace `/login`; assert the receiving storage handler does not write another storage event. Assert session query options use `refetchOnWindowFocus: "always"`.

- [ ] **Step 2: Implement browser-native invalidation**

Add app-local session event helpers with one versioned non-sensitive key. Catch storage access failures. Route explicit logout and unauthenticated GraphQL errors through one origin-tab event. Keep global listeners at provider scope only.

- [ ] **Step 3: Write failing BFF deadline/body-cap tests**

Use a never-resolving upstream and a response stream larger than 1,048,576 bytes. Assert the route returns 504 for deadline/abort, 502 for oversized upstream, cancels the reader, and never calls unbounded `response.text()`.

- [ ] **Step 4: Implement bounded upstream streaming**

Combine the request signal with `AbortSignal.timeout(10_000)`. Read chunks through a Web Streams reader, count bytes before concatenation, cancel on overflow, and map timeout, client abort, network failure, malformed JSON, and upstream status separately. Share the pure bounded-reader implementation only when BO and Partner can import it without creating a package boundary violation.

- [ ] **Step 5: Write failing upload timeout and image tests**

Advance fake timers to 60,000 ms and assert XHR aborts/rejects exactly once and the next queued upload starts. Render product lists and assert remote URLs are optimized; editor `blob:` previews alone remain unoptimized.

- [ ] **Step 6: Implement upload and image boundaries**

Set XHR timeout to 60 seconds with `ontimeout`, `onabort`, single settlement, and existing queue `finally` release. Remove list `unoptimized`; make editor preview conditional on `blob:`.

- [ ] **Step 7: Write failing Next origin and accessibility tests**

Test parsing `DADAMJANG_IMAGE_ORIGINS` into exact protocol/hostname/port patterns, rejection of wildcard/private origins in production, and localhost only in development. Add Partner axe coverage for authenticated dashboard/products/editor and assert zero serious/critical violations.

- [ ] **Step 8: Implement allowlists and AA colors**

Use exact configured origins and fail production configuration when empty. Darken Partner accent/text colors until normal text is at least 4.5:1 and interactive focus/disabled states remain distinguishable.

- [ ] **Step 9: Consolidate product status types**

Add `DRAFT` to the canonical domain union, import it in Partner and BO contracts, and add compile-time/executable fixtures proving every GraphQL status is accepted while arbitrary strings are rejected.

- [ ] **Step 10: Pin CI and dependency policy**

Replace every mutable Action in FE workflows with the exact commits in the spec. Replace the removed Maestro action with checksum-verified CLI installation and exact EAS 23.0.0. Set `minimumReleaseAge: 1440`, retain only necessary exclusions, remove unused `@testing-library/user-event` and BO `@testing-library/react`, and keep Partner axe/domain because the task now uses them.

- [ ] **Step 11: Add one formatting gate**

Create root `format:check` covering BO, Partner, FO source, and workspace packages without generated `.next`, coverage, build, or native output. Run it in `frontend-static.yml` and format only files covered by the gate.

- [ ] **Step 12: Run web verification and commit**

Run frozen install, BO/Partner typecheck, lint, FSD, unit, build, Playwright, root format, workflow YAML parse/action-reference checks, and audit. Commit with `fix(web): harden sessions proxies and supply chain`.
