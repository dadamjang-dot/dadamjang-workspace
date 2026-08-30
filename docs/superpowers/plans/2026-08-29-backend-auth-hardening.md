# Backend Auth Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close account-relay, identity-relay, login-abuse, JWT-confusion, and Inicis redirect vulnerabilities.

**Architecture:** Callback completion uses one-time hashed callback tokens delivered only through the callback deep link. Existing database admission control protects password work, while a small auth configuration boundary validates production secrets and token claims.

**Tech Stack:** NestJS, GraphQL, Drizzle ORM, PostgreSQL, Passport JWT, Jest.

**Spec:** `docs/superpowers/specs/2026-08-29-whole-codebase-remediation.md`

## Global Constraints

- Use the exact `callbackToken` cross-repository contract from the spec.
- Add no runtime package.
- Use arrow functions and no comments.
- Write and observe each regression test failing before implementation.
- Commit only files in `dadamjang-be` and use Conventional Commits.

---

### Task 1: Close backend authentication trust boundaries

**Files:**
- Modify: `src/modules/fo-auth/kakao-flow.service.ts`
- Modify: `src/modules/fo-auth/kakao-flow.repository.ts`
- Modify: `src/modules/fo-auth/kakao-flow.resolver.ts`
- Modify: `src/modules/fo-auth/fo-auth.types.ts`
- Modify: `src/guards/kakao.guard.ts`
- Modify: `src/modules/auth/auth.controller.ts`
- Modify: `src/modules/identity-verification/identity-verification.service.ts`
- Modify: `src/modules/identity-verification/identity-verification.repository.ts`
- Modify: `src/modules/identity-verification/identity-verification.controller.ts`
- Modify: `src/modules/identity-verification/identity-verification.resolver.ts`
- Modify: `src/modules/identity-verification/identity-verification.types.ts`
- Modify: `src/modules/identity-verification/inicis-identity.adapter.ts`
- Modify: `src/modules/auth/auth.service.ts`
- Modify: `src/modules/fo-auth/fo-auth.service.ts`
- Modify: `src/modules/auth/auth.module.ts`
- Modify: `src/modules/fo-auth/fo-auth.module.ts`
- Modify: `src/modules/app.module.ts`
- Modify: `src/modules/auth/auth.types.ts`
- Modify: `src/strategies/access-token.strategy.ts`
- Modify: `src/strategies/refresh-token.strategy.ts`
- Modify: `src/modules/database/schema.ts`
- Create: next ordered SQL migration under `migrations/`
- Test: existing auth, Kakao-flow, identity-verification, adapter, migration, and integration specs

**Interfaces:**
- Consumes: `hashToken`, `AdmissionLimiter`, `requestOriginFromRequest`, current device-ID headers and deep-link redirects.
- Produces: required `CompleteKakaoLoginInput.callbackToken`; required `completeIdentityVerification(sessionId, callbackToken)`; JWT payload `tokenUse: "access" | "refresh"`.

- [ ] **Step 1: Write failing relay and one-time-consumption tests**

Add tests that express these literal behaviors:

```typescript
await expect(service.completeLogin(flowId, "device-a", "token-seen-only-by-device-b")).rejects.toThrow();
await expect(service.complete("identity-session", "device-a", "token-seen-only-by-device-b")).rejects.toThrow();
await expect(service.completeLogin(flowId, "device-b", callbackToken)).resolves.toMatchObject({ status: "SIGNED_IN" });
await expect(service.completeLogin(flowId, "device-b", callbackToken)).rejects.toThrow();
```

Controller tests must assert redirects contain `callbackToken` and never contain a hash. Repository integration tests must prove token-hash, device, expiry, status, and consumed predicates are in one update/transaction.

- [ ] **Step 2: Run focused tests and verify RED**

Run the smallest matching Jest specs. The tests must fail because callback-token fields and matching do not exist, not because of fixture or TypeScript errors.

- [ ] **Step 3: Implement callback-token persistence and redemption**

Add nullable callback-token hash columns to Kakao flow and identity session records. Generate tokens with:

```typescript
const callbackToken = randomBytes(32).toString("base64url");
```

Persist only `hashToken(callbackToken)`, redirect the plaintext once, require it in GraphQL completion, and atomically match and consume it. Preserve all existing device, expiry, status, and uniqueness checks.

- [ ] **Step 4: Verify relay tests GREEN and run migration tests**

Run focused unit and integration specs plus migration clean-install and upgrade-path tests.

- [ ] **Step 5: Write failing admission tests before bcrypt work**

Use an ordered fake to prove `AdmissionLimiter.assertAllowed` runs before `findByUserid`, `findByEmail`, or `bcrypt.compare`. Assert separate literal scopes for IP, normalized account, and device; BO/Partner portal limits must be lower than FO limits.

- [ ] **Step 6: Implement login admission at the service boundary**

Pass `RequestOrigin` from resolvers into signin services. Inject the existing Admission module and consume rules before account lookup. Do not leak account existence through messages or limiter scope names.

- [ ] **Step 7: Write failing JWT configuration and token-confusion tests**

Add tests asserting production startup rejects `replace-me`, secrets shorter than 32 bytes, and equal access/refresh secrets. Sign a refresh payload with the access secret in a fixture and assert the access strategy rejects `tokenUse: "refresh"`, wrong issuer, wrong audience, invalid role, and missing identifiers.

- [ ] **Step 8: Implement fail-closed JWT contracts**

Add a pure configuration validator wired through `ConfigModule.forRoot({ validate })`. Issue distinct `tokenUse`, issuer, audience, and HS256 claims. Configure both strategies with issuer, audience, algorithms, and runtime payload narrowing before assigning `req.user`.

- [ ] **Step 9: Write and pass the Inicis redirect regression test**

The adapter test must return an allowed-host 302 to `http://127.0.0.1/` and assert verification rejects without making the second request. Implement with `redirect: "error"` while preserving the existing 5-second signal and response checks.

- [ ] **Step 10: Run backend auth verification and commit**

Run focused specs, full unit suite, full integration suite, typecheck, lint, build, and audit. Commit with `fix(auth): bind callback proofs and harden tokens`.
