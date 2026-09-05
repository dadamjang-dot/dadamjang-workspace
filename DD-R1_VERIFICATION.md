# DD-R1 주문 결과 조회·안전한 재시도 검증

검증일: 2026-09-06

## 기준과 환경

- 기준 SHA: workspace `450f54f9e766a6cc9e6f1ac8438149d258f9a721`, BE `d8e581bebc69d3ec9df757a921abfd789e1f1ec9`, FE `9003ceed30b2c9c874d45508984637ff040a845a`
- 검증 브랜치 SHA: BE `4d348febe8a4a2f3926e3030c4d4fd9a7166592b`, FE `e4bf3bcfaa9a7df3030abc214bc6d66e4153aa66`
- develop squash SHA: BE `14be7595f21f2183e8427b714182528fd32c07d3`, FE `bc6e96c65d103f30e009fe4074e370b71665b8b5`
- 환경: macOS Darwin 25.4.0 arm64, Node.js 22.13.0, pnpm 11.20.0, Docker 29.6.1, PostgreSQL 격리 테스트 컨테이너 `dadamjang_test:55432`
- 데이터: `src/database/fixtures.ts`의 합성 사용자·상품과 매 시나리오 초기화된 테스트 DB만 사용
- 상대 순서는 `t0`, `t1`, `t2`로 기록했다. 동시성 시나리오는 wall-clock sleep 대신 deferred promise와 PostgreSQL lock latch를 사용했다.

## D1–D12 결과

| ID | 사건 상대 순서 | 기대값 | 실측값 | 근거 |
| --- | --- | --- | --- | --- |
| D1 | t0 첫 제출 보류 → t1 버튼 4회 추가 입력 → t2 완료 | 같은 키·snapshot, 중복 입력 차단, 주문 1건 | FE API 호출 1회와 같은 키·snapshot을 확인했다. 실제 DB 동시 요청도 같은 orderId, order/key 각 1건이었다. 2초 sleep은 deferred latch로 대체했다. | `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/checkout-attempt-state.test.tsx`, `dadamjang-be/test/order-concurrency.integration-spec.ts` |
| D2 | t0 최초 mutation 미도달 → t1 조회 | `NOT_OBSERVED`, 주문·cart·키·activity 불변 | 조회 3종(없는 키, PROCESSING, 타 계정 키)이 모두 `NOT_OBSERVED`였고 6개 checkout 관련 테이블 snapshot이 같았다. FE는 uncertain을 유지했고 mutation을 호출하지 않았다. | `dadamjang-be/test/graphql.integration-spec.ts`, `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/checkout-attempt-state.test.tsx`, `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/api-contracts.test.ts` |
| D3 | t0 주문 commit → t1 응답 유실 → t2 결과 조회 | 원 orderId 확인, 신규 주문 0건 | 조회 결과의 원 orderId를 confirmed로 보관하고 해당 주문 화면을 열었다. 확인 동작에서 checkout mutation 호출은 0회였다. Android 경로도 5/5 같았다. | `dadamjang-be/test/graphql.integration-spec.ts`, `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/checkout-attempt-state.test.tsx`, `dadamjang-fe/apps/dadamjang-fo/android-tests/checkout-recovery.test.tsx` |
| D4 | t0 checkout을 activity insert에서 lock → t1 조회 → t2 commit → t3 재조회 | 첫 조회 미확정, 다음 조회는 같은 주문 | 실제 PostgreSQL에서 5/5 `NOT_OBSERVED` 후 같은 orderId의 `CONFIRMED`였다. order/key/activity 각 1건, cart item 0건이었다. | `dadamjang-be/test/order-concurrency.integration-spec.ts` |
| D5 | t0 미관측 → t1 사용자가 재시도 선택 → t2 같은 요청 전송 | 최초 키·snapshot으로 주문 1건 | retry는 최초 키와 네 필드 snapshot을 그대로 보냈고 matching server snapshot으로 주문 1건을 만들었다. | `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/checkout-attempt-state.test.tsx`, `dadamjang-be/test/graphql.integration-spec.ts` |
| D6 | t0 최초 요청과 재시도 동시 대기 → t1 lock 해제 | 두 응답 orderId 동일, 주문 1건 | 같은 키·snapshot의 두 응답 orderId가 같았고 실제 DB order/key가 각 1건이었다. | `dadamjang-be/test/order-concurrency.integration-spec.ts` |
| D7 | t0 snapshot 고정 → t1 cartItemId·skuId·quantity·unitPrice 중 하나 변경 → t2 checkout | snapshot 오류, 변경 내용 주문 0건 | 네 필드 각각 `CART_SNAPSHOT_CHANGED`였고 checkout 전후 DB snapshot이 같았다. | `dadamjang-be/test/graphql.integration-spec.ts` |
| D8 | t0 주문 commit → t1 새 cart 생성 → t2 조회·같은 키 재시도 | 기존 orderId, 새 cart 보존 | replay가 기존 orderId를 반환했고 새 cart item이 남았으며 order는 1건이었다. | `dadamjang-be/test/graphql.integration-spec.ts` |
| D9 | t0 응답 유실 → t1 cart refetch 실패 → t2 unmount/remount → t3 조회 | 키·snapshot·uncertain UI 유지 | 재진입 후 복구 UI와 원 snapshot이 먼저 보였고 같은 키로 조회할 수 있었다. Android 경로도 5/5 같았다. | `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/checkout-attempt-state.test.tsx`, `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/commerce-screens.test.tsx`, `dadamjang-fe/apps/dadamjang-fo/android-tests/checkout-recovery.test.tsx` |
| D10 | t0 조회·mutation 시작 → t1 키·계정·generation 변경 또는 confirmed 저장 → t2 늦은 결과 | 새 세션 오염 0건, confirmed 후퇴 0건 | 늦은 key/user/generation 결과를 거부했고 clear 뒤 시도를 복원하지 않았으며 늦은 오류 뒤에도 confirmed/orderId가 유지됐다. Android late-error 경로도 5/5 같았다. | `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/checkout-attempt-state.test.tsx`, `dadamjang-fe/apps/dadamjang-fo/android-tests/checkout-recovery.test.tsx` |
| D11 | t0 무snapshot 또는 잘못된 입력·타 계정 키 → t1 API 호출 | 호환 범위·검증·소유권 정책 준수 | 무snapshot 기존 호출은 유지했고 snapshot 없는 로컬 시도는 재시도를 숨겼다. 빈/101개/UUID/수량/금액/대소문자 중복 입력을 거절했으며 타 계정 키는 공개하지 않았다. | `dadamjang-be/src/modules/order/order.service.spec.ts`, `dadamjang-be/test/graphql.integration-spec.ts`, `dadamjang-fe/apps/dadamjang-fo/__tests__/integration/checkout-attempt-state.test.tsx` |
| D12 | t0 업무 오류 수신 또는 인증 갱신 → t1 재전송 업무 오류 | `extensions.code` 보존, 인증 회귀 없음 | 최초 응답과 refresh 뒤 응답 모두 code를 보존했다. 기존 refresh 성공·실패·토큰 정리 계약도 통과했다. | `dadamjang-fe/apps/dadamjang-fo/__tests__/unit/graphql-client-refresh.test.ts` |

## 반복과 전체 회귀

- PostgreSQL 근거 묶음(`graphql.integration-spec.ts`, `order-concurrency.integration-spec.ts`)을 독립 프로세스로 5회 실행했다. 매회 2 suites, 38 tests가 통과했다.
- FE 상태·API·화면·오류 근거 묶음을 독립 프로세스로 5회 실행했다. 매회 5 suites, 87 tests가 통과했다.
- Android preset에서 D3·D9·D10을 각각 `it.each([1, 2, 3, 4, 5])`로 실행했다. 전체 Android 결과는 2 suites, 21 tests 통과였다.
- 최종 BE: lint·build 통과, unit 39 suites/315 tests, integration 20 suites/255 tests 통과.
- 최종 FE: typecheck 통과, unit 18 suites/130 tests, integration 36 suites/241 tests, Android 2 suites/21 tests 통과. lint는 오류 0건이며 기존 `require()` 경고 3건이 남아 있다.
- 전체 FE integration을 BE와 병렬 실행할 때 React `act` 진단이 출력됐지만 changed commerce suite 단독 실행은 26/26 통과하며 진단이 없었다.

## 보장 범위

- 주문 생성 확인은 `PAYMENT_PENDING`/`PENDING`이며 결제 승인으로 표시하지 않는다.
- 시도는 같은 앱 프로세스·로그인 세션 안에서만 유지한다. 로그아웃, 앱 종료, 재설치, 다중 기기 복원은 보장하지 않는다.
- snapshot 없는 기존 API 호출자는 하위 호환 대상이며 새 snapshot 일치 보호를 받는다고 주장하지 않는다.
- 미확정 중 cart가 바뀌어도 새 주문을 자동 시작하지 않는다.
- 실제 결제 provider, 운영 DB, 실제 사용자는 사용하지 않았다.
- Android 검증은 `jest-expo/android` 화면 경로이며 UI 응답 시간 측정이나 실기기 검증이 아니다. 시뮬레이터는 실행하지 않았다.
