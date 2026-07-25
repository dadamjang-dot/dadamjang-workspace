# SearchInput Focus Animation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ProductHeader에 SearchInput focus 시 검색창이 우측 버튼 영역 축소와 함께 부드럽게 늘어나고, ActionButton들이 취소 버튼으로 morphing되는 애니메이션을 react-native-reanimated로 구현한다.

**Architecture:** Reanimated `useSharedValue` + `useAnimatedStyle`으로 버튼 영역 너비를 childrenWidth↔cancelWidth 사이에서 보간하고, SearchInput wrapper 너비를 containerWidth에 맞춰 실시간 계산한다. 단일 버튼(2 actions)은 width 축소, 두 개 버튼(single action 각각)은 첫 번째 버튼 translateX → 두 번째 버튼 width 확장 → 취소 버튼 fade in 순서로 phase가 나뉜다.

**Tech Stack:** react-native-reanimated 4.5.0, @expo/ui (SwiftUI / Jetpack Compose), react-native-unistyles

## Global Constraints

- react-native-reanimated: `4.5.0` (package.json에 이미 존재)
- babel plugin: `react-native-reanimated/plugin` 필요
- `@expo/ui` native 컴포넌트는 수정하지 않음. Animated.View wrapper로 감싸서 제어.
- iOS/Android 공유 component: `product-header.shared.tsx` 사용
- `as any`, `@ts-ignore`, `@ts-expect-error` 사용 금지

---

### Task 1: Babel config 추가

**Files:**
- Create: `dadamjang-fe/apps/dadamjang-fo/babel.config.js`

**Interfaces:**
- Consumes: 없음
- Produces: `babel.config.js` — reanimated plugin이 적용된 Expo babel config

- [ ] **Step 1: babel.config.js 생성**

```js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: ['react-native-reanimated/plugin'],
};
```

- [ ] **Step 2: 타입체크 통과 확인**

```bash
cd dadamjang-fe/apps/dadamjang-fo && npx tsc --noEmit 2>&1 | tail -20
```

Expected: 타입 에러 없음 (기존 타입체크 통과 기준)

---

### Task 2: SearchInput 스타일 수정

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/shared/components/search-input/search-input.tsx`

**Interfaces:**
- Consumes: SearchInputProps (기존 유지)
- Produces: Animated.View wrapper가 너비를 제어할 수 있도록 `flex: 1` → `width: '100%'`

- [ ] **Step 1: StyleSheet.input의 flex: 1 제거, width: '100%' 추가**

```tsx
const s = StyleSheet.create({
  input: {
    width: '100%',
    height: 40,
    backgroundColor: '#FFFFFF',
    borderRadius: 20,
    paddingHorizontal: 16,
  },
});
```

- [ ] **Step 2: 타입체크**

```bash
cd dadamjang-fe/apps/dadamjang-fo && npx tsc --noEmit 2>&1 | tail -20
```

Expected: 타입 에러 없음

---

### Task 3: ProductHeader iOS/Android → shared re-export

**Files:**
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/shared/components/product-header/product-header.ios.tsx`
- Modify: `dadamjang-fe/apps/dadamjang-fo/src/shared/components/product-header/product-header.android.tsx`

**Interfaces:**
- Consumes: ProductHeaderProps (기존 유지)
- Produces: 두 platform 파일이 `./product-header.shared`를 re-export

- [ ] **Step 1: product-header.ios.tsx를 shared re-export로 변경**

```tsx
export { default } from './product-header.shared';
```

- [ ] **Step 2: product-header.android.tsx도 동일하게 변경**

```tsx
export { default } from './product-header.shared';
```

---

### Task 4: ProductHeader shared base — measurement + shared values

**Files:**
- Create: `dadamjang-fe/apps/dadamjang-fo/src/shared/components/product-header/product-header.shared.tsx`

**Interfaces:**
- Consumes:
  - `ProductHeaderProps` (기존 타입과 동일: children, isSearching, onSearchFocus, onSearchCancel, searchValue, onSearchValueChange)
  - `children` — 단일 ActionButton (actions.length===2) 또는 2개 ActionButton (각 actions.length===1)
- Produces:
  - `firstBtnProgress: SharedValue<0|1>` — 첫 번째 버튼 translateX/opacity
  - `secondBtnProgress: SharedValue<0|1>` — 두 번째 버튼 width 확장
  - `cancelProgress: SharedValue<0|1>` — 취소 버튼 opacity

- [ ] **Step 1: Imports 및 컴포넌트 셸 작성**

```tsx
import { useCallback, useEffect, useRef, type ReactNode } from 'react';
import { View, type LayoutChangeEvent, type TextInput } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  useDerivedValue,
  interpolate,
  withSpring,
  withDelay,
  Extrapolation,
  type SharedValue,
} from 'react-native-reanimated';
import { StyleSheet } from 'react-native-unistyles';

import { ActionButton, SearchInput } from '@/shared/components';

export interface ProductHeaderProps {
  children?: ReactNode;
  isSearching?: boolean;
  onSearchFocus?: () => void;
  onSearchCancel?: () => void;
  searchValue?: string;
  onSearchValueChange?: (text: string) => void;
}

const ProductHeader = ({
  children,
  isSearching = false,
  onSearchFocus,
  onSearchCancel,
  searchValue,
  onSearchValueChange,
}: ProductHeaderProps) => {
  const inputRef = useRef<TextInput>(null);

  // Layout measurements
  const containerWidth = useSharedValue(0);
  const childrenWidth = useSharedValue(0);
  const cancelWidth = useSharedValue(0);

  // Animation progress shared values
  const firstBtnProgress = useSharedValue(0);
  const secondBtnProgress = useSharedValue(0);
  const cancelProgress = useSharedValue(0);

  const handleCancel = useCallback(() => {
    inputRef.current?.blur();
    onSearchCancel?.();
  }, [onSearchCancel]);

  // ... continued
};
```

- [ ] **Step 2: Children type detection**

```tsx
// Children type detection helpers
const isSingleActionButton = (child: ReactNode): boolean => {
  if (!child || typeof child !== 'object' || !('type' in child)) return false;
  return child.type === ActionButton && (child.props as any)?.actions?.length === 2;
};

const isTwoActionButtons = (children: ReactNode): boolean => {
  const arr = Array.isArray(children) ? children : [children];
  if (arr.length !== 2) return false;
  return arr.every(
    (child) =>
      child && typeof child === 'object' && 'type' in child && child.type === ActionButton
  );
};

// Inside component:
const childArray = Array.isArray(children) ? children : [children];
const isTwoBtnCase = isTwoActionButtons(children);
```

- [ ] **Step 3: onLayout handlers**

```tsx
const handleContainerLayout = useCallback(
  (e: LayoutChangeEvent) => {
    containerWidth.value = e.nativeEvent.layout.width;
  },
  [containerWidth]
);

const handleChildrenLayout = useCallback(
  (e: LayoutChangeEvent) => {
    childrenWidth.value = e.nativeEvent.layout.width;
  },
  [childrenWidth]
);

const handleCancelLayout = useCallback(
  (e: LayoutChangeEvent) => {
    cancelWidth.value = e.nativeEvent.layout.width;
  },
  [cancelWidth]
);
```

- [ ] **Step 4: Derived animated button width**

```tsx
const btnWrapperWidth = useDerivedValue(() => {
  // Use the firstBtnProgress timeline to interpolate wrapper width
  return interpolate(
    firstBtnProgress.value,
    [0, 1],
    [childrenWidth.value, cancelWidth.value],
    Extrapolation.CLAMP
  );
});
```

- [ ] **Step 5: isSearching effect — animation dispatch**

```tsx
useEffect(() => {
  if (isSearching) {
    if (isTwoBtnCase) {
      // Case B: two buttons
      firstBtnProgress.value = withSpring(1, { damping: 20, stiffness: 180 });
      secondBtnProgress.value = withDelay(250, withSpring(1, { damping: 20, stiffness: 180 }));
      cancelProgress.value = withDelay(250, withSpring(1, { damping: 20, stiffness: 180 }));
    } else {
      // Case A: single button 2 actions
      firstBtnProgress.value = withSpring(1, { damping: 20, stiffness: 180 });
      cancelProgress.value = withDelay(180, withSpring(1, { damping: 20, stiffness: 180 }));
    }
  } else {
    // Reverse: cancel fades first
    cancelProgress.value = withSpring(0, { damping: 20, stiffness: 180 });
    if (isTwoBtnCase) {
      secondBtnProgress.value = withDelay(100, withSpring(0, { damping: 20, stiffness: 180 }));
      firstBtnProgress.value = withDelay(200, withSpring(0, { damping: 20, stiffness: 180 }));
    } else {
      firstBtnProgress.value = withDelay(100, withSpring(0, { damping: 20, stiffness: 180 }));
    }
  }
}, [isSearching]);
```

- [ ] **Step 6: Animated styles for SearchInput wrapper and btnWrapper**

```tsx
const searchInputStyle = useAnimatedStyle(() => ({
  width: containerWidth.value - 32 - 16 - btnWrapperWidth.value,
}));

const btnWrapperStyle = useAnimatedStyle(() => ({
  width: btnWrapperWidth.value,
}));
```

- [ ] **Step 7: Animated styles for children layer — Case A (single button)**

```tsx
const singleButtonLayerStyle = useAnimatedStyle(() => ({
  width: interpolate(firstBtnProgress.value, [0, 1], [childrenWidth.value, cancelWidth.value], Extrapolation.CLAMP),
  opacity: interpolate(firstBtnProgress.value, [0, 0.7], [1, 0], Extrapolation.CLAMP),
  // Right-aligned in btnWrapper → right edge fixed, left edge shrinks
}));
```

- [ ] **Step 8: Animated styles for children layer — Case B (two buttons)**

```tsx
const firstBtnStyle = useAnimatedStyle(() => ({
  transform: [
    {
      translateX: interpolate(
        firstBtnProgress.value,
        [0, 1],
        [0, childrenWidth.value - cancelWidth.value],
        Extrapolation.CLAMP
      ),
    },
  ],
  opacity: interpolate(firstBtnProgress.value, [0, 0.7], [1, 0], Extrapolation.CLAMP),
}));

const secondBtnStyle = useAnimatedStyle(() => ({
  width: interpolate(secondBtnProgress.value, [0, 1], [40, cancelWidth.value], Extrapolation.CLAMP),
}));
```

- [ ] **Step 9: Cancel layer animated style**

```tsx
const cancelStyle = useAnimatedStyle(() => ({
  opacity: cancelProgress.value,
}));
```

- [ ] **Step 10: Render 함수 작성**

```tsx
return (
  <View style={s.container} onLayout={handleContainerLayout}>
    <Animated.View style={[s.searchInputWrapper, searchInputStyle]}>
      <SearchInput
        ref={inputRef}
        value={searchValue}
        placeholder="Search"
        onValueChange={onSearchValueChange}
        onFocus={onSearchFocus}
      />
    </Animated.View>

    <Animated.View style={[s.btnWrapper, btnWrapperStyle]}>
      {/* Measurement layer — children */}
      <View style={s.measureLayer} onLayout={handleChildrenLayout}>
        {children}
      </View>

      {/* Measurement layer — cancel */}
      <View style={s.measureLayer} onLayout={handleCancelLayout}>
        <ActionButton actions={[{ label: '취소', onPress: handleCancel }]} />
      </View>

      {/* Animated children layer */}
      <Animated.View style={[s.childrenLayer, isTwoBtnCase ? undefined : singleButtonLayerStyle]}>
        {isTwoBtnCase ? (
          <>
            <Animated.View style={firstBtnStyle}>
              {childArray[0]}
            </Animated.View>
            <Animated.View style={secondBtnStyle}>
              {childArray[1]}
            </Animated.View>
          </>
        ) : (
          <Animated.View style={singleButtonLayerStyle}>
            {children}
          </Animated.View>
        )}
      </Animated.View>

      {/* Animated cancel layer */}
      <Animated.View style={[s.cancelLayer, cancelStyle]}>
        <ActionButton actions={[{ label: '취소', onPress: handleCancel }]} />
      </Animated.View>
    </Animated.View>
  </View>
);
```

- [ ] **Step 11: StyleSheet + export**

```tsx
const s = StyleSheet.create({
  container: {
    flexDirection: 'row',
    gap: 16,
    paddingHorizontal: 16,
    paddingVertical: 8,
  },
  searchInputWrapper: {
    height: 40,
  },
  btnWrapper: {
    height: 40,
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'flex-end',
  },
  measureLayer: {
    position: 'absolute',
    opacity: 0,
    pointerEvents: 'none',
    flexDirection: 'row',
    gap: 6,
    alignItems: 'center',
  },
  childrenLayer: {
    position: 'absolute',
    right: 0,
    top: 0,
    bottom: 0,
    flexDirection: 'row',
    gap: 6,
    alignItems: 'center',
    justifyContent: 'flex-end',
  },
  cancelLayer: {
    position: 'absolute',
    right: 0,
    top: 0,
    bottom: 0,
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'flex-end',
  },
});

export default ProductHeader;
```

- [ ] **Step 12: 타입체크**

```bash
cd dadamjang-fe/apps/dadamjang-fo && npx tsc --noEmit 2>&1 | tail -30
```

Expected: 모든 타입 에러 해결

---

### Task 5: 통합 검증

**Files:**
- 변경 전 파일들 전체 대상

- [ ] **Step 1: 전체 타입체크**

```bash
cd dadamjang-fe/apps/dadamjang-fo && npx tsc --noEmit 2>&1
```

Expected: exit 0

- [ ] **Step 2: UI 동작 확인 (iOS 시뮬레이터)**

```bash
cd dadamjang-fe/apps/dadamjang-fo && npx expo run:ios 2>&1 | tail -5
```

- Home Screen: SearchInput 탭 → 단일 버튼 축소 + 취소 등장
- Style Screen: SearchInput 탭 → 첫 버튼 translateX + 두 번째 버튼 확장 + 취소 등장
- 취소 버튼 탭 → 역애니메이션

- [ ] **Step 3: 최종 커밋**

```bash
git add dadamjang-fe/apps/dadamjang-fo/babel.config.js
git add dadamjang-fe/apps/dadamjang-fo/src/shared/components/search-input/search-input.tsx
git add dadamjang-fe/apps/dadamjang-fo/src/shared/components/product-header/
git add docs/superpowers
git commit -m "feat: add SearchInput focus morphing animation"
```

(실행 전 수정된 파일 목록 확인)
