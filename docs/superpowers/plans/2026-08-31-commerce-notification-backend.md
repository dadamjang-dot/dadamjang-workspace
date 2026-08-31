# Commerce Notification Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Persist real FO commerce notifications transactionally, expose a secure inbox API, send Expo Push with retries and receipts, and add the missing Partner published-SKU update flow that produces price-drop and restock events.

**Architecture:** Domain writers call one notification service inside their existing database transaction. That service owns copy, route, dedupe keys, preference checks, and Push Outbox creation. A Nest worker reuses the email Outbox claim/retry/retention pattern and uses Node `fetch` against Expo; no queue or Expo server SDK is added.

**Tech Stack:** NestJS, GraphQL, Drizzle ORM, PostgreSQL, Node fetch, Expo Push API, Next.js Partner, TanStack Query, Vitest, Playwright, Jest.

**Spec:** `docs/superpowers/specs/2026-08-31-fo-header-actions-notifications-design.md`

## Global Constraints

- Complete the account-lifecycle plan first. This plan then owns `migrations/0026_notifications_push_outbox.sql`.
- Use arrow functions and no code comments.
- Write and observe each focused test failing before implementation.
- Generate routes only in the notification service: `/order/:id`, `/product/:id`, `/style/:id`.
- Preferences suppress Push Outbox rows only. The app notification row is always created.
- Do not invent a payment gateway or weaken current order transition rules. Wire every existing actual status writer. Keep `PAID` copy and creation supported, but treat delivery from a real payment as externally blocked until a payment provider and signed callback verification contract are supplied.
- Keep one production replica with `PUSH_OUTBOX_WORKER_ENABLED=true`; do not add a distributed rate limiter before horizontal Push workers exist.
- Update published SKU rows in place. Never delete/reinsert them because carts and historical orders reference SKU IDs.
- Commit backend and frontend changes inside their submodules. Do not commit the parent submodule pointers until the FO surfaces plan is complete.

## File Structure

| Unit | Responsibility |
| --- | --- |
| `migrations/0026_notifications_push_outbox.sql` and `database/schema.ts` | Inbox, device, preference, and delivery state invariants |
| `modules/notification/notification.types.ts` | GraphQL enums, inputs, objects, and connection contracts |
| `modules/notification/notification.repository.ts` | Authorized inbox queries, device ownership, atomic notification/Outbox writes, worker claims |
| `modules/notification/notification.service.ts` | Stable copy, route, dedupe, preference, and producer rules |
| `modules/notification/notification.resolver.ts` | Authenticated FO GraphQL surface and request-device extraction |
| `modules/notification/notification.sender.ts` | Strict Expo send/receipt HTTP client |
| `modules/notification/notification.outbox.ts` | Claim, retry, receipt, invalid-device, and retention loop |
| Existing admin/style/partner services | Transactional event producer seams only |
| Existing Partner product editor and contract | Published price/stock-only mutation UX |
| Focused backend and Partner tests named by each task | Schema, transaction, worker, and UI regression proof |

## Stable Notification Copy

| Event | Title | Body |
| --- | --- | --- |
| `PAID` | 결제가 완료됐어요 | 주문 상품을 준비할게요. |
| `FULFILLING` | 상품을 준비하고 있어요 | 준비가 끝나면 다시 알려드릴게요. |
| `COMPLETED` | 주문이 완료됐어요 | 구매한 상품을 확인해 보세요. |
| `FAILED` | 주문 처리가 완료되지 않았어요 | 주문 상세에서 상태를 확인해 주세요. |
| `CANCELLED` | 주문이 취소됐어요 | 주문 상세에서 취소 내용을 확인해 주세요. |
| `WISH_PRICE_DROP` | 위시 상품 가격이 내려갔어요 | 찜한 상품을 지금 확인해 보세요. |
| `WISH_RESTOCK` | 위시 상품이 다시 입고됐어요 | 품절되기 전에 확인해 보세요. |
| `STYLE_LIKE` | 스타일에 좋아요가 달렸어요 | 내 스타일 게시물을 확인해 보세요. |

## GraphQL Contract

```graphql
enum FoNotificationType {
  ORDER_STATUS
  WISH_PRICE_DROP
  WISH_RESTOCK
  STYLE_LIKE
}

enum FoPushPlatform {
  IOS
  ANDROID
}

type FoNotification {
  notificationId: ID!
  type: FoNotificationType!
  title: String!
  body: String!
  route: String!
  entityId: ID!
  readAt: DateTime
  createdAt: DateTime!
}

type FoNotificationConnection {
  nodes: [FoNotification!]!
  nextCursor: String
  hasNextPage: Boolean!
  unreadCount: Int!
}

type FoNotificationPreferences {
  pushEnabled: Boolean!
  orderPushEnabled: Boolean!
  wishPushEnabled: Boolean!
  stylePushEnabled: Boolean!
  updatedAt: DateTime!
}

input RegisterFoPushDeviceInput {
  expoPushToken: String!
  platform: FoPushPlatform!
}

input UpdateFoNotificationPreferencesInput {
  pushEnabled: Boolean
  orderPushEnabled: Boolean
  wishPushEnabled: Boolean
  stylePushEnabled: Boolean
}
```

Operations:

```graphql
foNotifications(first: Int, after: String): FoNotificationConnection!
foNotification(notificationId: ID!): FoNotification!
markFoNotificationRead(notificationId: ID!): FoNotification!
markAllFoNotificationsRead: Boolean!
foNotificationPreferences: FoNotificationPreferences!
updateFoNotificationPreferences(input: UpdateFoNotificationPreferencesInput!): FoNotificationPreferences!
registerFoPushDevice(input: RegisterFoPushDeviceInput!): Boolean!
unregisterFoPushDevice: Boolean!
updatePublishedProductSkus(input: UpdatePublishedProductSkusInput!): PartnerProductType!
```

The server derives `installationId` from `deviceIdFromRequest(req)` and verifies an active refresh session for that user/device. It is not a caller-controlled GraphQL field.

## External Payment Precondition

The current backend has no payment module, provider SDK, signed webhook, callback, or production `PAYMENT_PENDING -> PAID` writer. `AdminService.transitionOrder` intentionally forbids that transition. This plan therefore does not expose an internal or GraphQL shortcut that could mark an unpaid order as paid.

When the payment provider is chosen, add a provider-signed REST callback that verifies the signature/API result, provider order reference, exact amount, success state, and replay before a conditional `OrderService.confirmVerifiedPayment` update. That transaction must require `status=PAYMENT_PENDING` and `paymentStatus=PENDING`, set `PAID/APPROVED`, and call `createOrderStatus` before commit. Until then, real Push verification uses the existing `PAID -> FULFILLING` admin transition and PAID itself remains the one explicitly blocked producer.

---

### Task 1: Add notification, device, preference, and Push Outbox tables

**Files:**
- Create: `dadamjang-be/migrations/0026_notifications_push_outbox.sql`
- Modify: `dadamjang-be/src/modules/database/schema.ts`
- Modify: `dadamjang-be/test/database-migration.integration-spec.ts`

**Tables:**
- `notifications`
- `pushDevices`
- `notificationPreferences`
- `pushOutbox`

- [ ] **Step 1: Write failing migration tests**

Assert table, foreign-key, check, partial-index, and uniqueness behavior. Cover duplicate `(userId, dedupeKey)`, installation ID, Expo token, and `(notificationId, pushDeviceId)`; invalid notification type/platform/status; inconsistent claim state; and ticket/receipt state. Run the full migration twice with existing users, wishes, and orders preserved.

```typescript
await expect(insertNotification({ userId, dedupeKey: "same" })).resolves.toBeDefined();
await expect(insertNotification({ userId, dedupeKey: "same" })).rejects.toMatchObject({ code: "23505" });
await expect(insertPushDevice({ installationId, expoPushToken, platform: "WEB" })).rejects.toMatchObject({
  code: "23514",
});
await expect(insertOutbox({ status: "PROCESSING", claimToken: null, claimedAt: null })).rejects.toMatchObject({
  code: "23514",
});
```

- [ ] **Step 2: Run the migration test and observe RED**

From `dadamjang-be`:

```bash
pnpm db:test:up
pnpm test:integration -- --runTestsByPath test/database-migration.integration-spec.ts
```

- [ ] **Step 3: Implement SQL and matching Drizzle tables**

Use these database invariants:

```sql
UNIQUE ("userId", "dedupeKey")
UNIQUE ("installationId")
UNIQUE ("expoPushToken")
UNIQUE ("notificationId", "pushDeviceId")
```

Allow notification types `ORDER_STATUS`, `WISH_PRICE_DROP`, `WISH_RESTOCK`, `STYLE_LIKE`; device platforms `IOS`, `ANDROID`; outbox statuses `PENDING`, `PROCESSING`, `TICKETED`, `RECEIPT_OK`, `FAILED`. Add user/created keyset, unread, active-device, pending, ticketed-receipt, stale-processing, and terminal-retention indexes. Cascade notifications/devices into Outbox.

- [ ] **Step 4: Re-run migration tests and smoke**

```bash
pnpm test:integration -- --runTestsByPath test/database-migration.integration-spec.ts
node scripts/migrate-prod-smoke.mjs
```

- [ ] **Step 5: Commit the schema**

```bash
git add migrations/0026_notifications_push_outbox.sql src/modules/database/schema.ts test/database-migration.integration-spec.ts
git commit -m "feat(notification): add Push Outbox schema"
```

---

### Task 2: Implement the authorized inbox, preferences, and device API

**Files:**
- Create: `dadamjang-be/src/modules/notification/notification.types.ts`
- Create: `dadamjang-be/src/modules/notification/notification.repository.ts`
- Create: `dadamjang-be/src/modules/notification/notification.service.ts`
- Create: `dadamjang-be/src/modules/notification/notification.resolver.ts`
- Create: `dadamjang-be/src/modules/notification/notification.module.ts`
- Create: `dadamjang-be/test/notification.integration-spec.ts`
- Modify: `dadamjang-be/src/modules/app.module.ts`
- Modify: `dadamjang-be/src/modules/auth/auth.repository.ts`
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.repository.ts`
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.service.ts`
- Modify: `dadamjang-be/src/modules/fo-account/fo-account.worker.ts`
- Modify: `dadamjang-be/test/fo-account-lifecycle.integration-spec.ts`

**Service API:**

```typescript
list = async (userId: string, first?: number, after?: string): Promise<FoNotificationConnection>;
get = async (userId: string, notificationId: string): Promise<FoNotification>;
markRead = async (userId: string, notificationId: string): Promise<FoNotification>;
markAllRead = async (userId: string): Promise<boolean>;
getPreferences = async (userId: string): Promise<FoNotificationPreferences>;
updatePreferences = async (
  userId: string,
  input: UpdateFoNotificationPreferencesInput,
): Promise<FoNotificationPreferences>;
registerDevice = async (
  userId: string,
  installationId: string,
  input: RegisterFoPushDeviceInput,
): Promise<boolean>;
unregisterDevice = async (userId: string, installationId: string): Promise<boolean>;
```

- [ ] **Step 1: Write failing GraphQL integration tests**

Cover stable `createdAt + notificationId` cursor pagination, bounded `first` defaults/max, unread count, individual and all-read mutations, cross-user denial, singular push-tap lookup, default preferences, partial preference updates, device transfer between users, token transfer between installations, active-refresh-session validation, and disable-on-unregister. A token disabled with `DEVICE_NOT_REGISTERED` must stay disabled when the same token is submitted again; only a different token for that installation may become active. Assert unregister, logout, and deactivation terminally fail every unsettled delivery for the disabled devices and that those rows cannot be claimed. Extend the existing account lifecycle test to assert deactivation disables every user device and anonymization deletes Push Outbox, notifications, preferences, and devices in the same lifecycle transactions.

```typescript
expect(firstPage.body.data.foNotifications).toMatchObject({
  hasNextPage: true,
  nodes: [{ notificationId: newestId }],
  unreadCount: 2,
});
expect(secondPage.body.data.foNotifications.nodes).toEqual([{ notificationId: oldestId }]);
expect(crossUserRead.body.errors[0].extensions.code).toBe("NOT_FOUND");
expect(await activePushDeviceCount(deactivatedUserId)).toBe(0);
expect(await unsettledPushOutboxCount(loggedOutDeviceId)).toBe(0);
expect(await registerDevice({ expoPushToken: invalidToken })).toBe(false);
```

- [ ] **Step 2: Run the focused test and observe RED**

```bash
pnpm test:integration -- --runTestsByPath test/notification.integration-spec.ts
```

- [ ] **Step 3: Implement the minimal module**

Use `JwtAccessTokenGuard`, `RolesGuard`, and `@Roles(UserRole.User, UserRole.Partner)` because both roles have FO sign-in capability and Partner-authored style posts can receive likes. Use the existing composite cursor encoding pattern from `AdminService`; do not create a pagination package. Cap `first` at 100.

Registration locks any existing installation/token rows, verifies `refreshTokens(userId, deviceId)` is active, then atomically transfers the device. If the same Expo token has `disabledReason=DEVICE_NOT_REGISTERED`, return `false` without clearing its disabled state; a changed token for the same installation may be activated. Unregister and logout disable by user/device and set every `PENDING`, `PROCESSING`, or `TICKETED` row for those devices to terminal `FAILED` while clearing claim fields. Extend account deactivation to perform the same disable/fail operation for every user device in its existing transaction, and extend the existing anonymization worker's ordered cleanup with Outbox, notifications, preferences, and devices before it anonymizes the user.

Change the existing logout repository transaction so refresh-session deletion and current-installation disable/fail commit together. Worker claim queries must join `pushDevices` and select only `disabledAt IS NULL`; claim completion remains guarded by both status and claim token so a row disabled after claim cannot be completed by the stale worker.

For `foNotification`, select by both `notificationId` and authenticated `userId`; never accept or return another user's row.

```typescript
registerDevice = async (userId: string, installationId: string, input: RegisterFoPushDeviceInput) =>
  this.db.transaction(async (tx) => {
    const refreshSession = await tx.query.refreshTokens.findFirst({
      where: and(eq(refreshTokens.userId, userId), eq(refreshTokens.deviceId, installationId)),
    });
    if (!refreshSession) throw new CustomUnauthorizedException("로그인이 필요합니다.");
    return this.repository.transferDevice(tx, { userId, installationId, ...input });
  });
```

`transferDevice` locks rows matching either unique key, preserves the terminal invalid-token rule above, disables conflicting ownership and its unsettled deliveries, and upserts the current user/device/token with `disabledAt = NULL`, `disabledReason = NULL`, and `lastSeenAt = transaction_timestamp()`.

- [ ] **Step 4: Re-run the focused test and account lifecycle regression**

```bash
pnpm test:integration -- --runTestsByPath test/notification.integration-spec.ts test/fo-account-lifecycle.integration-spec.ts
```

- [ ] **Step 5: Commit the API**

```bash
git add src/modules/notification src/modules/app.module.ts src/modules/auth/auth.repository.ts src/modules/fo-account test/notification.integration-spec.ts test/fo-account-lifecycle.integration-spec.ts
git commit -m "feat(notification): add FO inbox and device API"
```

---

### Task 3: Centralize transactional notification creation

**Files:**
- Modify: `dadamjang-be/src/modules/notification/notification.repository.ts`
- Modify: `dadamjang-be/src/modules/notification/notification.service.ts`
- Modify: `dadamjang-be/test/notification.integration-spec.ts`

**Producer API:**

```typescript
createOrderStatus = async (
  tx: DatabaseTransaction,
  input: { userId: string; orderId: string; status: OrderStatus },
): Promise<void>;

createWishPriceDrop = async (
  tx: DatabaseTransaction,
  input: { userIds: readonly string[]; productId: string; skuUpdatedAt: Date; newPrice: number },
): Promise<void>;

createWishRestock = async (
  tx: DatabaseTransaction,
  input: { userIds: readonly string[]; productId: string; skuUpdatedAt: Date },
): Promise<void>;

createStyleLike = async (
  tx: DatabaseTransaction,
  input: { authorUserId: string; actorUserId: string; stylePostId: string },
): Promise<void>;
```

- [ ] **Step 1: Write failing rule-boundary tests**

Assert exact copy and routes from this plan, exact dedupe formats from the spec, one app row per user, Outbox only for active enabled devices, and no arbitrary caller-supplied route/title/body. Assert category toggles suppress Outbox but retain the app row.

```typescript
await service.createWishPriceDrop(tx, { userIds: [userId], productId, skuUpdatedAt, newPrice: 9000 });
expect(await notificationRows(userId)).toEqual([
  expect.objectContaining({
    body: "찜한 상품을 지금 확인해 보세요.",
    dedupeKey: `wish-price:${productId}:${skuUpdatedAt.toISOString()}:9000`,
    route: `/product/${productId}`,
    title: "위시 상품 가격이 내려갔어요",
  }),
]);
expect(await pushOutboxCount(userId)).toBe(1);
```

- [ ] **Step 2: Run the focused tests and observe RED**

```bash
pnpm test:integration -- --runTestsByPath test/notification.integration-spec.ts
```

- [ ] **Step 3: Implement one insert path**

Map type to copy, route, and preference category inside `NotificationService`. Insert the notification with `onConflictDoNothing`, then insert Outbox rows only when the notification insert returned a row. A duplicate event must not create new deliveries.

Use the exact dedupe keys:

```typescript
const orderKey = `order:${orderId}:${status}`;
const priceKey = `wish-price:${productId}:${skuUpdatedAt.toISOString()}:${newPrice}`;
const stockKey = `wish-stock:${productId}:${skuUpdatedAt.toISOString()}`;
const styleKey = `style-like:${stylePostId}:${actorUserId}`;
```

Use one repository method for every event:

```typescript
create = async (tx: DatabaseTransaction, input: CreateNotificationInput) => {
  const [notification] = await tx.insert(notifications).values(input.notification).onConflictDoNothing().returning();
  if (!notification) return;
  const devices = await this.activeEligibleDevices(tx, input.userId, input.preferenceCategory);
  if (devices.length)
    await tx.insert(pushOutbox).values(
      devices.map(({ pushDeviceId }) => ({ notificationId: notification.notificationId, pushDeviceId })),
    );
};
```

- [ ] **Step 4: Re-run and commit**

```bash
pnpm test:integration -- --runTestsByPath test/notification.integration-spec.ts
git add src/modules/notification test/notification.integration-spec.ts
git commit -m "feat(notification): create transactional commerce alerts"
```

---

### Task 4: Wire order and style-like producers

**Files:**
- Modify: `dadamjang-be/src/modules/admin/admin.module.ts`
- Modify: `dadamjang-be/src/modules/admin/admin.service.ts`
- Modify: `dadamjang-be/src/modules/style-posts/style-posts.module.ts`
- Modify: `dadamjang-be/src/modules/style-posts/style-posts.service.ts`
- Modify: `dadamjang-be/test/admin.integration-spec.ts`
- Modify: `dadamjang-be/test/graphql.integration-spec.ts`

- [ ] **Step 1: Write failing producer tests**

For every currently reachable real transition, assert exactly one app notification and eligible Outbox row after commit. Assert initial `PAYMENT_PENDING` creates none and a losing optimistic concurrent transition creates none. For likes, assert new non-self like creates one, self-like creates none, repeated like creates no additional row, and unlike/re-like remains lifetime-deduped.

```typescript
await transitionOrder({ orderId, nextStatus: "FULFILLING" });
expect(await notificationCount(`order:${orderId}:FULFILLING`)).toBe(1);
await likeStylePost(stylePostId, actorUserId);
await unlikeStylePost(stylePostId, actorUserId);
await likeStylePost(stylePostId, actorUserId);
expect(await notificationCount(`style-like:${stylePostId}:${actorUserId}`)).toBe(1);
```

- [ ] **Step 2: Run focused producer tests and observe RED**

```bash
pnpm test:integration -- --runTestsByPath test/admin.integration-spec.ts test/graphql.integration-spec.ts
```

- [ ] **Step 3: Inject and call the notification service inside existing transactions**

In `AdminService.transitionOrder`, call `createOrderStatus(tx, updated)` only after the optimistic update returned a row and before audit insertion/commit. Do not add `PAYMENT_PENDING -> PAID` to admin rules.

In `StylePostsService.setLikeState`, select the post author, add `.returning()` to the active-like insert, and call `createStyleLike` only when a new active row was returned and author differs from actor. Keep the existing advisory lock and soft-delete semantics.

```typescript
await this.notificationService.createOrderStatus(tx, {
  orderId: updated.orderId,
  status: input.nextStatus,
  userId: updated.userId,
});

const [insertedLike] = await tx
  .insert(stylePostLikes)
  .values({ stylePostId, userId })
  .onConflictDoNothing()
  .returning({ stylePostLikeId: stylePostLikes.stylePostLikeId });
if (insertedLike && post.userId !== userId)
  await this.notificationService.createStyleLike(tx, {
    actorUserId: userId,
    authorUserId: post.userId,
    stylePostId,
  });
```

- [ ] **Step 4: Re-run and commit**

```bash
pnpm test:integration -- --runTestsByPath test/admin.integration-spec.ts test/graphql.integration-spec.ts
git add src/modules/admin src/modules/style-posts test/admin.integration-spec.ts test/graphql.integration-spec.ts
git commit -m "feat(notification): emit order and style alerts"
```

---

### Task 5: Add the published Partner SKU mutation and wish producers

**Files:**
- Modify: `dadamjang-be/src/modules/partner/partner.module.ts`
- Modify: `dadamjang-be/src/modules/partner/partner.types.ts`
- Modify: `dadamjang-be/src/modules/partner/partner.resolver.ts`
- Modify: `dadamjang-be/src/modules/partner/partner.service.ts`
- Modify: `dadamjang-be/test/partner-catalog.integration-spec.ts`

**Input contract:**

```typescript
@InputType()
export class UpdatePublishedProductSkuInput {
  @Field(() => ID) skuId!: string;
  @Field(() => Int) price!: number;
  @Field(() => Int) stock!: number;
}

@InputType()
export class UpdatePublishedProductSkusInput {
  @Field(() => ID) productId!: string;
  @Field(() => [UpdatePublishedProductSkuInput]) skus!: UpdatePublishedProductSkuInput[];
}
```

- [ ] **Step 1: Write failing Partner integration tests**

Cover authorization, ownership, `status=PUBLISHED AND approvalStatus=APPROVED`, duplicate/missing/foreign SKU IDs, integer price/stock boundaries, and no changes to code/option/isActive or product metadata. Assert SKU IDs remain stable.

Seed wishers and assert:

- active minimum price decrease creates `WISH_PRICE_DROP`;
- equal/increased minimum creates none;
- active total stock `0 -> >0` creates `WISH_RESTOCK`;
- other stock transitions create none;
- one save can create both events;
- concurrent/retried identical saves dedupe;
- all notification and SKU changes roll back together on failure.

```typescript
const beforeSkuIds = await productSkuIds(productId);
await updatePublishedProductSkus({
  productId,
  skus: [
    { skuId: firstSkuId, price: 9000, stock: 2 },
    { skuId: secondSkuId, price: 12000, stock: 0 },
  ],
});
expect(await productSkuIds(productId)).toEqual(beforeSkuIds);
expect(await notificationTypesForProduct(productId)).toEqual(["WISH_PRICE_DROP", "WISH_RESTOCK"]);
```

- [ ] **Step 2: Run the test and observe RED**

```bash
pnpm test:integration -- --runTestsByPath test/partner-catalog.integration-spec.ts
```

- [ ] **Step 3: Implement in-place locked updates**

In one transaction:

1. Before opening the transaction, reject any price or stock that is not an integer or is below zero.
2. Lock the owned published/approved product and all of its SKU rows.
3. Validate that input contains every current SKU exactly once and no foreign SKU.
4. Snapshot active minimum price and active total stock.
5. Read one `transaction_timestamp()` and update only `price`, `stock`, `updatedAt` on existing rows.
6. Recompute boundaries, load wish recipient IDs, and create price/restock notifications before commit.

Do not call draft update, publish, or review methods. Do not delete/reinsert SKUs.

```typescript
const [product] = await tx
  .select({ productId: products.productId })
  .from(products)
  .where(
    and(
      eq(products.productId, input.productId),
      eq(products.partnerId, partner.partnerId),
      eq(products.brandId, partner.brandId),
      eq(products.status, "PUBLISHED"),
      eq(products.approvalStatus, "APPROVED"),
    ),
  )
  .for("update")
  .limit(1);
if (!product) throw new CustomBadRequestException(PartnerErrorMessage.InvalidTransition);
const currentSkus = await tx.select().from(productSkus).where(eq(productSkus.productId, input.productId)).for("update");
const nextById = new Map(input.skus.map((sku) => [sku.skuId, sku]));
if (
  input.skus.length !== currentSkus.length ||
  nextById.size !== currentSkus.length ||
  currentSkus.some(({ skuId }) => !nextById.has(skuId))
)
  throw new CustomBadRequestException(PartnerErrorMessage.InvalidProductInput);
const clock = requireResult((await tx.select({ updatedAt: sql<Date>`transaction_timestamp()` }))[0]);
const nextSkus = currentSkus.map((sku) => ({ ...sku, ...requireResult(nextById.get(sku.skuId)) }));
for (const sku of nextSkus)
  await tx
    .update(productSkus)
    .set({ price: sku.price, stock: sku.stock, updatedAt: clock.updatedAt })
    .where(eq(productSkus.skuId, sku.skuId));
```

Use one service-level guard before the transaction:

```typescript
if (input.skus.some(({ price, stock }) => !Number.isInteger(price) || price < 0 || !Number.isInteger(stock) || stock < 0))
  throw new CustomBadRequestException(PartnerErrorMessage.InvalidProductInput);
```

Calculate before/after minimum price and total stock from the locked `currentSkus` and `nextSkus`, considering only `isActive`. Load wish user IDs once and call the two producer methods only on the specified boundary crossings.

- [ ] **Step 4: Re-run and commit**

```bash
pnpm test:integration -- --runTestsByPath test/partner-catalog.integration-spec.ts test/notification.integration-spec.ts
git add src/modules/partner test/partner-catalog.integration-spec.ts
git commit -m "feat(partner): update published SKU inventory"
```

---

### Task 6: Expose price and stock only in the existing Partner editor

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-partner/src/_pages/product-editor/ui/product-editor-page.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-partner/src/shared/api/partner-contracts.ts`
- Modify: `dadamjang-fe/apps/dadamjang-partner/tests/unit/product-contract.test.ts`
- Modify: `dadamjang-fe/apps/dadamjang-partner/tests/e2e/partner-product-editor.spec.ts`

- [ ] **Step 1: Write failing contract and browser tests**

For a published product, assert title, description, images, category, SKU code/option, flags, add/remove SKU, submit, and publish controls are read-only or absent; price and stock remain editable. Saving must call only `updatePublishedProductSkus`. Draft and rejected products must continue calling only `updatePartnerProductDraft` with all existing fields editable.

```typescript
expect(PARTNER_PRODUCT_MUTATION_FIELDS.publishedInventory).toBe("updatePublishedProductSkus");
await expect(page.getByLabel("상품명")).toBeDisabled();
await expect(page.getByLabel("SKU 1 가격")).toBeEnabled();
await page.getByRole("button", { name: "저장" }).click();
expect(graphQlOperations).toEqual(["updatePublishedProductSkus"]);
```

- [ ] **Step 2: Run focused tests and observe RED**

From `dadamjang-fe`:

```bash
pnpm --dir apps/dadamjang-partner exec vitest run tests/unit/product-contract.test.ts
pnpm --dir apps/dadamjang-partner exec playwright test tests/e2e/partner-product-editor.spec.ts
```

- [ ] **Step 3: Branch the existing editor by persisted product state**

Keep one editor component. Build the published mutation input from existing SKU IDs plus integer price/stock values. Disable metadata and SKU-identity controls through existing field props; do not create a second inventory page or form abstraction.

```typescript
const isPublished = product?.status === "PUBLISHED";
const publishedSkus = isPublished
  ? skus.map(({ skuId, price, stock }) => {
      if (!skuId) throw new Error("게시 상품 SKU 정보가 올바르지 않습니다.");
      return { skuId, price, stock };
    })
  : [];
if (isPublished && !productId) throw new Error("게시 상품 정보가 올바르지 않습니다.");
const saved = isPublished
  ? await savePublishedProductSkus(productId, publishedSkus)
  : await saveProduct(productId, input);
const persisted =
  "updatePublishedProductSkus" in saved
    ? saved.updatePublishedProductSkus
    : "createPartnerProductDraft" in saved
      ? saved.createPartnerProductDraft
      : saved.updatePartnerProductDraft;
```

- [ ] **Step 4: Re-run and commit**

```bash
pnpm --dir apps/dadamjang-partner exec vitest run tests/unit/product-contract.test.ts
pnpm --dir apps/dadamjang-partner exec playwright test tests/e2e/partner-product-editor.spec.ts
git add apps/dadamjang-partner/src apps/dadamjang-partner/tests
git commit -m "feat(partner): edit published price and stock"
```

---

### Task 7: Send Expo Push and reconcile receipts

**Files:**
- Create: `dadamjang-be/src/modules/notification/notification.sender.ts`
- Create: `dadamjang-be/src/modules/notification/notification.sender.spec.ts`
- Create: `dadamjang-be/src/modules/notification/notification.outbox.ts`
- Create: `dadamjang-be/src/modules/notification/notification.outbox.spec.ts`
- Modify: `dadamjang-be/src/modules/notification/notification.repository.ts`
- Modify: `dadamjang-be/src/modules/notification/notification.module.ts`
- Modify: `dadamjang-be/.env.example`
- Modify: `dadamjang-be/test/support/env.ts`
- Modify: `dadamjang-be/test/notification.integration-spec.ts`

**Sender API:**

```typescript
send = async (messages: readonly ExpoPushMessage[]): Promise<readonly ExpoPushTicket[]>;
getReceipts = async (ticketIds: readonly string[]): Promise<Readonly<Record<string, ExpoPushReceipt>>>;
parseExpoPushTickets = (value: unknown, expectedCount: number): readonly ExpoPushTicket[];
```

Keep the Expo wire contracts local to `notification.sender.ts`:

```typescript
type ExpoPushMessage = Readonly<{
  to: string;
  title: string;
  body: string;
  data: Readonly<{ notificationId: string; type: FoNotificationTypeValue; entityId: string }>;
}>;

type ExpoPushTicket =
  | Readonly<{ status: "ok"; id: string }>
  | Readonly<{ status: "error"; message: string; details?: Readonly<{ error?: string }> }>;

type ExpoPushReceipt =
  | Readonly<{ status: "ok" }>
  | Readonly<{ status: "error"; message: string; details?: Readonly<{ error?: string }> }>;
```

- [ ] **Step 1: Write failing sender and worker tests**

In sender/worker unit tests, cover 100-message batching, 10-second timeout, non-JSON/error response validation, network/429/5xx exponential retry, permanent 4xx failure, claim-token ownership, ticket persistence, receipt scheduling at 15 minutes, receipt success, `DeviceNotRegistered`, eight-attempt terminal failure, and seven-day terminal cleanup.

In `test/notification.integration-spec.ts`, run two real repository workers concurrently against PostgreSQL and assert `FOR UPDATE SKIP LOCKED` claims every row once. Seed a 30-second stale `PROCESSING` row and assert one worker recovers it while the old claim token cannot complete it. Seed a disabled device with a pending row and assert it is terminally failed and never passed to the sender.

```typescript
await Promise.all([firstWorker.runOnce(), secondWorker.runOnce()]);
expect(sendRequestBodies.flat()).toHaveLength(100);
expect(new Set(sentPushOutboxIds).size).toBe(100);
clock.advanceBy(15 * 60_000);
await firstWorker.runOnce();
expect(await outboxStatus(pushOutboxId)).toBe("RECEIPT_OK");
expect(await activeDevice(invalidPushDeviceId)).toBe(false);
```

- [ ] **Step 2: Run focused unit tests and observe RED**

```bash
pnpm test:unit -- modules/notification/notification.sender.spec.ts modules/notification/notification.outbox.spec.ts
pnpm test:integration -- --runTestsByPath test/notification.integration-spec.ts
```

- [ ] **Step 3: Implement with Node fetch and the email Outbox constants**

POST to:

```text
https://exp.host/--/api/v2/push/send
https://exp.host/--/api/v2/push/getReceipts
```

Send data only as `{ notificationId, type, entityId }`. One one-second tick claims at most 100 sends or 1000 receipts; do not copy the email worker's drain loop. Persist ticket IDs before receipt polling. On `DeviceNotRegistered` from ticket or receipt, disable the device and terminally fail its unsettled rows.

```typescript
class RetryablePushError extends Error {
  constructor(readonly status?: number) {
    super(status ? `Expo Push retryable HTTP ${status}` : "Expo Push retryable response");
  }
}

class PermanentPushError extends Error {
  constructor(readonly status: number) {
    super(`Expo Push permanent HTTP ${status}`);
  }
}

send = async (messages: readonly ExpoPushMessage[]) => {
  const response = await fetch("https://exp.host/--/api/v2/push/send", {
    body: JSON.stringify(messages),
    headers: { Accept: "application/json", "Content-Type": "application/json" },
    method: "POST",
    signal: AbortSignal.timeout(10_000),
  });
  if (response.status === 429 || response.status >= 500) throw new RetryablePushError(response.status);
  if (!response.ok) throw new PermanentPushError(response.status);
  return parseExpoPushTickets(await response.json(), messages.length);
};
```

`parseExpoPushTickets` accepts only an object with a `data` array of exactly the request length and ticket entries whose `status` is `ok` with an `id` or `error` with Expo details; every other shape is a retryable malformed-response failure.

- [ ] **Step 4: Re-run and commit**

```bash
pnpm test:unit -- modules/notification/notification.sender.spec.ts modules/notification/notification.outbox.spec.ts
pnpm test:integration -- --runTestsByPath test/notification.integration-spec.ts
git add src/modules/notification .env.example test/support/env.ts test/notification.integration-spec.ts
git commit -m "feat(notification): deliver Expo Push receipts"
```

---

### Task 8: Verify the notification backend and Partner flow

- [ ] **Step 1: Run backend verification**

From `dadamjang-be`:

```bash
pnpm build
pnpm lint
pnpm test:unit
pnpm test:integration
node scripts/migrate-prod-smoke.mjs
pnpm db:test:down
```

- [ ] **Step 2: Run Partner verification**

From `dadamjang-fe`:

```bash
pnpm partner:typecheck
pnpm partner:lint
pnpm partner:fsd
pnpm partner:test
pnpm partner:build
pnpm --dir apps/dadamjang-partner exec playwright test tests/e2e/partner-product-editor.spec.ts
```

- [ ] **Step 3: Inspect intended diffs**

Run from the parent workspace:

```bash
git -C dadamjang-be status --short
git -C dadamjang-fe status --short
git diff --submodule=log
```

Expected result: all tests pass, both submodule worktrees are clean, and no package/lockfile change exists for the backend.
