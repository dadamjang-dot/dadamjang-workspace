# dadamjang plan

## 제품 범위

- FO: Expo 구매자 앱. 카카오·이메일 로그인, 개인화 피드, 상품 검색, 위시, 장바구니, mock 결제 주문.
- Partner: Next.js 판매자 웹. 사업자 이메일 인증과 사업자등록번호 제출 후 관리자 승인. 승인 전 상품 발행 불가.
- BO: Next.js 운영 웹. 공개 가입 없이 관리자 초대 수락 후 접근. 파트너 심사, 상품 승인, 주문·배송 상태를 관리.

## backend

- NestJS GraphQL modular monolith과 PostgreSQL(Drizzle)을 사용한다.
- `USER`, `PARTNER`, `ADMIN` 역할 기반 JWT 권한을 적용한다.
- 도메인: catalog, feed, wishlist, cart, order, partner, admin, event.
- 주문은 재고·가격을 transaction으로 확정하고 결제 승인·실패는 mock으로 처리한다.
- 행동 이벤트와 운영 감사 로그는 append-only로 기록한다.

## image

- 원본 상품 이미지는 Cloudflare R2에 저장한다.
- 썸네일·상세 이미지는 Cloudflare Images 변환 URL로 WebP/AVIF와 크기 variant를 제공한다.
- Cloudflare Images Free의 월 5,000 unique transformation 한도를 첫 staging 기준으로 사용한다.
- AWS S3와 CloudFront는 사용하지 않는다.

## infrastructure

- local: Docker Compose로 PostgreSQL, Redis, MinIO, Mailpit을 실행한다.
- staging: Terraform으로 AWS VPC, ALB, ECS Fargate API, RDS PostgreSQL, ElastiCache Redis, ECR, Secrets Manager를 구성한다.
- 이미지 저장·전송은 Cloudflare R2 + Images가 담당한다.
- GitHub Actions는 lint, test, build, image push, staging ECS deploy를 수행한다. Terraform apply는 승인된 workflow에서만 실행한다.
- production은 만들지 않는다. staging을 공개 포트폴리오 데모로 운영한다.

## 검증

- 권한, 파트너 승인 흐름, 재고 부족, mock 결제 실패, 주문 상태 전이, cursor pagination을 테스트한다.
- staging에서 이미지 업로드, 상품 발행, 찜·장바구니·주문, BO 초대·심사를 end-to-end로 검증한다.
