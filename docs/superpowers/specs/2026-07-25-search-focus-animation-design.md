# SearchInput Focus Animation Design

**Date:** 2026-07-25
**Status:** Draft

## Summary

ProductHeader에 `react-native-reanimated`를 적용해 SearchInput focus 시 검색창이
부드럽게 늘어나고, 우측 ActionButton들이 단일 취소 버튼으로 morphing되는 애니메이션을
추가한다. isSearching이 false → true가 될 때 진행, 취소 시 역재생된다.

## Files

| File | Action | Description |
|------|--------|-------------|
| `product-header.shared.tsx` | 신규 | 애니메이션 로직 포함한 shared component |
| `product-header.ios.tsx` | 수정 | shared re-export만 하도록 변경 |
| `product-header.android.tsx` | 수정 | shared re-export만 하도록 변경 |
| `search-input.tsx` | 수정 | `flex: 1` → `width: '100%'` |
| `babel.config.js` | 신규 | `react-native-reanimated/plugin` 추가 |

## Layout Architecture

```
ProductHeader container (row, gap: 16, paddingHorizontal: 16, paddingVertical: 8)
├── Animated.View (searchInputWrapper)
│   └── SearchInput (width: 100%, height: 40)
└── Animated.View (btnWrapper, height: 40, justifyContent: flex-end)
    ├── children layer (absolute, right: 0, overflow: hidden)
    │   └── children (Animated wrappers)
    └── cancel layer (absolute, right: 0, opacity: animated)
        └── ActionButton (취소)
```

## Measurement Strategy

세 가지 너비를 `onLayout`으로 동적 측정한다.

| Shared Value | Source | Purpose |
|--------------|--------|---------|
| `containerWidth` | 최상위 View onLayout | SearchInput 너비 계산 기준 |
| `childrenWidth` | children 측정 View (opacity:0, absolute) | 애니메이션 시작점 버튼 영역 너비 |
| `cancelWidth` | 취소 버튼 측정 View (opacity:0, absolute) | 애니메이션 종료점 버튼 영역 너비 |

SearchInput wrapper 너비 = `containerWidth - 32(padding) - 16(gap) - animatedBtnWidth`

## Animation Flows

### 공유 Shared Values

```ts
firstBtnProgress  = useSharedValue(0)  // 첫 번째 버튼 translateX + opacity
secondBtnProgress = useSharedValue(0)  // 두 번째 버튼 width 확장
cancelProgress    = useSharedValue(0)  // 취소 버튼 layer opacity
```

### Case A: 단일 ActionButton (actions.length === 2) — Home, Shop

**isSearching: false → true**

| Phase | Timing | Action | Animated Style |
|-------|--------|--------|----------------|
| 1 | delay 0ms | 버튼 width 축소 + fade out | width: `childrenW→cancelW`, opacity: 1→0 |
| 2 | delay 180ms | 취소 버튼 fade in | cancelProgress: 0→1 (opacity) |

```
progress 0                        1
btnW     ──────────────────────→ 줄어듦
opacity  ■■■■■■■■■■■■■■■■■■──→ 사라짐
cancel   ────────────────────■■■■■■■■■■ 나타남
         0ms                  180ms
```

**isSearching: true → false** (역순)

캔슬 먼저 fade out, 동시에 버튼 width/opacity 복귀.

### Case B: 2개 ActionButton (각 single action, iconOnly) — Style

**isSearching: false → true**

| Phase | Timing | Action | Animated Style |
|-------|--------|--------|----------------|
| 1 | delay 0ms | firstBtn translateX + fade out | translateX: `0→(childrenW - cancelW)`, opacity: 1→0 |
| 2 | delay 250ms | secondBtn width 확장 | width: `40→cancelW` |
| 3 | delay 250ms | 취소 버튼 fade in | cancelProgress: 0→1 (opacity) |

```
        │← Phase 1 →│←── Phase 2 & 3 ──→│
       0ms          250ms                500ms
firstBtn ■■■■■■■■■■■■■■■■■■──→ 이동+fade
secondBtn──────────────────■■■■■■ width 40→cancelW
cancel  ────────────────────■■■■■■■■■■ 나타남
```

**isSearching: true → false** (역순)

cancel 먼저 fade out → secondBtn width cancelW→40 복귀 → firstBtn translateX + fade 복귀.

### Layout 변화 (SearchInput 너비)

btnWrapper의 현재 `animatedBtnWidth`에 따라 SearchInput wrapper 너비가 실시간으로
변한다. flex: 1 대신 직접 계산하므로 취소 버튼은 우측 끝에 고정된다.

```
SearchInput width = containerWidth - 32 - 16 - animatedBtnWidth
```

`animatedBtnWidth`는 btnWrapper의 animated style과 동일한 값으로,
childrenWidth → cancelWidth 사이를 보간한다.

## Elements by File

### product-header.shared.tsx

| Section | Detail |
|---------|--------|
| Imports | reanimated (`Animated`, `useSharedValue`, `useAnimatedStyle`, `interpolate`, `withSpring`, `withDelay`, `Extrapolation`), react, react-native, components |
| Props | 기존 ProductHeaderProps + children |
| Shared values | `containerWidth`, `childrenWidth`, `cancelWidth`, `firstBtnProgress`, `secondBtnProgress`, `cancelProgress` |
| Derived value | `btnWrapperWidth` (가장 긴 progress track 기준) |
| isSearching effect | `useEffect`로 isSearching 변화 감지, 정방향/역방향 각각 withSpring + withDelay |
| SearchInput style | `useAnimatedStyle` → width: 계산식 |
| BtnWrapper style | `useAnimatedStyle` → width: btnWrapperWidth |
| Children layer | type detection → case A or B AnimatedStyle 적용 |
| Cancel layer | `useAnimatedStyle` → opacity: cancelProgress |
| Children type detection | `React.Children.toArray` + isValidElement + actions.length 판단 |

### search-input.tsx

| Change |
|--------|
| Style `flex: 1` → `width: '100%'` |

### babel.config.js

```js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: ['react-native-reanimated/plugin'],
};
```

## Edge Cases / Non-goals

- **Children가 ActionButton이 아닌 다른 타입**: 측정 후 width 보간은 되지만 morphing 효과 없이 일반 fade out
- **측정 전 isSearching이 먼저 true가 되는 case**: onLayout이 mount 즉시 실행되므로 race condition 없음. 단 shared value가 0인 상태에서 interpolate될 수 있어, 초기 childrenWidth가 0이면 btnWrapper width도 0으로 시작하지만 이후 onLayout에서 업데이트됨.
- **isSearching이 측정 완료 전에 빠르게 토글**: onLayout이 비동기로 실행되어도 shared value가 최종값으로 덮어써지므로 문제 없음.

## Assumptions

- 취소 버튼은 btnWrapper 우측 끝에 고정되어 등장한다.
- iconOnly ActionButton은 40px 너비를 가진다.
- iOS/Android ProductHeader가 동일한 shared base를 사용한다.
- @expo/ui native ActionButton을 Animated.View로 감싸서 width/opacity를 제어한다.
- 기존 ActionButton 내부 SwiftUI/Jetpack Compose 콘텐츠는 reflow되지 않을 수 있으므로, 필요 시 overflow: hidden 처리.

## Test Plan

1. **Home screen**: 단일 ActionButton 2 actions. SearchInput focus → 버튼 줄어들며 취소 등장. 취소 터치 → 복귀.
2. **Shop screen**: 동일.
3. **Style screen**: 두 ActionButton. focus → 첫 번째 버튼 translateX + fade, 두 번째 width 40→cancelW, 취소 등장. 취소 → 역순.
4. 초기 렌더링 시 버튼 깜빡임 없음.
5. `pnpm run typecheck`, `pnpm run lint` 통과.
6. iOS/Android 시뮬레이터 확인.
