# dadamjang workspace

다담장 커머스 플랫폼의 통합 작업 공간입니다.

기획과 구현 범위는 [docs/PLAN.md](./docs/PLAN.md)를 기준으로 관리합니다.

## 구성

- `dadamjang-fe`: 구매자 FO Expo 앱, 파트너/어드민 웹 앱을 위한 프론트엔드 저장소
- `dadamjang-be`: NestJS GraphQL API와 커머스 도메인 저장소
- `dadamjang-infra`: 로컬 개발 의존성, staging AWS 인프라, CI/CD 저장소

## 클론

```bash
git clone --recurse-submodules https://github.com/dadamjang-dot/dadamjang-workspace.git
cd dadamjang-workspace
```

이미 클론했다면 submodule을 초기화합니다.

```bash
git submodule update --init --recursive
```

submodule을 최신 원격 커밋으로 맞춥니다.

```bash
git submodule update --remote --merge
```

## 작업 순서

1. `dadamjang-infra`에서 로컬 DB/Redis/Mailpit을 실행합니다.
2. `dadamjang-be`에서 migration 후 API를 실행합니다.
3. `dadamjang-fe/apps/dadamjang-fo`에서 Expo 앱을 실행합니다.

## 원칙

- 각 submodule은 독립 repo입니다.
- 기능 변경은 해당 submodule에서 먼저 커밋/푸시합니다.
- workspace는 검증된 submodule SHA만 갱신합니다.
- GraphQL 계약은 `dadamjang-be`가 기준입니다.
- 비밀값은 `.env`, GitHub Secrets, AWS Secrets Manager에만 둡니다.
