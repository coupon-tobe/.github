# CUPI To-Be Coupon Platform

CUPI To-Be 쿠폰플랫폼은 쿠폰 생애주기와 데이터 소유권을 기준으로 분리된 Java/Spring Boot 기반 멀티 레포지토리 프로젝트입니다.
각 서비스는 독립적으로 개발, 빌드, 배포되며 공통 계약은 각 레포지토리의 `docs/` 문서와 OpenAPI 초안을 기준으로 맞춥니다.

## Repository 구성

| Repository | 역할 | Local port |
| --- | --- | --- |
| `coupon-catalog` | 쿠폰 속성, 템플릿, 생성, 승인 | 8081 |
| `coupon-trx` | 판매, 주문, 결제, 환불, 보유, 사용, 선물, 양도 | 8082 |
| `coupon-query` | 조회 전용 API, Read Replica 기반 조회 | 8083 |
| `coupon-omss` | TV 쿠폰, 충전, 기존 OMSS 통합 | 8084 |
| `coupon-issue` | 쿠폰 발행, 쿠폰번호 생성, 엑셀 생성 | 8085 |
| `coupon-if` | 문자발송, 시스템 발송, 외부 인터페이스 | 8086 |
| `coupon-auth` | RBAC, 사용자/역할/메뉴 권한, JWT/JWKS | 8087 |
| `coupon-common` | 고객, 상품, 도메인, 채널 등 기준정보 | 8088 |
| `coupon-batch` | 일/월 배치, 정산, 통계, 만료 처리 | 8089 |
| `cupi-gitops` | Helm Chart, ArgoCD, 환경별 values, 로컬 개발 인프라 | - |

## 기술 기준

- Java 21
- Spring Boot 3.x
- Gradle Groovy DSL
- PostgreSQL, Redis, Kafka
- Helm Chart + ArgoCD 기반 GitOps
- JWT/RBAC 기반 인증/인가

## 로컬 개발 세팅

### 1. 공통 인프라 실행

```bash
git clone https://github.com/coupon-tobe/cupi-gitops.git
cd cupi-gitops/local-dev
cp .env.example .env
docker compose up -d
```

기본 로컬 인프라 포트는 다음과 같습니다.

| Component | Port |
| --- | --- |
| PostgreSQL write | 5432 |
| PostgreSQL read | 5433 |
| Redis | 6379 |
| Kafka | 9092 |
| Kafka UI | 9000 |
| pgAdmin | 5050 |

### 2. 서비스 clone 및 빌드

```bash
git clone https://github.com/coupon-tobe/coupon-catalog.git
cd coupon-catalog
./gradlew clean build
```

다른 Spring Boot 서비스도 동일하게 빌드합니다.

```bash
./gradlew test
./gradlew clean build
```

### 3. 서비스 로컬 실행

각 서비스는 `local` profile로 실행합니다.

```bash
SPRING_PROFILES_ACTIVE=local ./gradlew bootRun
```

실행 후 다음 엔드포인트로 확인합니다.

```bash
curl http://localhost:8081/actuator/health
curl http://localhost:8081/api/v1/coupon-catalog/ping
```

서비스별 포트와 ping path는 각 레포지토리 `README.md`와 `docs/local-dev.md`를 기준으로 확인합니다.

## 개발 문서

각 Spring Boot 서비스는 다음 문서를 포함합니다.

- `README.md`: 서비스 목적, 담당 업무, 주요 API, 이벤트, 로컬 실행 방법
- `docs/developer-guide.md`: 담당 개발자가 구현해야 할 기능과 완료 기준
- `docs/architecture.md`: AA 설계상 위치, 책임, 데이터 소유권, 서비스 관계
- `docs/api.md`, `docs/api/openapi.yaml`: API 계약과 OpenAPI 초안
- `docs/events.md`: Kafka topic/event envelope/DLQ 정책
- `docs/redis.md`: Redis key, TTL, fallback 정책
- `docs/security.md`: JWT/RBAC 계약
- `docs/error-format.md`: RFC 9457 Problem Details 에러 응답 계약
- `docs/local-dev.md`: 로컬 실행 및 의존성 설정

## 개발 원칙

- 실제 비즈니스 로직은 각 서비스 책임 경계 안에서 구현합니다.
- 다른 서비스의 소유 데이터를 직접 수정하지 않습니다.
- 서비스 간 비동기 협력은 Kafka를 우선 사용합니다.
- 강한 동기 호출이 필요한 경우에는 호출 사유와 fallback 정책을 문서화합니다.
- Kotlin 소스코드와 Kotlin DSL은 사용하지 않습니다.
- 운영 Secret 값은 커밋하지 않습니다.
