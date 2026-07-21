# Frontend Rules

```txt
src/app/              # Expo Router
src/features/<name>/  # Domain feature code
src/shared/           # Shared code
src/providers/        # Provider code
src/theme/            # Theme code
src/assets/           # images, fonts
```

- Feature code stays in `src/features/<name>/`.
- API calls stay in `src/features/<name>/api.ts`.
- Hooks stay in `src/features/<name>/hooks.ts`.
- Types stay in `src/features/<name>/types.ts`.
- Shared UI stays in `src/shared/components/`.
- File name: `kebab-case`.
- Platform file: `file-name.ios.tsx`, `file-name.android.tsx`.
- Use `@/` imports only across sibling folders: `@/<sibling>/<path>`.
- Use arrow function expressions for all functions, including hooks and components: `const func = () => {}`.
- Re-export modules through `index.ts`.
- Use `react-native-unistyles` for styling only.
- 주석을 작성하지 않는다. 코드만으로 의도가 드러나도록 작성한다.

## Component Structure

컴포넌트 파일과 인덱스 파일은 다음 구조를 따른다:

```
// component-name.tsx
const ComponentName = () => {};

export default ComponentName;

// index.ts
export { default as ComponentName } from './component-name';
```
