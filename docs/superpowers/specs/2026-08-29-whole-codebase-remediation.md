# Whole-codebase remediation specification

## Goal

Close every confirmed P1-P3 finding from the 2026-08-29 review without adding speculative features or replacing existing architecture unnecessarily.

## Release requirements

- No confirmed P1 or P2 finding may remain.
- P3 findings must be fixed unless the replacement would add more code or operational risk than the finding; every such ruling must be recorded in the SDD ledger.
- Every behavioral fix follows red-green-refactor and leaves a regression test.
- No new runtime dependency is allowed when the platform, standard library, or an existing dependency covers the behavior.
- Existing arrow-function, no-comment, strict-TypeScript, Unistyles, FSD, and Conventional Commit rules remain binding.
- No push, deployment, publication, Terraform apply, or secret mutation is authorized.

## Cross-repository contracts

### OAuth and identity callback binding

- Both callback flows use the exact field name `callbackToken`.
- `KakaoFlowService.acceptCallback` creates a 32-byte base64url token, persists only `hashToken(callbackToken)`, and returns the plaintext once for the callback redirect.
- `/api/auth/kakao/callback` redirects to the configured app URL with `flowId` and `callbackToken`.
- `CompleteKakaoLoginInput` requires both `flowId` and `callbackToken`; redemption atomically matches flow, initiating device, callback-token hash, status, expiry, and unused state.
- Identity verification success creates a 32-byte base64url callback token, persists only its hash, and redirects with `sessionId`, `status`, and `callbackToken`.
- `completeIdentityVerification` requires `sessionId` and `callbackToken`; redemption atomically matches session, initiating device, callback-token hash, verified state, expiry, and unused state.
- A URL relayed from device A to device B must never let A complete the flow. B receives the callback token through its own deep link and may complete only using its own matching device/session contract.
- Callback tokens are one-time values and never appear in logs or database plaintext.

### Frontend session invalidation

- BO and Partner use a versioned, non-sensitive `localStorage` key plus the native `storage` event for cross-tab invalidation.
- The originating tab is invalidated immediately through the existing custom event; other tabs receive one storage event without rebroadcast loops.
- Logout, explicit unauthenticated responses, and session role mismatch all clear query caches and replace the route with `/login`.
- Session queries refetch on window focus even though ordinary application queries may retain the existing focus policy.

### CI supply chain

- Action pins use these verified commits: `actions/checkout@11d5960a326750d5838078e36cf38b85af677262`, `pnpm/action-setup@b906affcce14559ad1aafd4ab0e942779e9f58b1`, `actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020`, `expo/expo-github-action@c7b66a9c327a43a8fa7c0158e7f30d6040d2481e`, `actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02`, and `ReactiveCircus/android-emulator-runner@a421e43855164a8197daf9d8d40fe71c6996bb0d`.
- EAS CLI is exactly `23.0.0`.
- The removed `mobile-dev-inc/action-maestro` action is replaced by Maestro CLI `2.9.0`, downloaded from the official `cli-2.9.0` GitHub release and verified against SHA-256 `855bb2ce1399d82f4f4a73d84a4d945f70b0d43eb86127e027af82809f63f0bd` before extraction.

## Backend requirements

- Password admission is consumed before user lookup and bcrypt work, with IP plus normalized account and device scopes. BO and Partner use stricter limits than FO.
- Production configuration rejects placeholder, short, or identical access/refresh JWT secrets. Access and refresh tokens carry and validate distinct `tokenUse`, issuer, audience, algorithm, role, and required identifiers.
- Inicis result fetch rejects redirects and retains the existing host, HTTPS, timeout, and response-shape checks.
- Media garbage remains in a retryable deleting state after any external delete attempt; it is never restored to READY or PREPARING after the object may be gone.
- Email verification consumption and proof issuance are one transaction and idempotently return the same valid proof after response loss.
- SKU input has an explicit maximum of 100 and bounded code, option, price, and stock values before database work.
- Anonymous expensive queries are bounded at the resolver and database layers; collection APIs have a hard maximum and do not hydrate unbounded histories.
- The backend runtime image exposes and uses a production migration command that depends only on shipped production artifacts.

## FO mobile requirements

- The callback-token contract above is used by `openAuthSessionAsync` completion calls.
- Home, authenticated My, Search, and Compare render useful existing product/style/account data or navigation; no visible action may be a no-op.
- Cart, orders, product cart mutation, protected likes/follows, and protected queries route unauthenticated users through the existing sanitized `returnTo` flow.
- Cross-platform actions use native tab icon descriptors, Material icons, text, or bundled assets; no Android runtime action depends on `sf:` image sources.
- Style submission cannot be closed or gesture-dismissed while upload/create is in progress.
- A checkout attempt owns one idempotency key across user retry until success or cart mutation invalidates the attempt.
- The iOS full Maestro flow uses `e2e.auth.email.input` and `E2E_USER_EMAIL`.
- Every recycled list image has a stable `recyclingKey`; tests must exercise the real item contract rather than reducing all virtualization behavior to an unbounded `View`.
- Reanimated shared values use `.get()` and `.set()` with React Compiler.
- Style-like mutations are serialized per style post or protected by a latest-revision contract.
- React Native styling uses Unistyles only.

## BO and Partner requirements

- Next Image accepts only origins configured through `DADAMJANG_IMAGE_ORIGINS`; production builds fail when it is empty, while development may add localhost explicitly.
- GraphQL BFF upstream calls combine the incoming request signal with a 10-second deadline and reject bodies larger than 1 MiB without buffering the remainder.
- Partner XHR uploads time out after 60 seconds, abort cleanly, and always release their shared concurrency slot.
- Remote list thumbnails use Next optimization. `blob:` editor previews alone may use `unoptimized`.
- Partner primary text/background combinations meet WCAG AA; Partner Playwright runs axe against authenticated representative pages.
- Product approval status has one canonical shared union including `DRAFT`; BO and Partner do not widen it back to `string`.
- `minimumReleaseAge` is 1440 minutes. Direct dependencies without source or test use are removed.
- One root formatting command checks BO, Partner, FO source, and workspace packages; CI runs it.

## Infrastructure requirements

- The custom regex/brace-count HCL parser is removed.
- HCL semantics are asserted with native `terraform test` and mock providers.
- Ruby release-contract checks remain only for YAML, Dockerfile, shell, checksum, and cross-artifact behavior that Terraform cannot express directly.

## Verification

- Backend: typecheck, lint, build, unit, integration, audit, migration smoke.
- FO: typecheck, lint, unit, integration, Android tests, autolinking, Android and iOS production exports, Maestro YAML validation.
- BO and Partner: typecheck, lint, FSD, unit, build, Playwright, format.
- Infrastructure: `terraform fmt -check -recursive`, staging/e2e validate, `terraform test`, Ruby release contracts, Compose/Shell/YAML checks.
- Whole workspace: frozen install, dependency audit, `git diff --check`, conflict-marker scan, and a final independent thermo-nuclear review.
