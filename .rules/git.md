# Git Rules

## Workflow

- 작업 카테고리(브랜치 타입)별로 브랜치를 생성한다.
- 각 작업 단위별로 커밋을 남긴다.
- 작업 완료 후 원격 저장소에 PR을 생성한다.
- PR은 `develop` 브랜치로 squash merge 한다.
- 병합된 작업 브랜치는 삭제한다.

## Branch

Use:

- `feat/<short-kebab-summary>`
- `fix/<short-kebab-summary>`
- `refactor/<short-kebab-summary>`
- `chore/<short-kebab-summary>`
- `docs/<short-kebab-summary>`
- `test/<short-kebab-summary>`

Rules:

- lowercase kebab-case
- no identity prefixes
- short, descriptive names
- base branch: develop

Examples:

- `feat/chat-room-search`
- `fix/login-token-refresh`

## Commit

Use Conventional Commits:

- `feat(scope): summary`
- `fix(scope): summary`
- `refactor(scope): summary`
- `chore(scope): summary`
- `docs(scope): summary`
- `test(scope): summary`

Rules:

- lowercase type
- scope required
- English summary
- summary under 72 chars

Examples:

- `feat(chat): add room search`
- `fix(auth): refresh expired token`

## Pull Request

- PR title follows the same format as commit messages.
- PR의 base 브랜치는 `develop`으로 한다.
- PR은 squash merge로 병합하며, 병합 후 작업 브랜치는 삭제한다.
- CI should reject invalid branch names and PR titles.
