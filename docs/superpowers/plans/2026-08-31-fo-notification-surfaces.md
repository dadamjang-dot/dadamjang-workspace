# FO Notification Surfaces Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Use build-ios-apps:ios-debugger-agent for a required native rebuild, then build-ios-apps:ios-simulator-browser and testing-react-native-apps:agent-device for Task 7 simulator verification.

**Goal:** Put the approved action buttons on every FO tab, implement their destinations, register Expo Push, and verify the complete flow on the iPhone 17 Pro simulator and Codex simulator browser.

**Architecture:** Reuse `ProductLayout`, `ActionButtonGroup`, `useAuthActionGate`, `ShopFiltersProvider`, and the existing auth/reset/session primitives. Add one notification feature for inbox/API/Push behavior and one settings feature for presentation. The server remains the source of allowed notification routes; the client additionally validates every external Push payload and route before navigation.

**Tech Stack:** Expo SDK 57, Expo Router, React Native, expo-notifications, expo-constants, TanStack Query, LegendList, Unistyles, Jest, iOS Simulator.

**Spec:** `docs/superpowers/specs/2026-08-31-fo-header-actions-notifications-design.md`

## Global Constraints

- Complete the account lifecycle and notification backend plans before this plan.
- Use arrow functions, kebab-case file names, `@/` imports across sibling folders, feature `index.ts` exports, and Unistyles only.
- Do not modify `ProductLayout`, `ActionButtonGroup`, or `@dadamjang/mobile`; their existing icon and variant contracts already cover the design.
- The supplied `a5c9073b433051088423626d67b84cadc797fd5a` object is not in the Dadamjang repositories; the local object with that SHA belongs to `shopport-fe` and its diff is unrelated CI/auth config. Treat the user's instruction as a strict scope rule, not an executable cherry-pick: apply only the exact FO action definitions in Task 6 and ignore every unrelated change. The current workspace already contains the shared `ProductLayout` and action-button implementation.
- Write and observe every focused regression test fail before implementation.
- Do not cast Push data or a server string directly to Expo Router `Href`.
- Do not persist Push-tap or reactivation tokens; keep pending IDs in provider/hook memory until authentication finishes.
- Push registration failure never fails login. Retry on the next authenticated foreground.
- Reuse the running Metro server for JavaScript-only iterations. After adding `expo-notifications`, one native project sync and dev-client rebuild is mandatory because the current iOS Podfile lock does not contain `ExpoNotifications`; stop Metro before syncing/building and invoke the required iOS build skill.
- Commit frontend changes inside `dadamjang-fe`, then commit both submodule pointers and all plan documents in the parent repository.

## File Structure

| Unit | Responsibility |
| --- | --- |
| Five existing tab route files | Selective action-definition and destination wiring only |
| `features/shop/components/shop-menu-sheet.tsx` | Existing category/filter provider presented as the shopping menu |
| `features/notification/api.ts` and `types.ts` | Exact GraphQL operations and client contracts |
| `features/notification/hooks.ts` and `rules.ts` | Query mutations, cursor state, payload/route validation |
| `features/notification/push.ts` | Permission, token registration, listeners, cold/warm response handling |
| `features/notification/components/notification-inbox.tsx` | Inbox loading, error, empty, pagination, and row interactions |
| `features/settings/components/*` | Public app info plus authenticated preference/account controls |
| Thin Expo Router files under `src/app` | Route composition only; feature logic stays in feature folders |
| Focused Jest/native-action tests named by each task | Header, navigation, Push, inbox, settings, and lifecycle regression proof |

## Header Acceptance Matrix

| Tab | First action | Second action | Variant |
| --- | --- | --- | --- |
| Home | 알림 → `/notifications` | 장바구니 → `/cart` | `capsule` |
| Style | 등록 → `/style-compose` | 장바구니 → `/cart` | `circularPair` |
| Shop | 쇼핑 메뉴 → `/shop-menu-sheet` | 장바구니 → `/cart` | `capsule` |
| Wish | 없음 | 장바구니 → `/cart` | single existing action |
| My | 설정 → `/settings` | 장바구니 → `/cart` | `circularPair` |

`ProductLayout` remains limited to Home, Style, and Shop. Wish and My retain their current screen layouts.

## Client Contracts

```typescript
export type FoNotificationType =
  | "ORDER_STATUS"
  | "WISH_PRICE_DROP"
  | "WISH_RESTOCK"
  | "STYLE_LIKE";

export interface FoNotification {
  notificationId: string;
  type: FoNotificationType;
  title: string;
  body: string;
  route: string;
  entityId: string;
  readAt: string | null;
  createdAt: string;
}

export interface FoNotificationConnection {
  nodes: FoNotification[];
  nextCursor: string | null;
  hasNextPage: boolean;
  unreadCount: number;
}

export interface FoNotificationPreferences {
  pushEnabled: boolean;
  orderPushEnabled: boolean;
  wishPushEnabled: boolean;
  stylePushEnabled: boolean;
  updatedAt: string;
}

export interface FoPushNotificationData {
  notificationId: string;
  type: FoNotificationType;
  entityId: string;
}
```

---

### Task 1: Add and configure Expo Notifications

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-fo/package.json`
- Modify: `dadamjang-fe/pnpm-lock.yaml`
- Modify: `dadamjang-fe/apps/dadamjang-fo/app.config.js`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/mocks/expo-notifications.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/mocks/expo-constants.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/setup.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/api-contracts.test.ts`

- [ ] **Step 1: Write a failing configuration contract test**

Assert FO declares `expo-notifications` and `expo-constants` directly, `app.config.js` includes the `expo-notifications` plugin, and `extra.eas.projectId` remains `095bcf9d-2bf8-4274-bb83-838d70c4f608`.

```typescript
expect(packageJson.dependencies).toEqual(
  expect.objectContaining({ "expo-constants": expect.any(String), "expo-notifications": expect.any(String) }),
);
expect(appConfig.plugins).toContain("expo-notifications");
expect(appConfig.extra.eas.projectId).toBe("095bcf9d-2bf8-4274-bb83-838d70c4f608");
```

- [ ] **Step 2: Run the contract test and observe RED**

From `dadamjang-fe`:

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/unit/api-contracts.test.ts
```

- [ ] **Step 3: Install SDK-compatible packages and configure the plugin**

```bash
pnpm --dir apps/dadamjang-fo exec expo install expo-notifications expo-constants
```

Add `"expo-notifications"` to the existing plugin array without changing the EAS project ID or deployment target. Add deterministic Jest mocks for permissions, token fetch, listeners, last response, handler setup, and Android channel setup.

```javascript
plugins: [...existingPlugins, "expo-notifications"],
extra: { ...existingExtra, eas: { ...existingExtra.eas, projectId: "095bcf9d-2bf8-4274-bb83-838d70c4f608" } },
```

- [ ] **Step 4: Verify config and commit**

```bash
pnpm --dir apps/dadamjang-fo verify:config
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/unit/api-contracts.test.ts
git add apps/dadamjang-fo/package.json apps/dadamjang-fo/app.config.js apps/dadamjang-fo/__tests__ pnpm-lock.yaml
git commit -m "chore(fo): configure Expo notifications"
```

---

### Task 2: Implement the shopping menu sheet

**Files:**
- Create: `dadamjang-fe/apps/dadamjang-fo/src/app/shop-menu-sheet.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/shop/components/shop-menu-sheet.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/shop/components/index.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/shop/index.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/_layout.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/shop-menu-sheet.test.tsx`

**Component interface:**

```typescript
export interface ShopMenuSheetProps {
  categories: readonly Category[];
  isError: boolean;
  isLoading: boolean;
  selectedCategoryId?: string;
  onOpenComparison: () => void;
  onRetry: () => void;
  onSelectCategory: (categoryId?: string) => void;
}
```

- [ ] **Step 1: Write failing sheet tests**

Assert loading, retryable error, all-category option, every category level ordered by `sortOrder`, current selection, category selection, and compare navigation. Selection must apply `{ categoryId, categoryIds: [], categorySource }` to the existing provider and close the sheet; compare must replace the sheet with `/compare`.

```typescript
fireEvent.press(screen.getByText("전체"));
expect(updateFilters).toHaveBeenCalledWith({ categoryId: undefined, categoryIds: [], categorySource: undefined });
expect(router.back).toHaveBeenCalled();
fireEvent.press(screen.getByText("비교함"));
expect(router.replace).toHaveBeenCalledWith("/compare");
```

- [ ] **Step 2: Run the test and observe RED**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/shop-menu-sheet.test.tsx
```

- [ ] **Step 3: Build the sheet from existing category/filter primitives**

Use `useCategories()` and `useShopFilters()` in the route. Register the route using the same form-sheet options as filter/sort sheets. Reuse current typography, button, loading, and error-state components; do not turn `ShopCategoryBar` into a second mode or create a global menu system.

```typescript
const selectCategory = (categoryId?: string) => {
  updateFilters({ categoryId, categoryIds: [], categorySource: categoryId ? "navigation" : undefined });
  router.back();
};
const openComparison = () => router.replace("/compare");
```

- [ ] **Step 4: Re-run and commit**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/shop-menu-sheet.test.tsx
git add apps/dadamjang-fo/src/app/shop-menu-sheet.tsx apps/dadamjang-fo/src/app/_layout.tsx apps/dadamjang-fo/src/features/shop apps/dadamjang-fo/__tests__/integration/shop-menu-sheet.test.tsx
git commit -m "feat(fo): add shopping menu sheet"
```

---

### Task 3: Implement the in-app notification inbox

**Files:**
- Create: `dadamjang-fe/apps/dadamjang-fo/src/app/notifications.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/api.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/hooks.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/types.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/rules.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/components/notification-inbox.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/components/index.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/index.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/notification-inbox.test.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/notification-rules.test.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/api-contracts.test.ts`

**Hooks:**

```typescript
useFoNotifications = (): UseInfiniteQueryResult<FoNotificationConnection>;
useFoNotification = (notificationId: string | undefined): UseQueryResult<FoNotification>;
useMarkFoNotificationRead = (): UseMutationResult<FoNotification, Error, string>;
useMarkAllFoNotificationsRead = (): UseMutationResult<boolean, Error, void>;
```

- [ ] **Step 1: Write failing API, rules, and screen tests**

Assert GraphQL selection sets match the backend contract. Validate only these route/type pairs:

```typescript
ORDER_STATUS -> /order/${entityId}
WISH_PRICE_DROP | WISH_RESTOCK -> /product/${entityId}
STYLE_LIKE -> /style/${entityId}
```

Assert newest-first rows, unread treatment, pagination, loading, empty, retryable error, individual read, all-read, and safe navigation. A missing/forbidden destination or mismatched route must replace with `/notifications` and keep the app usable.

```typescript
expect(isAllowedNotificationRoute({ type: "ORDER_STATUS", entityId: orderId, route: `/order/${orderId}` })).toBe(true);
expect(isAllowedNotificationRoute({ type: "ORDER_STATUS", entityId: orderId, route: "https://example.test" })).toBe(false);
fireEvent.press(screen.getByText("모두 읽음"));
expect(markAllRead).toHaveBeenCalled();
```

- [ ] **Step 2: Run focused tests and observe RED**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/unit/api-contracts.test.ts __tests__/unit/notification-rules.test.ts __tests__/integration/notification-inbox.test.tsx
```

- [ ] **Step 3: Implement the feature using current query and list patterns**

Use the existing GraphQL request helper, TanStack query keys, `LegendList`, title header, button, and error/empty primitives. On row press, mark read, fetch/confirm the singular notification when necessary, validate its type/entity/route, then navigate. Invalidate list and unread count after mutations. Do not add a global event bus or normalized cache.

```typescript
const allowedRouteFor = ({ entityId, type }: Pick<FoNotification, "entityId" | "type">) =>
  type === "ORDER_STATUS"
    ? `/order/${entityId}`
    : type === "STYLE_LIKE"
      ? `/style/${entityId}`
      : `/product/${entityId}`;
const isAllowedNotificationRoute = (notification: FoNotification) =>
  notification.route === allowedRouteFor(notification);
```

- [ ] **Step 4: Re-run and commit**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/unit/api-contracts.test.ts __tests__/unit/notification-rules.test.ts __tests__/integration/notification-inbox.test.tsx
git add apps/dadamjang-fo/src/app/notifications.tsx apps/dadamjang-fo/src/features/notification apps/dadamjang-fo/__tests__
git commit -m "feat(fo): add notification inbox"
```

---

### Task 4: Register Push and handle cold, warm, and foreground notifications

**Files:**
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/push.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/api.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/hooks.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/index.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/providers/app-providers.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/push-notifications.test.tsx`

**Notification handler:**

```typescript
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: false,
    shouldSetBadge: false,
    shouldShowBanner: true,
    shouldShowList: true,
  }),
});
```

- [ ] **Step 1: Write failing Push lifecycle tests**

Cover undetermined/granted/denied permissions, Android channel-before-token order, EAS project ID forwarding, iOS/Android enum mapping, authenticated registration, retryable registration failure on foreground, terminal `false` registration suppression for the unchanged token, no login failure on registration error, listener cleanup, foreground display handler, warm tap, cold last response, duplicate response suppression, malformed payload rejection, unauthenticated pending tap, post-auth resolution through `foNotification`, and forbidden/deleted fallback.

On iOS, treat `IosAuthorizationStatus.AUTHORIZED`, `PROVISIONAL`, and `EPHEMERAL` as allowed. When the top-level permission status is `undetermined`, call `requestPermissionsAsync()` once and evaluate the returned iOS authorization status. A denied status never requests again automatically.

```typescript
mockGetPermissions.mockResolvedValue({ status: "undetermined" });
mockRequestPermissions.mockResolvedValue({
  granted: false,
  ios: { status: Notifications.IosAuthorizationStatus.PROVISIONAL },
});
await bootstrapPush();
expect(mockRequestPermissions).toHaveBeenCalledTimes(1);
expect(registerFoPushDevice).toHaveBeenCalledWith(expect.objectContaining({ platform: "IOS" }));
```

- [ ] **Step 2: Run the test and observe RED**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/push-notifications.test.tsx
```

- [ ] **Step 3: Implement one provider-mounted Push bootstrap**

Mount `useFoPushNotifications()` inside the existing provider tree after auth/session state is available. Before token fetch on Android, create the default high-importance channel. Read `Constants.expoConfig?.extra?.eas?.projectId`; if absent, report a retryable configuration error without crashing login.

Call `getExpoPushTokenAsync({ projectId })` only for authenticated users with granted permission. Register token/platform through the backend; the GraphQL client already sends the stable device header. On app foreground, retry network/server failures if registration has not succeeded for the current token/session. If the backend returns `false` because Expo permanently invalidated that exact token, remember the rejected token in memory and do not retry it until Expo returns a different token or the auth session changes.

```typescript
const hasIosAuthorization = (status: Notifications.IosAuthorizationStatus | undefined) =>
  status === Notifications.IosAuthorizationStatus.AUTHORIZED ||
  status === Notifications.IosAuthorizationStatus.PROVISIONAL ||
  status === Notifications.IosAuthorizationStatus.EPHEMERAL;

const ensureNotificationPermission = async () => {
  const current = await Notifications.getPermissionsAsync();
  const permission = current.status === "undetermined" ? await Notifications.requestPermissionsAsync() : current;
  return permission.granted || hasIosAuthorization(permission.ios?.status);
};
```

For response handling, accept only runtime-validated `{ notificationId, type, entityId }`, fetch `foNotification`, validate its allowed route, mark read, and navigate. Hold an unauthenticated `notificationId` in memory and resume after auth.

- [ ] **Step 4: Re-run and commit**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/push-notifications.test.tsx __tests__/integration/notification-inbox.test.tsx
git add apps/dadamjang-fo/src/features/notification apps/dadamjang-fo/src/providers/app-providers.tsx apps/dadamjang-fo/__tests__/integration/push-notifications.test.tsx
git commit -m "feat(fo): register and route Expo Push"
```

---

### Task 5: Implement Settings, password reset, logout, and deactivation

**Files:**
- Create: `dadamjang-fe/apps/dadamjang-fo/src/app/settings/index.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/app/settings/password.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/(tabs)/my.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/settings/components/settings-screen.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/settings/components/password-change-screen.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/settings/components/index.ts`
- Create: `dadamjang-fe/apps/dadamjang-fo/src/features/settings/index.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/api.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/notification/hooks.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/api.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/hooks.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/types.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/components/password-reset-form.tsx`
- Create: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/settings.test.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/account-recovery.test.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/api-contracts.test.ts`

- [ ] **Step 1: Write failing public/authenticated settings tests**

Public settings shows `Constants.expoConfig?.version` and gates account/Push actions through login. Authenticated settings shows email, OS permission status, overall/order/wish/style toggles, and logout. It shows deactivation only for role `USER`, because the lifecycle contract deliberately rejects Partner business accounts. It shows password change only when `hasPassword=true`; Kakao-only users must not see it. Permanently denied notification permission exposes `Linking.openSettings()`.

Assert preference update rollback/error behavior, exact deactivation confirmation, active-order server error, successful session clearing, and scheduled date copy. Assert logout uses the existing server logout before local reset so the backend disables the current device.

```typescript
expect(screen.getByText("앱 버전 1.0.0")).toBeTruthy();
expect(screen.queryByText("비밀번호 변경")).toBeNull();
fireEvent.press(screen.getByText("로그아웃"));
expect(logoutFo).toHaveBeenCalled();
expect(logoutFo.mock.invocationCallOrder[0]).toBeLessThan(resetAuthSession.mock.invocationCallOrder[0]);
```

- [ ] **Step 2: Write failing authenticated password-reset tests**

Reuse the existing request-code, verify-code, and reset mutations with the authenticated email fixed/read-only. On completion, call `resetAuthSession()` and replace with `/auth/signin`; do not call the regular logout mutation after reset has already revoked all refresh sessions.

```typescript
expect(screen.getByDisplayValue("member@example.test").props.editable).toBe(false);
await completePasswordReset();
expect(logoutFo).not.toHaveBeenCalled();
expect(resetAuthSession).toHaveBeenCalled();
expect(router.replace).toHaveBeenCalledWith("/auth/signin");
```

- [ ] **Step 3: Run focused tests and observe RED**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/settings.test.tsx __tests__/integration/account-recovery.test.tsx __tests__/unit/api-contracts.test.ts
```

- [ ] **Step 4: Implement settings with existing primitives**

Keep settings routes as presentation wrappers and feature API calls in their owning `auth` or `notification` feature. The Settings screen itself remains public so app version is always visible; use the existing auth gate only when a protected account/Push row is pressed, preserving `/settings` as the safe return path. Reuse existing `TitleHeader`, `Button`, and Unistyles tokens. Move the existing authenticated 주문 내역 action into the My body and move logout from My into Settings; Task 6 changes only the My header action array. Do not create a settings state store.

```typescript
const showPasswordChange = viewer?.hasPassword === true;
const showDeactivation = viewer?.role === "USER";
const logout = async () => {
  await logoutFo();
  await resetAuthSession();
  router.replace("/");
};
```

Native deactivation confirmation copy:

```text
계정을 탈퇴할까요?
30일 안에는 다시 로그인해 계정을 복구할 수 있어요. 이후에는 계정 정보가 익명화되어 복구할 수 없어요.
```

Buttons: `취소`, `탈퇴하기` with destructive style on the latter.

- [ ] **Step 5: Re-run and commit**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/settings.test.tsx __tests__/integration/account-recovery.test.tsx __tests__/unit/api-contracts.test.ts
git add apps/dadamjang-fo/src/app/settings 'apps/dadamjang-fo/src/app/(tabs)/my.tsx' apps/dadamjang-fo/src/features/settings apps/dadamjang-fo/src/features/auth apps/dadamjang-fo/src/features/notification apps/dadamjang-fo/__tests__
git commit -m "feat(fo): add notification and account settings"
```

---

### Task 6: Selectively apply only the five approved header action sets

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/(tabs)/index.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/(tabs)/style.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/(tabs)/shop.tsx`
- Verify only: `dadamjang-fe/apps/dadamjang-fo/src/app/(tabs)/wish.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/app/(tabs)/my.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/features/auth/rules.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/route-completeness.test.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/protected-auth-actions.test.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/auth-rules.test.ts`
- Modify: `dadamjang-fe/apps/dadamjang-fo/android-tests/native-actions.test.tsx`

**Exact action definitions:**

```typescript
const homeActions: IconAction[] = [
  { accessibilityLabel: "알림", icon: { md: "notifications", sf: "bell" }, onPress: openNotifications },
  { accessibilityLabel: "장바구니", icon: { md: "shopping_cart", sf: "cart" }, onPress: openCart },
];
const styleActions: [IconAction, IconAction] = [
  { accessibilityLabel: "스타일 등록", icon: { md: "add", sf: "plus" }, onPress: openStyleCompose },
  { accessibilityLabel: "장바구니", icon: { md: "shopping_cart", sf: "cart" }, onPress: openCart },
];
const shopActions: IconAction[] = [
  { accessibilityLabel: "쇼핑 메뉴", icon: { md: "menu", sf: "line.3.horizontal" }, onPress: openShopMenu },
  { accessibilityLabel: "장바구니", icon: { md: "shopping_cart", sf: "cart" }, onPress: openCart },
];
const myActions: [IconAction, IconAction] = [
  { accessibilityLabel: "설정", icon: { md: "settings", sf: "gearshape" }, onPress: openSettings },
  { accessibilityLabel: "장바구니", icon: { md: "shopping_cart", sf: "cart" }, onPress: openCart },
];
```

Wish keeps its existing single `{ accessibilityLabel: "장바구니", icon: { md: "shopping_cart", sf: "cart" } }` action.

- [ ] **Step 1: Write failing header matrix tests**

Render every tab and capture the props passed to the existing `ProductLayout`, `ActionButtonGroup`, or `ActionButton`. Assert the definitions above, route order, and variants. Assert `ProductLayout` is rendered only by Home, Style, and Shop. Assert My renders Settings and Cart actions in both authenticated and unauthenticated states. For unauthenticated Home notifications, assert the existing sanitized auth route preserves `returnTo=/notifications`.

```typescript
expect(homeLayoutProps).toMatchObject({ headerActions: homeActions, variant: "capsule" });
expect(styleLayoutProps).toMatchObject({ headerActions: styleActions, variant: "circularPair" });
expect(shopLayoutProps).toMatchObject({ headerActions: shopActions, variant: "capsule" });
expect(myActionGroupProps).toMatchObject({ actions: myActions, variant: "circularPair" });
expect(wishActionProps.actions).toHaveLength(1);
```

- [ ] **Step 2: Run focused native and JavaScript tests and observe RED**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/route-completeness.test.tsx __tests__/integration/protected-auth-actions.test.tsx __tests__/unit/auth-rules.test.ts
pnpm --dir apps/dadamjang-fo exec jest --config jest.android.config.ts --runInBand --runTestsByPath android-tests/native-actions.test.tsx
```

- [ ] **Step 3: Change only the tab action definitions and safe return paths**

Do not cherry-pick `a5c9073b433051088423626d67b84cadc797fd5a` and do not copy its auth, config, CI, or test changes. The shared `ProductLayout`, `ActionButtonGroup`, `ActionButton`, and mobile package remain untouched. In Home only, replace the existing style shortcut with the notification action:

```typescript
const notificationsGate = useAuthActionGate("/notifications");
const openNotifications = () => notificationsGate.runProtectedAction(() => router.push("/notifications"));
```

Set Shop's first action to `router.push("/shop-menu-sheet")`. Render My's action group unconditionally, with its first action calling `router.push("/settings")`; protected rows inside Settings own their auth gate. Change Style's existing accessibility label from `스타일 작성` to the approved `스타일 등록` and otherwise retain its action behavior. Leave Wish production code unchanged because it already matches. Add `/notifications`, `/settings`, and `/settings/password` to existing safe return paths. Orders and logout were already relocated in Task 5 and are not part of this action-only edit.

- [ ] **Step 4: Re-run and commit**

```bash
pnpm --dir apps/dadamjang-fo exec jest --runInBand --runTestsByPath __tests__/integration/route-completeness.test.tsx __tests__/integration/protected-auth-actions.test.tsx __tests__/unit/auth-rules.test.ts
pnpm --dir apps/dadamjang-fo exec jest --config jest.android.config.ts --runInBand --runTestsByPath android-tests/native-actions.test.tsx
git add 'apps/dadamjang-fo/src/app/(tabs)' apps/dadamjang-fo/src/features/auth/rules.ts apps/dadamjang-fo/__tests__ apps/dadamjang-fo/android-tests
git commit -m "feat(fo): wire tab header actions"
```

---

### Task 7: Verify FO and the real iPhone 17 Pro flow

- [ ] **Step 1: Run static and automated FO verification**

From `dadamjang-fe`:

```bash
pnpm fo:typecheck
pnpm fo:lint
pnpm fo:test:unit
pnpm fo:test:integration
pnpm fo:test:android
pnpm --dir apps/dadamjang-fo verify:config
pnpm fo:verify:autolinking
pnpm fo:verify:android-export
```

- [ ] **Step 2: Reload the existing dev client first**

Reuse iPhone 17 Pro simulator UDID `50E71B7B-ADD4-4C54-8F17-DB947D537333`. Stop Metro on port 8081, then sync the generated iOS project and Pods from `dadamjang-fe`:

```bash
pnpm --dir apps/dadamjang-fo exec expo prebuild --platform ios
rg 'ExpoNotifications' apps/dadamjang-fo/ios/Podfile.lock
plutil -extract aps-environment raw -o - apps/dadamjang-fo/ios/app/app.entitlements
```

The prebuild must complete successfully, `Podfile.lock` must contain `ExpoNotifications`, and the generated `aps-environment` value must be `development`; do not hand-edit ignored generated native files to fake either result.

Invoke `build-ios-apps:ios-debugger-agent`: list simulators and select the booted iPhone 17 Pro, set session defaults to workspace `dadamjang-fe/apps/dadamjang-fo/ios/app.xcworkspace`, scheme `app`, Debug configuration, and the fixed UDID, then call its simulator build-and-run operation. After success, use its UI description or screenshot to prove `com.dadamjang.fo` launched. Restart Metro with `pnpm --dir apps/dadamjang-fo exec expo start --dev-client`, then invoke `build-ios-apps:ios-simulator-browser` to expose the launched app in Codex. Do not run an unrelated full build while Metro is active.

- [ ] **Step 3: Verify screens through the Codex simulator browser**

Use `build-ios-apps:ios-simulator-browser` and the existing `http://localhost:3200/` mirror. Check every header matrix row, auth return, shopping menu category/compare behavior, notification inbox states, settings visibility, password warning, and deactivation/reactivation confirmation. Capture evidence only after the UI is stable.

- [ ] **Step 4: Verify actual Expo Push end to end**

Confirm the simulator is iOS 16+ and the dev build uses the configured EAS project/APNs credentials. Resolve the process listening on port 5500 with `lsof -nP -iTCP:5500 -sTCP:LISTEN`, stop that backend gracefully in its owning terminal, confirm the port is free, then start the implemented backend from `dadamjang-be` in a long-lived terminal:

```bash
PORT=5500 PUSH_OUTBOX_WORKER_ENABLED=true pnpm start:dev
```

In another terminal, require readiness before continuing:

```bash
curl -fsS http://127.0.0.1:5500/health/ready | jq -e '.status == "ok"'
```

This restart guarantees exactly the intended local worker owns the Outbox instead of starting a conflicting second API replica. Sign into the FO simulator as `integration@example.test` / `IntegrationPassword123!`, grant notification permission, and bring the app to foreground so the fixture user's device is registered.

Preflight the non-destructive fixture state from the parent workspace. The order must still be `PAID` and the user must have one active Push device. If either check fails, stop and report the state instead of resetting production-like data by hand.

```bash
PUSH_DB_PASSWORD=postgres
PGPASSWORD="$PUSH_DB_PASSWORD" psql -h 127.0.0.1 -p 5432 -U postgres -d dadamjang -v ON_ERROR_STOP=1 -c \
  'SELECT "orderId", status, "paymentStatus" FROM orders WHERE "orderId" = '\''91000000-0000-4000-8000-000000000001'\'';'
PGPASSWORD="$PUSH_DB_PASSWORD" psql -h 127.0.0.1 -p 5432 -U postgres -d dadamjang -v ON_ERROR_STOP=1 -c \
  'SELECT "installationId", "expoPushToken", "disabledAt" FROM "pushDevices" WHERE "userId" = '\''10000000-0000-4000-8000-000000000001'\'';'
```

Obtain an admin token through the existing BO sign-in and trigger the real `PAID -> FULFILLING` writer:

```bash
PUSH_GRAPHQL_URL=http://127.0.0.1:5500/graphql
PUSH_ADMIN_ACCESS_TOKEN="$(curl -sS "$PUSH_GRAPHQL_URL" \
  -H 'Content-Type: application/json' \
  -H 'x-device-id: push-verification-admin' \
  --data '{"query":"mutation Signin($input: SigninAuthInput!) { signin(input: $input) { accessToken } }","variables":{"input":{"userid":"integration-admin","password":"IntegrationAdmin123!","portal":"BO"}}}' \
  | jq -er '.data.signin.accessToken')"
curl -sS "$PUSH_GRAPHQL_URL" \
  -H 'Content-Type: application/json' \
  -H 'x-device-id: push-verification-admin' \
  -H "Authorization: Bearer $PUSH_ADMIN_ACCESS_TOKEN" \
  --data '{"query":"mutation Transition($input: TransitionOrderInput!) { transitionOrder(input: $input) { orderId status } }","variables":{"input":{"orderId":"91000000-0000-4000-8000-000000000001","nextStatus":"FULFILLING"}}}' \
  | jq -e '.data.transitionOrder.status == "FULFILLING"'
```

Query the durable evidence immediately, then re-run the same query in bounded checks no more frequently than once per minute until receipt reconciliation completes. Keep the user updated between checks; do not use a blocking wait longer than 60 seconds.

```bash
PGPASSWORD="$PUSH_DB_PASSWORD" psql -h 127.0.0.1 -p 5432 -U postgres -d dadamjang -v ON_ERROR_STOP=1 -c \
  'SELECT n.type, n."dedupeKey", n."readAt", o.status, o."expoTicketId", o."lastError" FROM notifications n JOIN "pushOutbox" o ON o."notificationId" = n."notificationId" WHERE n."dedupeKey" = '\''order:91000000-0000-4000-8000-000000000001:FULFILLING'\'';'
```

Require all of:

1. `notifications` row committed.
2. `pushOutbox` reaches `TICKETED`.
3. Expo receipt reaches `RECEIPT_OK`.
4. iPhone 17 Pro receives the remote Push.
5. Tapping it opens the validated detail route and marks the row read.

After tapping, query the matching notification's `readAt` and confirm it is non-null. If credentials are absent or Expo returns a credential error, report that external configuration explicitly. Do not substitute a local notification or a mocked ticket as success.

- [ ] **Step 5: Commit final frontend verification changes**

```bash
git status --short
git add apps/dadamjang-fo pnpm-lock.yaml
git commit -m "test(fo): cover header and Push flows"
```

Skip this commit if verification produced no file changes.

---

### Task 8: Final cross-repository verification and parent commit

- [ ] **Step 1: Confirm both submodules are clean and on the feature branch**

```bash
git -C dadamjang-be branch --show-current
git -C dadamjang-be status --short
git -C dadamjang-fe branch --show-current
git -C dadamjang-fe status --short
```

Expected branch: `feat/fo-notifications`; expected status: clean.

- [ ] **Step 2: Run the smallest cross-stack smoke**

```bash
git -C dadamjang-be log -1 --oneline
git -C dadamjang-fe log -1 --oneline
git diff --submodule=log
```

Confirm the diff contains only the approved docs and the two submodule pointers.

- [ ] **Step 3: Commit the parent integration**

```bash
git add dadamjang-be dadamjang-fe docs/superpowers/specs/2026-08-31-fo-header-actions-notifications-design.md docs/superpowers/plans/2026-08-31-fo-account-lifecycle.md docs/superpowers/plans/2026-08-31-commerce-notification-backend.md docs/superpowers/plans/2026-08-31-fo-notification-surfaces.md
git commit -m "feat(fo): integrate commerce notifications"
```

- [ ] **Step 4: Final evidence check**

```bash
git status --short
git log --oneline -5
```

Expected result: clean parent worktree with the spec, all three plans, and updated submodule pointers committed.
