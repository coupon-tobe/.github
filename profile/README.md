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

## 로컬 개발 환경

개발자 PC에는 Docker Desktop이 설치되어 있다고 가정합니다. 로컬 개발 환경의 기준 파일은 `cupi-gitops/local-dev/docker-compose.yml`입니다.

### 1. 준비

```bash
git clone https://github.com/coupon-tobe/cupi-gitops.git
cd cupi-gitops/local-dev
cp .env.example .env
```

`.env`는 커밋하지 않습니다. 로컬 DB 비밀번호, PaaS DB 비밀번호 같은 값은 개발자 PC의 `.env`에만 둡니다.

### 2. 인프라만 실행

```bash
docker compose up -d
```

기본 인프라:

| Component | Host port | Container/usage |
| --- | ---: | --- |
| PostgreSQL write | 5432 | `cupi-postgres-write` |
| PostgreSQL read scaffold | 5433 | `cupi-postgres-read` |
| Redis | 6379 | `cupi-redis` |
| Kafka | 9092 | host app: `localhost:9092`, container app: `kafka:29092` |
| Kafka UI | 9000 | http://localhost:9000 |
| pgAdmin | 5050 | http://localhost:5050 |

Kafka 기본 topic과 DLQ topic은 `kafka-init` 컨테이너가 생성합니다.

### 3. 전체 서비스 일괄 실행

```bash
docker compose --profile apps up -d --build
```

9개 Spring Boot 서비스가 모두 컨테이너로 실행됩니다.

| Service | Port | Health | Ping |
| --- | ---: | --- | --- |
| coupon-catalog | 8081 | `/actuator/health` | `/api/v1/coupon-catalog/ping` |
| coupon-trx | 8082 | `/actuator/health` | `/api/v1/coupon-trx/ping` |
| coupon-query | 8083 | `/actuator/health` | `/api/v1/coupon-query/ping` |
| coupon-omss | 8084 | `/actuator/health` | `/api/v1/coupon-omss/ping` |
| coupon-issue | 8085 | `/actuator/health` | `/api/v1/coupon-issue/ping` |
| coupon-if | 8086 | `/actuator/health` | `/api/v1/coupon-if/ping` |
| coupon-auth | 8087 | `/actuator/health` | `/api/v1/coupon-auth/ping` |
| coupon-common | 8088 | `/actuator/health` | `/api/v1/coupon-common/ping` |
| coupon-batch | 8089 | `/actuator/health` | `/api/v1/coupon-batch/ping` |

### 4. 개별 서비스 실행

필요한 인프라와 특정 서비스만 실행할 수 있습니다.

```bash
docker compose up -d --build coupon-auth
docker compose up -d --build coupon-catalog
docker compose up -d --build coupon-trx
```

이미 이미지가 빌드되어 있으면 `--build`를 생략할 수 있습니다.

호스트 JVM에서 개별 서비스를 실행할 수도 있습니다.

```bash
git clone https://github.com/coupon-tobe/coupon-catalog.git
cd coupon-catalog
./gradlew bootRun --args='--spring.profiles.active=local'
```

### 5. PaaS DB로 로컬 개발

로컬 PostgreSQL 대신 PaaS PostgreSQL에 붙어서 개발해야 하면 PaaS DB 템플릿을 사용합니다.

```bash
cd cupi-gitops/local-dev
cp .env.paas.example .env
vi .env   # APP_POSTGRES_WRITE_PASSWORD 값만 로컬에서 입력
docker compose --profile apps up -d --build
```

호스트 JVM에서 PaaS DB로 개별 실행:

```bash
cd cupi-gitops/local-dev
set -a
. ./.env
set +a

cd ../../coupon-catalog
./gradlew bootRun --args='--spring.profiles.active=local,paas-db'
```

`paas-db` 프로필은 DB 접속지만 PaaS로 전환합니다. Redis, Kafka, JWKS 등 다른 개발 의존성은 로컬 compose 환경을 사용합니다.

### 6. 구동 확인

컨테이너 상태:

```bash
docker compose ps
docker compose logs -f kafka
docker compose logs -f coupon-catalog
```

인프라 확인:

```bash
docker exec cupi-postgres-write pg_isready -U cupi -d cupi_write
docker exec cupi-redis redis-cli ping
docker exec cupi-kafka /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

서비스 확인:

```bash
curl http://localhost:8081/actuator/health
curl http://localhost:8081/api/v1/coupon-catalog/ping
```

전체 종료:

```bash
docker compose down
docker compose down -v   # DB 볼륨까지 삭제
```

자세한 내용은 `cupi-gitops/docs/local-dev.md`를 기준으로 확인합니다.

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
