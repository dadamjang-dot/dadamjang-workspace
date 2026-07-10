# dadamjang workspace

다담장 커머스 플랫폼의 통합 작업 공간입니다.

구현 범위와 인프라 결정은 [PLAN.md](./PLAN.md)를 확인합니다.

## 구성

- `dadamjang-fe`: 구매자 Expo 앱, 파트너·어드민 Next.js 웹
- `dadamjang-be`: GraphQL API와 커머스 도메인
- `dadamjang-infra`: 로컬 개발·배포 인프라

## 시작하기

```bash
git clone --recurse-submodules https://github.com/dadamjang-dot/dadamjang-workspace.git
```

이미 복제했다면 다음 명령으로 submodule을 초기화합니다.

```bash
git submodule update --init --recursive
```

## 원칙

- 각 submodule은 독립적으로 빌드·테스트·배포합니다.
- 통합 검증을 통과한 커밋만 workspace의 submodule SHA로 갱신합니다.
- GraphQL 계약은 `dadamjang-be`를 기준으로 관리합니다.
Dadamjang submodule workspace and integration runbook
