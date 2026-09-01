# FO Account Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove synthetic Kakao passwords, add enumeration-safe password recovery, and implement a reversible 30-day FO account deactivation lifecycle.

**Architecture:** Keep authentication in the existing auth and FO-auth modules. Add one focused FO-account module for deactivation, reactivation, and anonymization. Database constraints own lifecycle invariants; every session-issuing path checks current account state. The existing email verification/reset flow is reused instead of creating a second password-change system.

**Tech Stack:** NestJS, GraphQL, Drizzle ORM, PostgreSQL, bcrypt, Expo Router, React Native, TanStack Query, Jest.

**Spec:** `docs/superpowers/specs/2026-08-31-fo-header-actions-notifications-design.md`

## Global Constraints

- Create or switch both submodules to `feat/fo-notifications` before the first edit; the parent repository already uses that branch.
- Reserve `migrations/0025_fo_account_lifecycle.sql` for this plan. The notification plan must use `0026`.
- Execute this plan completely before the notification backend plan. The later notification plan extends deactivation and anonymization with Push-table cleanup after those tables exist.
- Use arrow functions and no code comments.
- Observe every focused regression test fail before implementing its production change.
- Never reveal whether an email belongs to an email-password or Kakao-only account.
- Never issue access or refresh tokens while `deactivatedAt` or `anonymizedAt` is set.
- Preserve `users`, `orders`, `orderItems`, `stylePosts`, `userConsentAcceptances`, and `auditLogs` during anonymization. Remove the personal and behavioral rows named in Task 4.
- Do not add a queue dependency. The anonymization job is bounded, PostgreSQL-local work using `FOR UPDATE SKIP LOCKED`.
- Commit backend and frontend changes inside their submodules. Do not commit the parent submodule pointers until all three plans are complete.

## File Structure

| Unit | Responsibility |
| --- | --- |
| `migrations/0025_fo_account_lifecycle.sql` and `database/schema.ts` | Password nullability, lifecycle timestamps, token persistence, database constraints |
| `modules/fo-account/*` | Reactivation tokens, deactivation/reactivation transactions, due-account anonymization |
| `modules/auth/*` and `strategies/access-token.strategy.ts` | Viewer `hasPassword`, session issuance/revocation, immediate active-account checks |
| `modules/fo-auth/*` | Email and Kakao provider authentication plus `REACTIVATION_REQUIRED` branching |
| `modules/email/*` | Enumeration-safe suppression for passwordless reset requests |
| `modules/order/order.service.ts` | Checkout/deactivation serialization |
| `features/auth/*` and `app/auth/*` | Native password warning and in-memory account-reactivation UX |
| Focused unit/integration tests named by each task | Regression proof at every trust and transaction boundary |

## Contract Decisions

```graphql
enum FoSigninStatus {
  SIGNED_IN
  REACTIVATION_REQUIRED
}

type FoSigninResult {
  status: FoSigninStatus!
  tokenPayload: TokenPayload
  reactivationToken: String
}

type FoAccountDeactivationPayload {
  ok: Boolean!
  scheduledAnonymizationAt: DateTime!
}

input ReactivateFoAccountInput {
  reactivationToken: String!
}
```

- `signinFo` returns `FoSigninResult!` instead of `TokenPayload!`.
- `KakaoLoginStatus` adds `REACTIVATION_REQUIRED`; `KakaoLoginResult` adds nullable `reactivationToken`.
- `me` adds `hasPassword: Boolean!`.
- Reactivation tokens live for 10 minutes, are stored hashed, bound to the request device ID, single-use, and invalid once the 30-day deadline passes.
- `deactivateFoAccount` rejects `PAYMENT_PENDING`, `PAID`, and `FULFILLING` orders.

---

### Task 1: Add nullable-password and lifecycle database invariants

**Files:**
- Create: `dadamjang-be/migrations/0025_fo_account_lifecycle.sql`
- Modify: `dadamjang-be/src/modules/database/schema.ts`
- Modify: `dadamjang-be/test/database-migration.integration-spec.ts`

**Interfaces:**
- `users.password: string | null`
- `users.deactivatedAt: Date | null`
- `users.scheduledAnonymizationAt: Date | null`
- `users.anonymizedAt: Date | null`
- `accountReactivationTokens(tokenHash, userId, deviceIdHash, expiresAt, usedAt, createdAt)`

- [ ] **Step 1: Write the failing migration tests**

Add assertions that a USER may have a null password, non-USER roles may not, lifecycle timestamp constraints reject impossible rows, duplicate reactivation token hashes fail, and the due-account partial index exists. Seed three migration cases: a same-transaction Kakao-only user, an email user later linked to Kakao, and a non-USER account.

```typescript
expect(kakaoOnly.password).toBeNull();
expect(linkedEmail.password).toBe(emailPasswordHash);
await expect(insertNonUserWithoutPassword()).rejects.toThrow();
```

- [ ] **Step 2: Run the focused migration test and observe RED**

Run from `dadamjang-be`:

```bash
pnpm db:test:up
pnpm test:integration -- --runTestsByPath test/database-migration.integration-spec.ts
```

Expected failure: lifecycle columns and `accountReactivationTokens` do not exist.

- [ ] **Step 3: Add the migration and matching Drizzle schema**

Use the following invariants in the migration and mirror them in `schema.ts`:

```sql
ALTER TABLE "users" ALTER COLUMN "password" DROP NOT NULL;
ALTER TABLE "users" ADD COLUMN IF NOT EXISTS "deactivatedAt" timestamptz;
ALTER TABLE "users" ADD COLUMN IF NOT EXISTS "scheduledAnonymizationAt" timestamptz;
ALTER TABLE "users" ADD COLUMN IF NOT EXISTS "anonymizedAt" timestamptz;

UPDATE "users" AS u
SET "password" = NULL
FROM "authIdentities" AS ai
WHERE ai."userId" = u."userId"
  AND ai."provider" = 'kakao'
  AND ai."createdAt" = u."createdAt"
  AND u."updatedAt" = u."createdAt"
  AND u."role" = 'USER';
```

Add named checks for non-USER passwords and lifecycle ordering, a partial due index on `(scheduledAnonymizationAt, userId)`, and `ON DELETE CASCADE` from reactivation tokens to users. Follow the existing ordered migration style and let the migration ledger prevent reapplication.

- [ ] **Step 4: Re-run the migration test and smoke migration**

```bash
pnpm test:integration -- --runTestsByPath test/database-migration.integration-spec.ts
node scripts/migrate-prod-smoke.mjs
```

Expected result: the integration test proves `0025` applies once and the migration ledger prevents reapplication; the smoke script proves the compiled production migration command loads without a TypeScript runtime.

- [ ] **Step 5: Commit the database change**

```bash
git add migrations/0025_fo_account_lifecycle.sql src/modules/database/schema.ts test/database-migration.integration-spec.ts
git commit -m "feat(auth): add FO account lifecycle schema"
```

---

### Task 2: Make password and sign-in behavior provider-correct

**Files:**
- Modify: `dadamjang-be/src/modules/auth/auth.types.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.repository.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.service.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.service.spec.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.types.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.module.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.repository.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.service.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.service.spec.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.resolver.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/kakao-flow.repository.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/kakao-flow.service.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/kakao-flow.service.spec.ts`
- Modify: `dadamjang-be/src/modules/email/email.repository.ts`
- Modify: `dadamjang-be/src/modules/email/email.service.spec.ts`
- Create: `dadamjang-be/src/modules/fo-account/fo-account.module.ts`
- Create: `dadamjang-be/src/modules/fo-account/fo-account.repository.ts`
- Create: `dadamjang-be/src/modules/fo-account/fo-account.service.ts`
- Modify: `dadamjang-be/src/modules/app.module.ts`
- Modify: `dadamjang-be/test/fo-auth.integration-spec.ts`
- Modify: `dadamjang-be/test/fo-recovery.integration-spec.ts`
- Modify: `dadamjang-be/test/email-outbox.integration-spec.ts`

**Interfaces:**

```typescript
export const FoSigninStatus = {
  SignedIn: "SIGNED_IN",
  ReactivationRequired: "REACTIVATION_REQUIRED",
} as const;

export class FoSigninResult {
  status!: (typeof FoSigninStatus)[keyof typeof FoSigninStatus];
  tokenPayload?: TokenPayload;
  reactivationToken?: string;
}
```

Extend the existing Kakao result without weakening its non-null field:

```typescript
export const KakaoLoginStatus = {
  SIGNED_IN: "SIGNED_IN",
  SIGNUP_REQUIRED: "SIGNUP_REQUIRED",
  REACTIVATION_REQUIRED: "REACTIVATION_REQUIRED",
} as const;

const kakaoReactivationResult = {
  status: KakaoLoginStatus.REACTIVATION_REQUIRED,
  tokenPayload: null,
  kakaoSignupToken: null,
  email: null,
  emailVerificationRequired: false,
  reactivationToken,
};
```

Add nullable `reactivationToken` to `KakaoLoginResult`; keep `emailVerificationRequired` non-null for all three statuses.

- [ ] **Step 1: Write failing unit and integration tests**

Cover all of these cases:

- New Kakao-only signup stores `password = NULL`.
- Email signup still stores a bcrypt hash.
- Null-password email sign-in performs the existing dummy bcrypt comparison and returns the same generic authentication error as a missing account.
- Password-reset request returns `{ ok: true }`, retains one terminal `SUPPRESSED` Outbox row, and sends no mail for a null-password user.
- A stale reset proof cannot set a password on a null-password user.
- Active email and Kakao users receive normal tokens.
- Deactivated email and Kakao users receive `REACTIVATION_REQUIRED` and no session tokens.
- Kakao reactivation returns `emailVerificationRequired=false`, all unrelated nullable fields as null, and the device-bound reactivation token.
- `me.hasPassword` is true only when the authenticated user's password is non-null.

```typescript
expect(await signinPasswordless()).rejects.toMatchObject({
  message: "이메일 또는 비밀번호가 올바르지 않습니다.",
});
expect(await signinDeactivated()).toEqual({
  status: "REACTIVATION_REQUIRED",
  reactivationToken: expect.any(String),
});
expect(await requestPasswordReset(passwordlessEmail)).toEqual({ ok: true });
expect(emailOutboxRows).toEqual([
  expect.objectContaining({ kind: "PASSWORD_RESET_LINK", status: "SUPPRESSED" }),
]);
expect(emailSender).not.toHaveBeenCalled();
```

- [ ] **Step 2: Run the focused tests and observe RED**

```bash
pnpm test:unit -- modules/auth/auth.service.spec.ts modules/fo-auth/fo-auth.service.spec.ts modules/fo-auth/kakao-flow.service.spec.ts modules/email/email.service.spec.ts
pnpm test:integration -- --runTestsByPath test/fo-auth.integration-spec.ts test/fo-recovery.integration-spec.ts test/email-outbox.integration-spec.ts
```

Expected failure: current Kakao signup writes a random hash and `signinFo` returns a raw token payload.

- [ ] **Step 3: Implement the smallest coordinated contract change**

Remove random-password generation from `KakaoFlowService.completeSignup` and insert null. Add `hasPassword` to the existing viewer query. Reuse the existing dummy hash branch for both missing and passwordless accounts. Change reset delivery preparation to retain the existing enumeration-safe terminal `SUPPRESSED` Outbox row for unknown or passwordless users while never calling the sender and retaining the public success response.

Create the focused FO-account module now with only `createReactivationToken(userId, deviceId)`. It generates a 32-byte random token, stores only its SHA-256 token hash and device-ID hash with a 10-minute expiry, and returns the plaintext once. Email and Kakao sign-in call it only after provider authentication succeeds and account state is found deactivated. Derive the device ID in the existing resolvers with `deviceIdFromRequest(req)`.

Return one of these exact shapes from FO email sign-in:

```typescript
return deactivatedAt
  ? { status: FoSigninStatus.ReactivationRequired, reactivationToken }
  : { status: FoSigninStatus.SignedIn, tokenPayload };
```

Expose the provider-independent token method with this exact shape:

```typescript
createReactivationToken = async (userId: string, deviceId: string) => {
  const reactivationToken = randomBytes(32).toString("base64url");
  await this.repository.insertReactivationToken({
    tokenHash: hashToken(reactivationToken),
    userId,
    deviceIdHash: hashToken(deviceId),
    expiresAt: new Date(Date.now() + 10 * 60_000),
  });
  return reactivationToken;
};
```

Do not issue tokens before the lifecycle branch. Apply the same rule after successful Kakao identity validation.

- [ ] **Step 4: Re-run the focused tests**

Run the commands from Step 2. Expected result: PASS.

- [ ] **Step 5: Commit provider-correct authentication**

```bash
git add src/modules/auth src/modules/fo-auth src/modules/email src/modules/fo-account src/modules/app.module.ts test/fo-auth.integration-spec.ts test/fo-recovery.integration-spec.ts test/email-outbox.integration-spec.ts
git commit -m "fix(auth): remove synthetic Kakao passwords"
```

---

### Task 3: Add deactivation, device-bound reactivation, and immediate session rejection

**Files:**
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.module.ts`
- Create: `dadamjang-be/src/modules/fo-account/fo-account.types.ts`
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.repository.ts`
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.service.ts`
- Create: `dadamjang-be/src/modules/fo-account/fo-account.resolver.ts`
- Create: `dadamjang-be/test/fo-account-lifecycle.integration-spec.ts`
- Modify: `dadamjang-be/src/modules/auth/auth-http.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.repository.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.service.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.service.spec.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/fo-auth.resolver.ts`
- Modify: `dadamjang-be/src/modules/fo-auth/kakao-flow.resolver.ts`
- Modify: `dadamjang-be/src/strategies/access-token.strategy.ts`
- Modify: `dadamjang-be/src/strategies/token.strategy.spec.ts`
- Modify: `dadamjang-be/src/modules/order/order.service.ts`
- Modify: `dadamjang-be/test/order-concurrency.integration-spec.ts`

**Service API:**

```typescript
deactivate = async (userId: string): Promise<FoAccountDeactivationPayload>;
createReactivationToken = async (userId: string, deviceId: string): Promise<string>;
reactivate = async (reactivationToken: string, deviceId: string): Promise<TokenPayload>;
```

- [ ] **Step 1: Write failing lifecycle and race tests**

Assert that deactivation:

- locks the USER row;
- rejects active orders;
- records DB-time `deactivatedAt` and exactly 30-day `scheduledAnonymizationAt`;
- deletes all refresh sessions in the same transaction;
- causes old access and refresh tokens to fail immediately;
- clears access and refresh cookies in the deactivation response;
- races safely with checkout, so only checkout or deactivation commits when both start together.
- races safely with generic, FO email, Kakao, signup-completion, and refresh token issuance: the lifecycle update or session issuance may commit first, but no refresh row remains for a deactivated USER.

Assert that reactivation tokens are hashed, valid only for the issuing device, usable once, expire after 10 minutes, fail at the anonymization deadline, and atomically clear lifecycle timestamps before issuing tokens. A successful reactivation response must set the normal access and refresh cookies; every `REACTIVATION_REQUIRED` email/Kakao response must set neither cookie.

```typescript
const [checkoutResult, deactivationResult] = await Promise.allSettled([
  checkoutCart(userId, checkoutInput),
  deactivateFoAccount(userId),
]);
expect([checkoutResult.status, deactivationResult.status].filter((status) => status === "fulfilled")).toHaveLength(1);
await expect(validateAccessToken(oldAccessToken)).rejects.toThrow();
await expect(reactivateFoAccount(reactivationToken, "other-device")).rejects.toThrow();
expect(deactivateResponse.headers["set-cookie"]).toEqual(
  expect.arrayContaining([expect.stringContaining("access_token=;"), expect.stringContaining("refresh_token=;")]),
);
expect(reactivateResponse.headers["set-cookie"]).toEqual(
  expect.arrayContaining([expect.stringContaining("access_token="), expect.stringContaining("refresh_token=")]),
);
```

- [ ] **Step 2: Run the focused tests and observe RED**

```bash
pnpm test:unit -- modules/auth/auth.service.spec.ts strategies/token.strategy.spec.ts
pnpm test:integration -- --runTestsByPath test/fo-account-lifecycle.integration-spec.ts test/order-concurrency.integration-spec.ts
```

- [ ] **Step 3: Implement transactional lifecycle operations**

Use the repository's `DatabaseTransaction` type. Lock the user with `FOR UPDATE`, query blocking order states inside the same transaction, use `transaction_timestamp()`, and revoke refresh sessions. Push devices do not exist yet; Notification Backend Task 2 adds their disable operation to this same transaction.

Centralize the lifecycle guard at the session-write boundary. Add one repository `withActiveUserSession(userId, action, store?)` transaction helper that locks and reloads the user, rejects either lifecycle field, and invokes the action with the same transaction. `AuthService.issueTokensForUser` signs and saves only inside that helper. `AuthService.refresh` moves refresh lookup, token creation, and compare-and-swap rotation inside the same helper instead of creating tokens before the lifecycle lock. When an existing outer transaction is supplied, reuse it and take the same row lock. This covers generic portal sign-in, FO email sign-in, Kakao login/signup completion, signup token issuance, reactivation, and refresh without duplicating guards at every caller. In the sign-in/deactivation race, whichever transaction acquires the user lock first wins; deactivation deletes any refresh row created by an earlier sign-in, while a later sign-in observes the inactive row and creates none.

`AccessTokenStrategy.validate` must also load the user and reject when either lifecycle field is set. In `OrderService.checkout`, lock and recheck the same user before creating the order so checkout and deactivation serialize.

Derive the reactivation device ID with the existing `deviceIdFromRequest(req)`. Hash both token and device ID before persistence; use `crypto.randomBytes(32)` and the existing SHA-256 helper or Node `createHash`.

The deactivation transaction must contain this lock/update sequence:

```typescript
deactivate = async (userId: string) =>
  this.db.transaction(async (tx) => {
    const [user] = await tx.select().from(users).where(eq(users.userId, userId)).for("update").limit(1);
    if (!user || user.role !== "USER" || user.anonymizedAt)
      throw new CustomForbiddenException("탈퇴할 수 없는 계정입니다.");
    const [blockingOrder] = await tx
      .select({ orderId: orders.orderId })
      .from(orders)
      .where(and(eq(orders.userId, userId), inArray(orders.status, ["PAYMENT_PENDING", "PAID", "FULFILLING"])))
      .limit(1);
    if (blockingOrder) throw new CustomConflictException("진행 중인 주문이 있어 탈퇴할 수 없습니다.");
    const [updated] = await tx
      .update(users)
      .set({
        deactivatedAt: sql`transaction_timestamp()`,
        scheduledAnonymizationAt: sql`transaction_timestamp() + interval '30 days'`,
      })
      .where(eq(users.userId, userId))
      .returning({ scheduledAnonymizationAt: users.scheduledAnonymizationAt });
    await tx.delete(refreshTokens).where(eq(refreshTokens.userId, userId));
    const scheduledAnonymizationAt = requireResult(updated).scheduledAnonymizationAt;
    if (!scheduledAnonymizationAt) throw new Error("Scheduled anonymization timestamp is missing");
    return { ok: true, scheduledAnonymizationAt };
  });
```

Add `clearTokenCookies(res)` beside `setTokenCookies` in `auth-http.ts` and reuse the existing `authCookieOptions`. The deactivation resolver calls it only after commit; the reactivation resolver calls `setTokenCookies` only for the returned payload. Email and Kakao sign-in resolvers set cookies only when their discriminated result contains `tokenPayload`.

- [ ] **Step 4: Re-run focused tests**

Run the commands from Step 2. Expected result: PASS.

- [ ] **Step 5: Commit the lifecycle API**

```bash
git add src/modules/fo-account src/modules/app.module.ts src/modules/auth src/modules/fo-auth src/strategies src/modules/order test/fo-account-lifecycle.integration-spec.ts test/order-concurrency.integration-spec.ts
git commit -m "feat(auth): add FO deactivation and recovery"
```

---

### Task 4: Add bounded 30-day anonymization

**Files:**
- Create: `dadamjang-be/src/modules/fo-account/fo-account.worker.ts`
- Create: `dadamjang-be/src/modules/fo-account/fo-account.worker.spec.ts`
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.module.ts`
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.repository.ts`
- Modify: `dadamjang-be/.env.example`
- Modify: `dadamjang-be/test/support/env.ts`
- Modify: `dadamjang-be/test/fo-account-lifecycle.integration-spec.ts`
- Modify: `dadamjang-be/src/modules/style-posts/style-posts.service.ts`
- Modify: `dadamjang-be/src/modules/style-posts/style-posts.service.spec.ts`

**Worker behavior:**
- Tick every 60 seconds when `FO_ACCOUNT_ANONYMIZATION_WORKER_ENABLED=true`.
- Claim at most 100 due users with `FOR UPDATE SKIP LOCKED`.
- Complete each user's DB-only cleanup in the same transaction as its row lock.

- [ ] **Step 1: Write failing cleanup and display tests**

Seed every currently existing personal/behavioral relation for one due user. Assert removal of refresh and recovery tokens, reset email records/outbox, Kakao flows/tokens, identities, verification sessions, likes, follows, wishes, recent views, comparisons, carts/items, checkout idempotency keys, and activity events. Notification Backend Task 2 adds the four Push/notification tables to this cleanup after they exist.

Assert preservation of users, orders/items, style posts, consent acceptances, audit logs, and media ownership records. The preserved user must become:

```typescript
expect(user.userid).toBe(`deleted-${userId.replaceAll("-", "")}`);
expect(user.email).toBe(`deleted+${userId.replaceAll("-", "")}@invalid.local`);
expect(user.password).toBeNull();
expect(user.anonymizedAt).not.toBeNull();
```

```typescript
expect(await countPersonalRows(userId)).toEqual({
  authIdentities: 0,
  cartItems: 0,
  carts: 0,
  refreshTokens: 0,
  stylePostLikes: 0,
  wishes: 0,
});
expect(await countPreservedRows(userId)).toEqual({ orders: 1, orderItems: 1, stylePosts: 1, users: 1 });
```

Assert preserved style posts expose author label `탈퇴한 사용자`.

- [ ] **Step 2: Run the tests and observe RED**

```bash
pnpm test:unit -- modules/fo-account/fo-account.worker.spec.ts modules/style-posts/style-posts.service.spec.ts
pnpm test:integration -- --runTestsByPath test/fo-account-lifecycle.integration-spec.ts
```

- [ ] **Step 3: Implement ordered cleanup and worker lifecycle**

Capture original email, CI hash, and Kakao provider IDs before deleting relations. Delete dependent rows in the exact order asserted by the test, then update the retained user row. Reject reactivation using `scheduledAnonymizationAt <= transaction_timestamp()` even if the worker has not run yet.

Follow the existing email worker's Nest lifecycle and environment-disable pattern, but do not copy its external-I/O claim state because anonymization never leaves PostgreSQL.

```typescript
runOnce = async () => this.repository.anonymizeDueBatch(100);
```

`anonymizeDueBatch` acquires due user rows with `FOR UPDATE SKIP LOCKED`, executes the ordered deletes, and performs the deterministic user updates before the same transaction returns.

- [ ] **Step 4: Re-run tests and commit**

```bash
pnpm test:unit -- modules/fo-account/fo-account.worker.spec.ts modules/style-posts/style-posts.service.spec.ts
pnpm test:integration -- --runTestsByPath test/fo-account-lifecycle.integration-spec.ts
git add src/modules/fo-account src/modules/style-posts .env.example test/support/env.ts test/fo-account-lifecycle.integration-spec.ts
git commit -m "feat(auth): anonymize withdrawn FO accounts"
```

---

### Task 5: Implement FO password warning and reactivation UX

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/components/auth-links.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/components/password-reset-form.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/components/index.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/auth/find-password.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/app/auth/reactivate.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/auth/_layout.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/auth/index.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/auth/signin.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/api.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/kakao-api.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/types.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/hooks.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/auth-flow-provider.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/signin.test.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/account-recovery.test.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/account-lifecycle.test.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/api-contracts.test.ts`

**Client types:**

```typescript
export type SignInFoResult =
  | { status: "SIGNED_IN"; tokenPayload: TokenPayload; reactivationToken: null }
  | { status: "REACTIVATION_REQUIRED"; tokenPayload: null; reactivationToken: string };

export type KakaoLoginResult =
  | {
      status: "SIGNED_IN";
      tokenPayload: TokenPayload;
      kakaoSignupToken: null;
      email: null;
      emailVerificationRequired: false;
      reactivationToken: null;
    }
  | {
      status: "SIGNUP_REQUIRED";
      tokenPayload: null;
      kakaoSignupToken: string;
      email: string | null;
      emailVerificationRequired: boolean;
      reactivationToken: null;
    }
  | {
      status: "REACTIVATION_REQUIRED";
      tokenPayload: null;
      kakaoSignupToken: null;
      email: null;
      emailVerificationRequired: false;
      reactivationToken: string;
    };
```

- [ ] **Step 1: Write failing native Alert tests**

From both auth landing and email sign-in, press `비밀번호 찾기`. Assert `Alert.alert` receives the exact body below and two buttons. Invoke `취소` and assert no navigation; invoke `이메일 계정 계속` and assert navigation to the existing reset route.

```text
이메일로 가입한 계정만 비밀번호를 재설정할 수 있어요. 카카오로 가입했다면 카카오 로그인을 이용해 주세요.
```

```typescript
expect(Alert.alert).toHaveBeenCalledWith(
  "비밀번호 찾기",
  "이메일로 가입한 계정만 비밀번호를 재설정할 수 있어요. 카카오로 가입했다면 카카오 로그인을 이용해 주세요.",
  expect.arrayContaining([
    expect.objectContaining({ text: "취소" }),
    expect.objectContaining({ text: "이메일 계정 계속" }),
  ]),
);
```

- [ ] **Step 2: Write failing email and Kakao reactivation tests**

Assert a `REACTIVATION_REQUIRED` response stores the token only in `AuthFlowProvider`, opens `/auth/reactivate`, and issues no local session. Confirming calls `reactivateFoAccount`; cancelling clears the in-memory token. Cover email in `signin.tsx`, Kakao in `app/auth/index.tsx`, and token failure.

```typescript
expect(setAuthTokens).not.toHaveBeenCalled();
expect(router.push).toHaveBeenCalledWith("/auth/reactivate");
expect(await screen.findByText("계정을 다시 사용할까요?")).toBeTruthy();
fireEvent.press(screen.getByText("계정 복구"));
expect(reactivateFoAccount).toHaveBeenCalledWith("reactivation-token");
```

- [ ] **Step 3: Run focused FO tests and observe RED**

Run from `dadamjang-fe`:

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/signin.test.tsx __tests__/integration/account-recovery.test.tsx __tests__/integration/account-lifecycle.test.tsx __tests__/unit/api-contracts.test.ts
```

- [ ] **Step 4: Implement the shared warning, form extraction, and reactivation branch**

Put the native warning only in `auth-links.tsx` so both callers share it. Extract the current reset body without behavioral changes. Do not put reactivation tokens in route params or persistent storage. Store tokens only after successful provider authentication and clear them after confirm, cancel, expiry, or error.

Only persist `tokenPayload` for `SIGNED_IN`. On successful reactivation, use the existing auth-token/session setter and replace the route with the sanitized pending `returnTo` or `/`.

```typescript
const openPasswordReset = () =>
  Alert.alert("비밀번호 찾기", passwordResetWarning, [
    { text: "취소", style: "cancel" },
    { text: "이메일 계정 계속", onPress: onFindPassword },
  ]);

const completeSignIn = async (result: SignInFoResult) => {
  if (result.status === "REACTIVATION_REQUIRED") {
    setPendingReactivationToken(result.reactivationToken);
    router.push("/auth/reactivate");
    return;
  }
  await setAuthTokens(result.tokenPayload);
};
```

- [ ] **Step 5: Re-run tests and commit**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/signin.test.tsx __tests__/integration/account-recovery.test.tsx __tests__/integration/account-lifecycle.test.tsx __tests__/unit/api-contracts.test.ts
git add apps/dadamjang-fo/src apps/dadamjang-fo/__tests__
git commit -m "feat(fo): add account recovery UX"
```

---

### Task 6: Verify the account lifecycle plan

- [ ] **Step 1: Run backend verification**

Run from `dadamjang-be`:

```bash
pnpm build
pnpm lint
pnpm test:unit
pnpm test:integration
node scripts/migrate-prod-smoke.mjs
pnpm db:test:down
```

- [ ] **Step 2: Run FO verification**

Run from `dadamjang-fe`:

```bash
pnpm fo:typecheck
pnpm fo:lint
pnpm fo:test:unit
pnpm fo:test:integration
```

- [ ] **Step 3: Inspect only the intended diffs**

Run from the parent workspace:

```bash
git -C dadamjang-be status --short
git -C dadamjang-fe status --short
git diff --submodule=log
```

Expected result: submodule worktrees are clean, focused and full suites pass, and only submodule pointers remain changed in the parent repository.
