# FO 헤더 액션·알림·설정 설계

## 목표

FO의 기존 헤더 액션 UI를 재사용해 탭별 액션을 정확한 화면에 연결하고, 실제 쇼핑 이벤트를 앱 내 알림과 Expo Push로 전달한다. 쇼핑 메뉴, 설정, 비밀번호 모델, 30일 탈퇴 유예까지 액션의 모든 목적지를 완성한다.

## 범위

### 포함

- 홈, 스타일, 쇼핑, 위시, 마이 헤더 액션 연결
- 앱 내 알림함과 Expo Push 발송
- 주문 상태, 위시 상품, 스타일 좋아요 알림 생성
- 쇼핑 카테고리 전체 메뉴 시트
- 푸시 설정과 계정 설정
- Partner 게시 상품의 가격·재고 수정 경로
- 카카오 전용 계정의 nullable 비밀번호 모델
- 비밀번호 찾기 진입 경고
- 30일 탈퇴 유예, 복구, 익명화

### 제외

- 쿠폰, 프로모션, 배송 추적 알림
- BO 화면 변경
- 새로운 헤더 컴포넌트나 디자인 시스템
- 알림 마케팅 캠페인 도구
- APNs 직접 발송 경로

BO의 주문 상태 변경 화면은 유지한다. Partner는 기존 상품 편집기를 재사용하되 게시 상품에서는 가격·재고만 수정할 수 있고, 해당 백엔드 쓰기 흐름이 위시 알림 생성 지점이 된다.

## 기존 UI 재사용

현재 `dadamjang-fe`의 `ProductLayout`과 `ActionButton`을 그대로 사용한다. 별도의 헤더 액션 추상화는 만들지 않는다.

사용자가 지정한 `a5c9073b433051088423626d67b84cadc797fd5a` 기준에서는 액션 버튼 관련 설정만 선별해 적용한다. 커밋 전체를 cherry-pick하거나 인증, 설정, CI 등 무관한 변경을 가져오지 않는다. 현재 워크스페이스에 공용 버튼 구현이 이미 있으므로 공용 컴포넌트는 수정하지 않고 각 FO 탭의 액션 정의와 목적지 연결만 바꾼다.

| 탭 | 레이아웃 | 액션 | 형태 |
| --- | --- | --- | --- |
| 홈 | `ProductLayout` | 알림, 장바구니 | 캡슐 |
| 스타일 | `ProductLayout` | 등록, 장바구니 | 원형 2개 |
| 쇼핑 | `ProductLayout` | 메뉴, 장바구니 | 캡슐 |
| 위시 | 기존 화면 레이아웃 | 장바구니 | 단일 원형 |
| 마이 | 기존 화면 레이아웃 | 설정, 장바구니 | 원형 2개 |

비로그인 상태에서도 액션은 같은 위치에 보인다. 인증이 필요한 액션은 기존 `returnTo` 기반 인증 게이트를 사용한다.

## 선택한 알림 아키텍처

DB 알림함과 트랜잭션 Outbox를 사용한다. 도메인 변경과 알림 생성은 같은 데이터베이스 트랜잭션에서 커밋하고, Expo Push는 워커가 비동기로 발송한다.

이 방식은 Expo 장애가 주문, 상품, 좋아요 처리를 지연하거나 롤백하지 않게 하며, 앱 내 기록과 Push 재시도를 함께 보장한다. 동기 Expo 호출과 사후 폴링 방식은 각각 도메인 요청 결합, 누락과 중복 위험 때문에 사용하지 않는다.

백엔드는 새 Push 프레임워크를 만들지 않고 기존 이메일 Outbox의 `FOR UPDATE SKIP LOCKED`, claim token, 재시도, 만료 처리 패턴을 재사용한다. Expo API 호출은 Node의 `fetch`를 사용하며 별도 서버 SDK를 추가하지 않는다.

## 데이터 모델

### `notifications`

- `notificationId`: UUID 기본 키
- `userId`: 수신 사용자
- `type`: `ORDER_STATUS`, `WISH_PRICE_DROP`, `WISH_RESTOCK`, `STYLE_LIKE`
- `title`, `body`: 앱 내 표시와 Push 본문
- `route`: 허용된 FO 내부 경로
- `entityId`: 주문, 상품 또는 스타일 게시물 ID
- `dedupeKey`: 이벤트별 고유 키
- `readAt`, `createdAt`

`userId + dedupeKey`를 고유하게 만들어 트랜잭션 재시도와 중복 도메인 요청이 같은 알림을 두 번 만들지 못하게 한다. `route`는 임의 URL이 아니라 알림 타입별 서버 매핑으로 생성한다.

키 형식은 주문 상태 `order:<orderId>:<status>`, 위시 가격 `wish-price:<productId>:<skuUpdatedAt>:<newPrice>`, 재입고 `wish-stock:<productId>:<skuUpdatedAt>`, 스타일 좋아요 `style-like:<stylePostId>:<actorUserId>`다. SKU 갱신 시각은 같은 상품 수정 트랜잭션의 확정된 DB 시각을 사용한다.

### `pushDevices`

- `pushDeviceId`: UUID 기본 키
- `userId`, `installationId`
- `expoPushToken`, `platform`
- `disabledAt`, `disabledReason`, `lastSeenAt`
- 생성·수정 시각

`installationId`는 새 식별자를 만들지 않고 현재 인증 세션이 이미 사용하는 안정적인 device ID를 재사용한다. `installationId`와 `expoPushToken`은 각각 고유하다. 로그인한 사용자가 바뀌면 같은 설치의 소유자를 원자적으로 갱신한다. 로그아웃과 탈퇴는 해당 설치의 토큰을 비활성화하고 아직 완료되지 않은 Outbox 행을 종료한다. `DeviceNotRegistered`로 비활성화된 같은 토큰은 재등록으로 되살리지 않으며, 해당 설치가 새 토큰을 받은 경우에만 다시 활성화한다.

### `notificationPreferences`

- `userId`: 기본 키
- `pushEnabled`
- `orderPushEnabled`
- `wishPushEnabled`
- `stylePushEnabled`
- `updatedAt`

기본값은 모두 활성화다. 이 설정은 Push 발송만 제어하며 앱 내 알림 기록은 항상 생성한다.

### `pushOutbox`

- `pushOutboxId`: UUID 기본 키
- `notificationId`, `pushDeviceId`
- `status`: `PENDING`, `PROCESSING`, `TICKETED`, `RECEIPT_OK`, `FAILED`
- `attemptCount`, `availableAt`
- `claimToken`, `claimedAt`
- `expoTicketId`, `receiptAvailableAt`
- `lastError`, 생성·수정 시각

`notificationId + pushDeviceId`를 고유하게 한다. 알림 생성 시 활성화된 기기와 카테고리 설정에 대해서만 Outbox 행을 만든다.

### 사용자 수명주기와 비밀번호

`users.password`는 nullable로 변경한다. 이메일 가입은 bcrypt 해시를 저장하고, 새 카카오 전용 가입은 `NULL`을 저장한다. USER가 아닌 BO·Partner 계정은 비밀번호가 반드시 존재하도록 DB 제약을 둔다.

기존 카카오 전용 계정의 임의 해시는 같은 트랜잭션에서 생성된 `users`와 `authIdentity`의 동일 생성 시각을 이용해 식별하고 `NULL`로 이관한다. 두 테이블의 `defaultNow()`가 같은 PostgreSQL 트랜잭션 시각을 기록하는 현재 생성 경로를 migration test로 고정한다. 기존 이메일 계정에 나중에 카카오를 연결한 경우 생성 시각이 다르므로 비밀번호를 유지한다.

사용자에는 `deactivatedAt`, `scheduledAnonymizationAt`, `anonymizedAt`을 추가한다.

## 알림 생성 규칙

### 주문

실제 상태가 바뀐 경우에만 주문 소유자에게 알린다.

- `PAID`: 결제 완료
- `FULFILLING`: 상품 준비 중
- `COMPLETED`: 주문 완료
- `FAILED`: 결제 또는 주문 처리 실패
- `CANCELLED`: 주문 취소

초기 `PAYMENT_PENDING` 생성은 알리지 않는다. 알림의 대상 경로는 주문 상세다.

현재 코드에는 결제 모듈, 서명된 결제 콜백, 또는 production `PAYMENT_PENDING -> PAID` writer가 없다. 관리자 전이 규칙도 이 변경을 의도적으로 금지한다. 따라서 이번 구현은 `PAID` copy와 dedupe 생성을 지원하되, 실제 `PAID` producer는 결제 공급자와 서명 검증 계약이 정해질 때까지 외부 의존성으로 남긴다. 결제 확인용 GraphQL/admin 우회 mutation은 만들지 않는다. 현재 실제 writer가 있는 `FULFILLING`, `COMPLETED`, `FAILED`, `CANCELLED`는 모두 같은 트랜잭션에서 알림을 생성한다.

### 위시 상품

게시된 상품의 활성 SKU를 기준으로 변경 전후 값을 비교한다.

- 최저 판매가가 내려가면 가격 인하 알림
- 총 재고가 0에서 양수로 바뀌면 재입고 알림

해당 상품을 위시한 사용자마다 앱 내 알림을 생성한다. 같은 상품 수정에서 가격 인하와 재입고가 동시에 발생하면 두 알림을 각각 생성한다. 대상 경로는 상품 상세다.

현재 `updatePartnerProductDraft`는 게시 상품을 수정할 수 없으므로 `updatePublishedProductSkus(productId, skus)` mutation을 추가한다. 서비스는 소유한 `PUBLISHED/APPROVED` 상품을 잠그고 기존 활성 SKU의 최저가와 총 재고를 읽은 뒤 SKU 가격·재고만 갱신한다. SKU 코드, 옵션, 상품명, 설명, 이미지, 카테고리는 이 mutation으로 바꿀 수 없다. 변경 전후를 비교해 알림과 Outbox를 같은 트랜잭션에 저장한다.

Partner 편집 화면은 게시 상품일 때 상품 정보 필드를 읽기 전용으로 만들고 SKU 가격·재고만 활성화한다. 저장은 `updatePublishedProductSkus`를 호출하며 기존 draft 저장이나 승인 절차를 우회하지 않는다.

### 스타일 좋아요

좋아요 행이 새로 활성화된 경우 게시물 작성자에게 알린다. 작성자가 자기 게시물에 누른 좋아요는 제외한다. 같은 사용자와 게시물 조합은 좋아요를 취소하고 다시 눌러도 한 번만 알린다. 대상 경로는 스타일 상세다.

## Expo Push 흐름

1. 인증된 첫 로그인 후 FO가 알림 권한을 요청한다.
2. 허용되면 `Constants`의 EAS project ID로 Expo Push Token을 발급받아 설치 ID와 함께 등록한다.
3. 도메인 트랜잭션이 앱 내 알림과 Push Outbox를 함께 저장한다.
4. 발송 워커가 대기 행을 claim하고 Expo Push API에 최대 100개씩 보낸다.
5. 네트워크 오류, HTTP 429, 5xx는 지수 백오프로 재시도한다.
6. ticket을 저장하고 약 15분 뒤 receipt를 조회한다.
7. `DeviceNotRegistered` receipt는 기기 토큰을 영구 비활성화한다.
8. 재시도 한도를 넘긴 발송은 `FAILED`로 끝내고 앱 내 알림은 보존한다.

Push payload는 민감정보를 넣지 않고 알림 ID, 타입, 목적지 ID만 포함한다. 알림을 눌렀을 때 FO가 서버의 알림을 조회하고 허용된 내부 경로로 이동한다. 삭제됐거나 접근할 수 없는 대상은 알림함으로 돌아간다.

`expo-notifications`와 설정 플러그인은 FO에 추가한다. 이미 설치된 `expo-constants`는 project ID와 앱 버전에 재사용한다. EAS project ID와 APNs credential은 실제 원격 Push 검증의 필수 조건이다.

## FO 화면과 API

### 알림함

- 최신순 cursor 페이지네이션
- 읽지 않은 행 표시
- 개별 읽음 처리
- 전체 읽음 처리
- 알림 선택 시 읽음 처리 후 대상 화면 이동
- 빈 상태, 재시도 가능한 오류 상태

GraphQL 계약은 `foNotifications`, `markFoNotificationRead`, `markAllFoNotificationsRead`를 제공한다. 목록 응답에는 현재 `unreadCount`를 포함한다.

### 쇼핑 메뉴

쇼핑의 첫 액션은 카테고리 전체 메뉴 시트를 연다. 기존 카테고리 쿼리와 스타일을 재사용한다.

- 카테고리 선택: 쇼핑의 `categoryId` 필터를 적용하고 시트 닫기
- 비교함 선택: 기존 비교함 경로로 이동

이 시트는 전역 앱 메뉴가 아니다.

### 설정

- 전체 Push 토글
- 주문, 위시, 스타일 Push 토글
- OS 알림 권한 상태와 시스템 설정 이동
- 계정 이메일
- 비밀번호 변경 또는 재설정
- 로그아웃
- 회원 탈퇴
- 앱 버전

GraphQL 계약은 `foNotificationPreferences`, `updateFoNotificationPreferences`, `registerFoPushDevice`, `unregisterFoPushDevice`를 제공한다. 비로그인 사용자는 앱 버전을 볼 수 있고 계정·Push 항목 선택 시 로그인으로 이동한다.

`me`에는 본인에게만 보이는 `hasPassword`를 추가한다. 카카오 전용 계정은 설정에서 비밀번호 변경 항목을 표시하지 않는다.

## 비밀번호 찾기

로그인 화면의 `비밀번호 찾기`를 누르면 React Native 네이티브 Alert를 먼저 표시한다.

- 본문: `이메일로 가입한 계정만 비밀번호를 재설정할 수 있어요. 카카오로 가입했다면 카카오 로그인을 이용해 주세요.`
- 버튼: `취소`, `이메일 계정 계속`
- `이메일 계정 계속`을 선택한 경우에만 기존 비밀번호 재설정 화면으로 이동

비밀번호가 `NULL`인 계정의 이메일 로그인은 dummy bcrypt 비교를 수행한 뒤 기존의 일반 로그인 오류를 반환한다. 비밀번호 재설정 메일 요청도 계정 존재 여부와 `hasPassword`를 외부에 노출하지 않고 동일한 성공 응답을 반환하며, 비밀번호가 있는 계정에만 메일을 보낸다.

설정에서 비밀번호를 변경하는 이메일 계정은 기존 이메일 인증 코드 흐름을 재사용한다. 변경 완료 시 모든 refresh session을 폐기하고 로그인 화면으로 이동한다.

## 탈퇴와 복구

이 수명주기는 역할이 `USER`인 FO 계정에만 적용한다. 사업자 상품·정산 보존 정책이 필요한 `PARTNER` 계정 탈퇴는 이번 범위에 포함하지 않으며, FO 설정에서도 파트너에게 회원 탈퇴 항목을 노출하지 않는다.

진행 중인 주문이 `PAYMENT_PENDING`, `PAID`, `FULFILLING` 중 하나면 탈퇴 요청을 거부하고 주문 완료 또는 취소를 안내한다.

탈퇴 확인 시 다음 변경을 한 트랜잭션에서 수행한다.

- `deactivatedAt` 기록
- `scheduledAnonymizationAt`을 30일 뒤로 설정
- 모든 refresh session 폐기
- 모든 Push 기기 비활성화
- 현재 앱 세션 종료

유예 기간에는 일반 세션을 발급하지 않는다. 이메일 비밀번호 또는 카카오 인증이 성공하면 device-bound 단기 재활성화 토큰과 `REACTIVATION_REQUIRED` 상태를 반환한다. FO가 복구 확인창을 표시하고 사용자가 승인하면 계정을 복원하고 정상 세션을 발급한다.

30일 뒤 워커는 아직 비활성 상태인 계정을 claim해 다음 작업을 원자적으로 수행한다.

- 이메일과 userid를 충돌하지 않는 익명 값으로 교체
- 비밀번호, 카카오 identity, 본인인증 식별자, 세션, 복구 토큰 제거
- Push 기기, 알림 설정, 알림, Outbox 제거
- 위시, 비교함, 장바구니, 최근 본 상품, 팔로우와 좋아요 제거
- 사용자 행과 주문·주문 상품은 익명 사용자 ID로 보존
- 스타일 게시물은 유지하되 작성자를 `탈퇴한 사용자`로 표시
- `anonymizedAt` 기록

익명화된 계정은 복구할 수 없다.

## 오류 처리

- Expo 장애는 도메인 트랜잭션을 실패시키지 않는다.
- claim이 만료된 Outbox 행은 기존 이메일 Outbox 방식으로 다시 대기시킨다.
- Push 권한 거부 시 앱 내 알림함은 계속 동작한다.
- 알림 권한이 영구 거부된 경우 설정에서 `Linking.openSettings()`를 제공한다.
- 기기 토큰 등록 실패는 로그인 자체를 실패시키지 않고 다음 foreground에서 재시도한다.
- 중복 도메인 요청은 `dedupeKey`와 Outbox 고유 키로 흡수한다.
- 존재하지 않거나 권한 없는 딥링크 대상은 알림함으로 안전하게 복귀한다.
- 비밀번호, 복구, 탈퇴 오류는 계정 존재 여부나 인증 공급자를 익명 사용자에게 노출하지 않는다.

## 검증

### Backend

- nullable 비밀번호와 사용자 수명주기 migration smoke test
- 카카오 가입이 `NULL` 비밀번호를 저장하고 이메일 가입이 bcrypt 해시를 저장하는 테스트
- 비밀번호 없는 로그인과 재설정 요청의 계정 열거 방지 테스트
- 현재 production writer가 있는 주문 상태별 알림 생성과 중복 방지 통합 테스트, `PAID` copy/dedupe 규칙 테스트
- 위시 가격 인하·재입고 경계 테스트
- 스타일 자기 좋아요 제외와 중복 방지 테스트
- Outbox claim, 재시도, ticket, receipt, `DeviceNotRegistered` 테스트
- 푸시 설정이 Outbox만 억제하고 앱 내 알림은 남기는 테스트
- 30일 탈퇴, 복구, 진행 중 주문 차단, 익명화 테스트

### FO

- 탭별 액션 아이콘, 형태, 경로 통합 테스트
- `ProductLayout`이 홈, 스타일, 쇼핑에만 적용되는 회귀 테스트
- 비밀번호 찾기 Alert의 취소와 계속 테스트
- 알림 목록, 개별·전체 읽음, 딥링크 fallback 테스트
- 쇼핑 메뉴 카테고리 적용과 비교함 이동 테스트
- Push 권한, 토큰 등록, 설정 토글 테스트
- 카카오 전용 계정에서 비밀번호 설정 항목이 숨겨지는 테스트
- 탈퇴와 재활성화 화면 테스트

### Partner

- 게시 상품에서 상품 정보·SKU 식별 필드가 읽기 전용인지 확인
- 게시 상품 가격·재고 저장이 `updatePublishedProductSkus`만 호출하는지 확인
- draft 상품이 계속 `updatePartnerProductDraft`를 사용하는 회귀 테스트

### 실행 검증

- Backend typecheck, lint, unit, integration, migration smoke
- FO typecheck, lint, unit, integration, native action tests
- Partner typecheck, lint, FSD, unit, build
- 실행 중인 Metro를 재사용해 iPhone 17 Pro 시뮬레이터에서 권한, 실제 `PAID -> FULFILLING` Expo Push 수신, 알림 선택 딥링크 확인
- Codex 내장 브라우저의 시뮬레이터 미러에서 탭별 헤더 액션과 목적지 확인

실제 Expo Push 완료 기준은 Expo ticket 성공만이 아니라 receipt 성공까지다. credential이 없으면 가짜 성공으로 대체하지 않고 외부 구성 누락으로 명시한다.

## 운영 기준

- Expo Push 요청은 공식 제한인 요청당 최대 100개와 project당 초당 600개를 넘지 않는다.
- receipt는 ticket 발급 후 조회하고 `DeviceNotRegistered`를 즉시 반영한다.
- 알림 본문은 주문번호 전체, 이메일, 주소 같은 민감정보를 포함하지 않는다.
- Outbox terminal 행과 앱 내 알림의 보존 기간은 기존 운영 정리 작업에 맞추되 계정 익명화 시 즉시 제거한다.

참고:

- [Expo Push 전송](https://docs.expo.dev/push-notifications/sending-notifications/)
- [Expo Push 설정](https://docs.expo.dev/push-notifications/push-notifications-setup/)
- [Expo Push FAQ](https://docs.expo.dev/push-notifications/faq/)
