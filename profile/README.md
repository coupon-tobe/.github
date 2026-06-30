# CUPI To-Be Coupon Platform

CUPI To-Be 쿠폰플랫폼은 쿠폰 생애주기와 데이터 소유권을 기준으로 분리된 Java/Spring Boot 기반 멀티 레포지토리 프로젝트입니다.
각 서비스는 독립적으로 개발, 빌드, 배포되며 공통 계약은 각 레포지토리의 `docs/` 문서와 OpenAPI 초안을 기준으로 맞춥니다.

## 목차

1. [플랫폼 한눈에 보기](#1-플랫폼-한눈에-보기) — 무엇이고, 어떤 레포로 나뉘는지
2. [먼저 알아야 할 핵심 개념](#2-먼저-알아야-할-핵심-개념) — 반복해서 나오는 전제들
3. [로컬 개발 환경](#3-로컬-개발-환경) — 준비부터 종료까지 단계별 절차
4. [개발 문서](#4-개발-문서) — 각 서비스가 갖춰야 할 문서
5. [개발 원칙](#5-개발-원칙) — 모든 서비스가 지키는 규칙

---

## 1. 플랫폼 한눈에 보기

### Repository 구성

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

### 기술 기준

- Java 21
- Spring Boot 3.x
- Gradle Groovy DSL
- PostgreSQL, Redis, Kafka
- Helm Chart + ArgoCD 기반 GitOps
- JWT/RBAC 기반 인증/인가

---

## 2. 먼저 알아야 할 핵심 개념

아래 4가지 전제는 이후 모든 절차에서 반복적으로 등장합니다. 먼저 이해하면 단계별 설명이 훨씬 쉽게 읽힙니다.

**1) 데이터 소유권은 서비스가 가진다**
각 서비스는 자신의 데이터를 소유하며, 다른 서비스의 DB를 직접 수정하지 않습니다. 서비스 간 협력은 Kafka 비동기 이벤트를 우선 사용하고, 강한 동기 호출(REST)은 필요한 경우에만 사유·정책을 문서화하고 사용합니다.

**2) 기본 DB는 PaaS PostgreSQL이다 (로컬 DB는 선택)**
로컬에서 Spring Boot 서비스를 띄우면 별도 설정이 없는 한 PaaS PostgreSQL에 연결됩니다. 로컬 PostgreSQL 컨테이너(scaffold)는 격리된 DB가 필요할 때만 선택적으로 사용합니다(→ [3-H](#h-로컬-db-scaffold-사용-단계-10)). 당분간 write/read DB는 동일한 PaaS DB 접속 정보를 사용합니다.

**3) 서비스를 실행하는 두 가지 방식이 있고, 의존성 주소가 다르다**
서비스를 "호스트 JVM(`./gradlew bootRun`)"으로 띄우느냐 "컨테이너(docker compose)"로 띄우느냐에 따라 Redis/Kafka/JWKS 주소가 달라집니다. 문서에 같은 의존성이 두 가지 값으로 나오는 이유입니다.

| Dependency | 호스트 JVM 실행 | 컨테이너 실행 |
| --- | --- | --- |
| PostgreSQL | `.env`의 PaaS DB `POSTGRES_*` | `.env`의 PaaS DB `POSTGRES_*` |
| Redis | `localhost:6379` | `redis:6379` |
| Kafka | `localhost:9092` | `kafka:29092` |
| JWKS | `http://localhost:8087/.well-known/jwks.json` | `http://coupon-auth:8087/.well-known/jwks.json` |

**4) 인증은 JWT 발급 + JWKS 로컬 검증이다**
`coupon-auth`가 JWT를 발급하고 공개키를 JWKS로 제공합니다. 다른 서비스는 매 요청마다 auth를 호출하지 않고 JWKS 공개키를 캐싱해 토큰을 검증합니다. 따라서 `coupon-auth`가 떠 있어야 다른 서비스의 인증/인가가 동작합니다(→ [3-G](#g-인증jwks와-rbac-확인-단계-9)).

---

## 3. 로컬 개발 환경

개발자 PC에 Docker Desktop이 설치되어 있다고 가정합니다. 로컬 개발 환경의 기준 파일은 `cupi-gitops/local-dev/docker-compose.yml`입니다.

전체 흐름은 다음과 같습니다.

```
A. 준비(clone, .env)  →  B. 인프라 실행  →  C. 서비스 실행
        ↓
D. 통신 기준 / E. API 문서 / F. Kafka / G. 인증  (필요한 부분 확인)
        ↓
H. 로컬 DB scaffold(선택)  →  I. 종료/초기화
```

### A. 최초 준비 (단계 1)

작업 디렉토리 예시는 모든 레포가 같은 상위 폴더에 놓이는 구조입니다.

```bash
mkdir -p ~/workspace/cupon-tobe
cd ~/workspace/cupon-tobe

git clone https://github.com/coupon-tobe/cupi-gitops.git
git clone https://github.com/coupon-tobe/coupon-auth.git
git clone https://github.com/coupon-tobe/coupon-catalog.git
git clone https://github.com/coupon-tobe/coupon-trx.git
git clone https://github.com/coupon-tobe/coupon-query.git
git clone https://github.com/coupon-tobe/coupon-omss.git
git clone https://github.com/coupon-tobe/coupon-issue.git
git clone https://github.com/coupon-tobe/coupon-if.git
git clone https://github.com/coupon-tobe/coupon-common.git
git clone https://github.com/coupon-tobe/coupon-batch.git
```

로컬 compose 환경변수를 준비합니다.

```bash
cd ~/workspace/cupon-tobe/cupi-gitops/local-dev
cp .env.example .env
vi .env
```

`.env`는 커밋하지 않습니다. PaaS DB 비밀번호는 개발자 PC의 `.env`에만 둡니다.

필수 확인 값:

```dotenv
POSTGRES_WRITE_HOST=db-cupi-poc-postgresql-17.postgres.database.azure.com
POSTGRES_WRITE_PORT=5432
POSTGRES_WRITE_DB=postgres
POSTGRES_WRITE_USER=cupidbadmin
POSTGRES_WRITE_PASSWORD=<개발자 로컬에서만 입력>
POSTGRES_READ_HOST=db-cupi-poc-postgresql-17.postgres.database.azure.com
POSTGRES_READ_PORT=5432
POSTGRES_READ_DB=postgres
POSTGRES_READ_USER=cupidbadmin
POSTGRES_READ_PASSWORD=<개발자 로컬에서만 입력>
POSTGRES_SSL_MODE=require
```

당분간 write/read DB는 동일한 PaaS DB 접속 정보를 사용합니다.

### B. 인프라만 실행 (단계 2)

```bash
cd ~/workspace/cupon-tobe/cupi-gitops/local-dev
docker compose up -d
```

기본 인프라:

| Component | Host port | Container/network usage |
| --- | ---: | --- |
| PostgreSQL write scaffold | 5432 | `cupi-postgres-write`, 선택 사용 |
| PostgreSQL read scaffold | 5433 | `cupi-postgres-read`, 선택 사용 |
| Redis | 6379 | host app: `localhost:6379`, container app: `redis:6379` |
| Kafka | 9092 | host app: `localhost:9092`, container app: `kafka:29092` |
| Kafka UI | 9000 | http://localhost:9000 |
| API Docs Hub | 9010 | http://localhost:9010 |
| pgAdmin | 5050 | http://localhost:5050 |

Kafka 기본 topic과 DLQ topic은 `kafka-init` 컨테이너가 생성합니다. Spring Boot 서비스는 별도 설정을 하지 않으면 PaaS PostgreSQL을 사용합니다.

### C. 서비스 실행 (단계 3)

> 실행 방식에 따라 의존성 주소가 달라지는 점은 [2-3) 핵심 개념](#2-먼저-알아야-할-핵심-개념)을 참고하세요.

**전체 서비스를 한 번에 컨테이너로 실행:**

```bash
cd ~/workspace/cupon-tobe/cupi-gitops/local-dev
docker compose --profile apps up -d --build
```

**Swagger/OpenAPI 통합 Hub만 실행:**

```bash
docker compose --profile docs up -d api-docs-proxy
open http://localhost:9010
```

**개별 서비스를 컨테이너로 실행:**

```bash
docker compose up -d --build coupon-auth
docker compose up -d --build coupon-catalog
docker compose up -d --build coupon-trx
```

이미 이미지가 빌드되어 있으면 `--build`를 생략할 수 있습니다. 개별 서비스 컨테이너는 Redis, Kafka, Kafka topic 초기화 컨테이너를 함께 준비합니다. DB는 기본적으로 PaaS DB에 붙습니다.

**호스트 JVM에서 개별 서비스를 실행:**

```bash
cd ~/workspace/cupon-tobe/cupi-gitops/local-dev
set -a
. ./.env
set +a

cd ~/workspace/cupon-tobe/coupon-catalog
./gradlew bootRun --args='--spring.profiles.active=local'
```

#### 서비스 포트와 기본 확인 (단계 4)

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

전체 서비스 ping 확인:

```bash
for port_service in \
  "8081 coupon-catalog" \
  "8082 coupon-trx" \
  "8083 coupon-query" \
  "8084 coupon-omss" \
  "8085 coupon-issue" \
  "8086 coupon-if" \
  "8087 coupon-auth" \
  "8088 coupon-common" \
  "8089 coupon-batch"; do
  set -- $port_service
  curl -fsS "http://localhost:$1/api/v1/$2/ping"
  echo
done
```

### D. 서비스 간 통신 기준 (단계 5)

서비스 간 데이터 소유권은 각 서비스가 보유합니다. 다른 서비스 DB를 직접 수정하지 않습니다.

| 통신 | 방식 | 확인 포인트 |
| --- | --- | --- |
| 인증/인가 | `coupon-auth` JWT 발급 + JWKS 로컬 검증 | 각 서비스는 매 요청마다 auth를 호출하지 않고 JWKS 공개키를 캐싱 |
| 비동기 이벤트 | Kafka topic | coupon 생성/거래/발행/OMSS/IF/공통/배치 이벤트와 DLQ |
| 캐시/락 | Redis | cache-aside, 멱등성, 분산 락 |
| DB | PaaS PostgreSQL | 기본 개발 DB. write/read는 당분간 동일 접속 정보 |
| 동기 REST | 필요한 경우에만 | 호출 사유, timeout, retry, fallback 정책을 각 서비스 문서에 기록 |

현재 로컬 Kafka topic:

| Topic | 용도 |
| --- | --- |
| `coupon.creation.events` / `.dlq` | 쿠폰 생성/승인 이벤트 |
| `coupon.transaction.events` / `.dlq` | 거래/사용/환불 이벤트 |
| `coupon.issue.events` / `.dlq` | 발행 이벤트 |
| `coupon.omss.events` / `.dlq` | OMSS 연동 이벤트 |
| `coupon.notification.events` / `.dlq` | 알림/외부 발송 이벤트 |
| `coupon.interface.events` / `.dlq` | 외부 인터페이스 이벤트 |
| `coupon.common.events` / `.dlq` | 기준정보 변경 이벤트 |
| `coupon.batch.events` / `.dlq` | 배치 실행/완료 이벤트 |

### E. API 문서 통합 확인 (단계 6)

9개 서비스의 Swagger/OpenAPI 문서는 API Docs Hub에서 한 번에 확인합니다.
API Docs Hub는 로컬 개발환경에서 여러 Spring Boot 서비스의 `/v3/api-docs`를 한 화면에 모아 보는 Swagger UI입니다. 서비스별 Swagger UI 주소를 따로 외우지 않고, 좌측/상단 서비스 선택 목록에서 대상 서비스를 바꿔가며 API 스펙, 인증 스킴, request/response schema를 확인할 수 있습니다.

```bash
cd ~/workspace/cupon-tobe/cupi-gitops/local-dev
docker compose --profile apps up -d --build
open http://localhost:9010
```

Hub 구성:

| Component | 용도 |
| --- | --- |
| `cupi-api-docs-ui` | `swaggerapi/swagger-ui` 기반 통합 Swagger UI |
| `cupi-api-docs` | 각 서비스 `/v3/api-docs`와 API 호출 경로를 같은 origin으로 묶는 nginx proxy |

Hub proxy 경로:

| Path | 연결 대상 |
| --- | --- |
| `/api-docs/coupon-auth` | `coupon-auth:8087/v3/api-docs` |
| `/api-docs/coupon-catalog` | `coupon-catalog:8081/v3/api-docs` |
| `/api-docs/coupon-trx` | `coupon-trx:8082/v3/api-docs` |
| `/api-docs/coupon-query` | `coupon-query:8083/v3/api-docs` |
| `/api-docs/coupon-omss` | `coupon-omss:8084/v3/api-docs` |
| `/api-docs/coupon-issue` | `coupon-issue:8085/v3/api-docs` |
| `/api-docs/coupon-if` | `coupon-if:8086/v3/api-docs` |
| `/api-docs/coupon-common` | `coupon-common:8088/v3/api-docs` |
| `/api-docs/coupon-batch` | `coupon-batch:8089/v3/api-docs` |

API Docs Hub만 따로 띄우면 Swagger UI와 proxy만 실행됩니다. 이 경우 문서를 조회하려는 대상 서비스 컨테이너는 별도로 떠 있어야 합니다.

```bash
docker compose --profile docs up -d api-docs-proxy
docker compose up -d --build coupon-catalog
open http://localhost:9010
```

Swagger UI의 `Authorize` 버튼에는 `coupon-auth`에서 발급받은 JWT access token을 `Bearer` prefix 없이 입력합니다. API 호출까지 Swagger Hub에서 확인할 수 있지만, 호출 대상 서비스가 기동되어 있고 JWT/JWKS 설정이 정상이어야 합니다.

각 서비스의 개별 Swagger UI도 사용할 수 있습니다.

```bash
open http://localhost:8081/swagger-ui.html  # coupon-catalog
open http://localhost:8087/swagger-ui.html  # coupon-auth
```

서비스 컨테이너가 떠 있지 않으면 해당 서비스의 API 문서는 Hub에서 로딩되지 않습니다.

#### Controller에 Swagger/OpenAPI 적용 방법

신규 Controller를 만들거나 기존 API를 수정할 때는 HTTP API 계약이 Swagger Hub에 바로 드러나도록 springdoc annotation을 함께 작성합니다.

1. 서비스에 springdoc 의존성이 있는지 확인합니다.

```gradle
implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui"
```

2. 서비스 공통 `OpenApiConfig`에 API 제목, 버전, JWT 보안 스킴을 둡니다.

```java
@Configuration
public class OpenApiConfig {
    private static final String BEARER = "bearerAuth";

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("coupon-catalog API")
                        .version("0.0.1-SNAPSHOT")
                        .description("CUPI To-Be coupon-catalog service API"))
                .addSecurityItem(new SecurityRequirement().addList(BEARER))
                .components(new Components().addSecuritySchemes(BEARER,
                        new SecurityScheme()
                                .type(SecurityScheme.Type.HTTP)
                                .scheme("bearer")
                                .bearerFormat("JWT")));
    }
}
```

3. Controller에는 `@Tag`, `@Operation`, `@ApiResponses`, `@Parameter`를 붙입니다.

```java
@Tag(name = "Coupon Catalog", description = "쿠폰 상품/정책 관리 API")
@RestController
@RequestMapping("/api/v1/catalog/coupons")
public class CouponCatalogController {

    @Operation(summary = "쿠폰 상품 조회", description = "쿠폰 상품 ID로 상세 정보를 조회합니다.")
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "조회 성공"),
            @ApiResponse(responseCode = "404", description = "쿠폰 상품 없음")
    })
    @GetMapping("/{couponId}")
    public CouponResponse getCoupon(
            @Parameter(description = "쿠폰 상품 ID", example = "CPN-2026-0001")
            @PathVariable String couponId
    ) {
        return couponService.getCoupon(couponId);
    }
}
```

4. Request/Response DTO에는 필드 단위 `@Schema`를 작성합니다.

```java
public record CouponResponse(
        @Schema(description = "쿠폰 상품 ID", example = "CPN-2026-0001")
        String couponId,

        @Schema(description = "쿠폰 이름", example = "신규 가입 10% 할인")
        String name
) {
}
```

Controller Swagger 작성 기준:

- API summary는 화면 목록에서 의미가 드러나도록 짧게 작성합니다.
- description에는 업무 조건, 권한, 주요 예외를 적습니다.
- path/query/header parameter에는 `description`과 `example`을 넣습니다.
- request/response DTO 필드는 `@Schema`로 의미와 예시를 남깁니다.
- 에러 응답은 공통 `ProblemDetail` 포맷을 기준으로 문서화합니다.
- 인증이 필요한 API는 전역 `bearerAuth` 스킴을 사용합니다.

작성 후 확인:

```bash
open http://localhost:8081/v3/api-docs
open http://localhost:8081/swagger-ui.html
open http://localhost:9010
```

#### Kafka 통신/연동 문서화 방법

Swagger/OpenAPI는 HTTP API 문서화 표준입니다. Kafka topic, event payload, producer/consumer 계약은 OpenAPI에 억지로 넣지 않고 AsyncAPI 또는 서비스별 `docs/events.md`에 이벤트 계약으로 관리합니다. Swagger Hub에는 HTTP API를 노출하고, Kafka 연동은 Kafka UI와 이벤트 계약 문서로 검증합니다.

Kafka 이벤트 계약 문서에 반드시 남길 항목:

| 항목 | 설명 |
| --- | --- |
| Topic | 발행/구독 topic 이름과 DLQ topic |
| Producer | 이벤트를 발행하는 서비스와 발행 시점 |
| Consumer | 이벤트를 구독하는 서비스와 처리 목적 |
| Key | partition key 기준 |
| Event type | `eventType` 값 |
| Payload schema | JSON field, type, required 여부, example |
| Retry/DLQ | 재시도 정책과 DLQ 이동 조건 |
| Idempotency | 중복 수신 처리 기준 |

권장 이벤트 envelope:

```json
{
  "eventId": "uuid",
  "eventType": "coupon.creation.requested",
  "aggregateId": "CPN-2026-0001",
  "occurredAt": "2026-06-30T10:00:00Z",
  "producer": "coupon-catalog",
  "traceId": "trace-id",
  "payload": {
    "couponId": "CPN-2026-0001",
    "name": "신규 가입 10% 할인"
  }
}
```

Kafka producer/consumer 코드에는 topic 이름을 하드코딩하지 않고 설정값 또는 상수로 분리합니다.

```java
@Component
public class CouponEventProducer {
    private final KafkaTemplate<String, CouponEvent> kafkaTemplate;
    private final String topic;

    public CouponEventProducer(
            KafkaTemplate<String, CouponEvent> kafkaTemplate,
            @Value("${app.kafka.topics.coupon-creation-events}") String topic
    ) {
        this.kafkaTemplate = kafkaTemplate;
        this.topic = topic;
    }

    public void publish(CouponEvent event) {
        kafkaTemplate.send(topic, event.aggregateId(), event);
    }
}
```

```java
@KafkaListener(
        topics = "${app.kafka.topics.coupon-creation-events}",
        groupId = "${spring.kafka.consumer.group-id}"
)
public void consume(CouponEvent event) {
    couponEventHandler.handle(event);
}
```

Kafka 문서 작성 후 확인:

```bash
open http://localhost:9000

docker exec cupi-kafka /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

Kafka 이벤트를 HTTP API와 연결해야 하는 경우에는 Controller의 `@Operation.description`에 "이 API 호출 시 `{topic}`으로 `{eventType}` 이벤트를 발행한다"처럼 연결 관계만 적고, 상세 payload 계약은 `docs/events.md` 또는 AsyncAPI 문서로 연결합니다.

### F. Kafka 동작 확인 (단계 7~8)

#### 7. Kafka 정상 구동 확인

컨테이너 상태와 로그:

```bash
cd ~/workspace/cupon-tobe/cupi-gitops/local-dev
docker compose ps kafka kafka-init kafka-ui
docker compose logs --tail=100 kafka
docker compose logs --tail=100 kafka-init
```

Kafka UI:

```bash
open http://localhost:9000
```

Topic 목록 확인:

```bash
docker exec cupi-kafka /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

필수 topic 존재 확인:

```bash
docker exec cupi-kafka /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list | sort | grep -E 'coupon\.(creation|transaction|issue|omss|notification|interface|common|batch)\.events(\.dlq)?'
```

#### 8. Kafka 송수신 확인

테스트용 consumer를 먼저 실행합니다.

```bash
docker exec -it cupi-kafka /opt/bitnami/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic coupon.creation.events \
  --from-beginning
```

다른 터미널에서 producer로 메시지를 넣습니다.

```bash
docker exec -i cupi-kafka /opt/bitnami/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic coupon.creation.events <<'EOF'
{"eventId":"local-smoke-001","eventType":"coupon.creation.smoke","aggregateType":"CouponCreation","aggregateId":"SMOKE-001","producer":"local-shell","traceId":"local-trace-001","occurredAt":"2026-06-30T00:00:00Z","payload":{"message":"kafka local smoke test"}}
EOF
```

consumer 터미널에 같은 JSON이 출력되면 Kafka broker, topic, produce/consume 경로가 정상입니다. Kafka UI에서도 `coupon.creation.events` topic의 message를 확인할 수 있습니다.

호스트 JVM 서비스에서 Kafka에 붙는지 확인할 때는 `KAFKA_BOOTSTRAP_SERVERS=localhost:9092`, 컨테이너 서비스에서 확인할 때는 `KAFKA_BOOTSTRAP_SERVERS=kafka:29092`인지 먼저 확인합니다.

### G. 인증/JWKS와 RBAC 확인 (단계 9)

`coupon-auth`가 떠 있어야 다른 서비스가 JWT 공개키와 RBAC 계약을 기준으로 인증/인가를 수행할 수 있습니다.

```bash
curl http://localhost:8087/api/v1/coupon-auth/ping
curl http://localhost:8087/api/v1/auth/rbac/system-admin-role
```

JWKS 계약 경로는 `/.well-known/jwks.json`입니다. JWT 발급/JWKS 구현이 완료된 서비스 버전에서는 다음 명령으로 공개키 응답을 확인합니다.

```bash
curl http://localhost:8087/.well-known/jwks.json
```

업무 서비스는 JWT의 `roles`/`permissions` claim을 Spring Security authority로 변환합니다. 전역 시스템 관리자는 다음 값으로 통일합니다.

| 항목 | 값 |
| --- | --- |
| Role claim | `SYSTEM_ADMIN` |
| Spring role authority | `ROLE_SYSTEM_ADMIN` |
| Permission authority | `system:admin` |

서비스 코드에서는 `@PreAuthorize("hasRole('SYSTEM_ADMIN')")` 또는 `@PreAuthorize("hasAuthority('system:admin')")`로 API 호출 전 권한을 검사합니다.

### H. 로컬 DB scaffold 사용 (단계 10)

기본은 PaaS DB입니다. 로컬 PostgreSQL scaffold를 강제로 쓰려면 `.env`를 다음처럼 바꿉니다.

```dotenv
POSTGRES_WRITE_HOST=postgres-write
POSTGRES_WRITE_PORT=5432
POSTGRES_WRITE_DB=cupi_write
POSTGRES_WRITE_USER=cupi
POSTGRES_WRITE_PASSWORD=cupi
POSTGRES_READ_HOST=postgres-write
POSTGRES_READ_PORT=5432
POSTGRES_READ_DB=cupi_write
POSTGRES_READ_USER=cupi
POSTGRES_READ_PASSWORD=cupi
POSTGRES_SSL_MODE=disable
```

호스트 JVM에서 로컬 DB scaffold를 쓸 때는 host가 compose service name을 해석하지 못하므로 `POSTGRES_WRITE_HOST=localhost`를 사용합니다.

```bash
docker exec cupi-postgres-write pg_isready -U cupi -d cupi_write
```

### I. 종료와 초기화 (단계 11)

```bash
cd ~/workspace/cupon-tobe/cupi-gitops/local-dev
docker compose down
docker compose down -v   # 로컬 scaffold DB/볼륨까지 삭제
```

자세한 내용은 `cupi-gitops/docs/local-dev.md`를 기준으로 확인합니다.

---

## 4. 개발 문서

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

---

## 5. 개발 원칙

- 실제 비즈니스 로직은 각 서비스 책임 경계 안에서 구현합니다.
- 다른 서비스의 소유 데이터를 직접 수정하지 않습니다.
- 서비스 간 비동기 협력은 Kafka를 우선 사용합니다.
- 강한 동기 호출이 필요한 경우에는 호출 사유와 fallback 정책을 문서화합니다.
- Kotlin 소스코드와 Kotlin DSL은 사용하지 않습니다.
- 운영 Secret 값은 커밋하지 않습니다.
