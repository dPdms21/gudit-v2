# Gudit v2

> **한정 수량 굿즈의 안정적인 선착순 구매와 결제를 위한 타임세일 서비스**

![Kotlin](https://img.shields.io/badge/Kotlin-7F52FF?style=flat-square&logo=kotlin&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![Toss Payments](https://img.shields.io/badge/Toss%20Payments-0064FF?style=flat-square&logoColor=white)
![k6](https://img.shields.io/badge/k6-7D64FF?style=flat-square&logo=k6&logoColor=white)

Gudit v1에서 동시성·정합성 문제를 개선한 이후에도 **DB Commit 이후 Redis 재고 복구 실패**, **외부 결제 보상 취소 실패**와 같이   
로컬 Transaction만으로 복구하기 어려운 장애 구간이 남아 있었다.

Gudit v2에서는 이를 해결하기 위해 **Transactional Outbox + Redis Streams 기반의 재처리 구조**를 적용하고,  
`eventId` 기반 멱등성으로 중복 복구를 방지했다.

또한 기존 비즈니스 계약을 유지하면서 Production 코드의 **98.1%를 Kotlin으로 전환하고 회귀 검증을 수행했다.**

| 항목 | 내용 |
| --- | --- |
| 개발 기간 | 2026.09.11 ~ 2026.09.18 |
| 개발 인원 | 3명 |
| 담당 역할 | 구매·결제 백엔드, Outbox·Redis Streams, AI CS, Kotlin 전환 |
| 이전 프로젝트 | [Gudit v1](https://github.com/dPdms21/gudit-v1) |
| 원본 저장소 | [Gudit 3차 팀 프로젝트](https://github.com/prgrms-be-devcourse/NBE11-13-3-Team03) |

---

## 1. Project Overview

### 1.1. 프로젝트 배경

Gudit은 한정 수량 굿즈를 판매하는 타임세일 서비스다.

선착순 판매에서는 짧은 시간 동안 동일한 재고와 구매·결제 데이터에 요청이 집중되므로  
단순 CRUD 구현만으로는 다음 문제를 안정적으로 처리하기 어렵다.

- 동시 구매에 따른 초과 판매
- 동일 구매·결제에 대한 중복 처리
- Redis 재고와 RDB 구매 상태 간 불일치
- 외부 결제 승인 결과와 내부 Purchase·Payment 상태 간 불일치
- 결제 승인·구매 취소·Timeout 간 상태 경쟁
- 장애 이후 실패한 복구 작업의 유실

Gudit v1에서는 Redis 기반 재고 처리, 비관적 락, Toss Payments 보상 처리 등을 통해 주요 동시성·정합성 문제를 개선했다.

그러나 DB Transaction이 완료된 이후 발생하는 외부 시스템 장애까지 하나의 Transaction으로 처리할 수는 없었다.  
이에 v2에서는 **실패한 복구 작업을 유실하지 않고 다시 처리할 수 있는 구조**를 만드는 데 집중했다.

### 1.2. Gudit v1 → v2

| v1에서 적용한 처리 | 남은 한계 | v2 개선 |
| --- | --- | --- |
| DB Commit 이후 Redis 재고 복구 | Redis 복구 자체가 실패하면 재처리할 수단이 없음 | Transactional Outbox로 복구 요청을 영속화하고 Redis Streams로 재처리 |
| Toss 승인 후 내부 처리 실패 시 즉시 보상 취소 | 보상 취소까지 실패하면 후속 복구 수단이 없음 | `PAYMENT_COMPENSATION_REQUIRED` 이벤트를 저장하고 비동기로 재처리 |
| 동기 또는 일회성 복구 처리 | 장애 이후 동일 작업이 재실행되는 상황을 고려한 멱등 처리 부족 | `eventId` 기반 Lua Script로 재고 복구 멱등성 확보 |
| 결제·취소·Timeout의 상태 경쟁 제어 | 실패 이후 Redis 복구 작업의 추적·재처리 수단 부족 | 상태 변경과 복구 요청을 동일 DB Transaction에서 기록 |

v2에서는 새로운 기술의 도입 자체보다 **v1에서 확인한 실패 구간에 복구 가능성과 멱등성을 추가하는 것**을 핵심 목표로 삼았다.

---

## 2. Key Features

| 영역 | 주요 기능 |
| --- | --- |
| 인증 | Kakao OAuth2 로그인, JWT 기반 인증·인가 |
| 상품·판매 | 굿즈 조회, 타임세일 등록·관리, 판매 상태 관리 |
| 구매 | 선착순 구매, 구매 내역 조회·취소, 미결제 구매 Timeout 처리 |
| 결제 | Toss Payments 승인·취소, Webhook 기반 결제 상태 보정 |
| 재고 | Redis 기반 실시간 재고 및 사용자별 구매 수량 관리 |
| 장애 복구 | Transactional Outbox + Redis Streams 기반 재고 복구·결제 보상 재처리 |
| 멱등성 | `eventId` 기반 재고 중복 복구 방지, Toss Idempotency-Key 기반 중복 요청 대응 |
| AI CS | 구매 내역 기반 문의 분석 및 Slack 답변 초안 전달 |

---

## 3. My Contributions

구매·결제 도메인을 중심으로 **외부 결제 시스템과 RDB·Redis 사이의 정합성 문제 및 실패 재처리 구조**를 담당했다.

### 3.1. 담당 영역

| 영역 | 주요 구현 |
| --- | --- |
| Purchase | 구매 생성·조회·취소, 상태 전이, Payment·재고 연동 |
| Payment | Toss 승인·취소, Webhook 상태 보정, 결제 보상 처리 |
| Failure Recovery | Transactional Outbox, Redis Streams, Pending 재처리, 재고 복구 멱등 처리 |
| AI CS | 구매 문의 API, 내부 결제 상태 조회 API, n8n Workflow 연동 |
| Kotlin Migration | Purchase·Payment·Outbox·CS 및 관련 테스트 전환 |
| Testing | Java/Kotlin 동시성·정합성 회귀 검증 |
| Frontend | Thymeleaf 기반 사용자·관리자 화면 및 API 연동 |

### 3.2. 주요 성과

- DB Commit 이후 Redis 복구 실패에 대비해 **복구 요청을 Outbox에 영속화하고 Redis Streams로 재처리**
- Consumer 장애 및 재전달 상황에 대비해 **Pending 재처리와 `eventId` 기반 재고 복구 멱등성 확보**
- Toss 보상 취소 실패 시 **보상 요청을 별도 Outbox에 기록하고 실제 Toss 상태를 재조회해 복구**
- Production 158개 파일 중 **155개를 Kotlin으로 전환해 98.1% 전환**
- Production LOC **8,399 → 7,338줄로 12.6% 감소**
- Java/Kotlin 환경에서 기존 동시성 시나리오를 반복 검증해 **검증 범위 내 기능·정합성 회귀 없음 확인**

### 3.3. 팀 내 역할 구분

| 팀원 | 담당 영역 |
| --- | --- |
| 박예은 | Purchase / Payment / Outbox·Redis Streams / AI CS / Frontend / 동시성·회귀 검증 / Kotlin 전환 / 발표 자료 |
| 신창석 | 팀장 / 인증·인가 / 공통 예외 처리 / Swagger / k6 테스트 / 테스트 자동화 / CI 자동화 |
| 정진협 | 상품 / 판매 / 재고 / 재고 동시성 테스트 / 모니터링·로깅 / 알림센터 / 발표 |

Redis 기반 재고 차감 및 구매 제한 Lua Script는 재고 담당 팀원이 구현했으며,  
본인은 구매·결제 도메인과 연동되는 **재고 복구용 Outbox·Redis Streams·멱등 처리 구조**를 담당했다.

---

## 4. Tech Stack

| Category | Stack |
| --- | --- |
| Language | Kotlin 2.3.21, Java 25 |
| Backend | Spring Boot 4.1.0, Spring MVC, Spring Data JPA, Spring Security |
| Database / Messaging | PostgreSQL, Redis, Redis Streams, Redis Lua Script |
| Reliability / Payment | Transactional Outbox, Toss Payments |
| AI / Automation | n8n, Claude Code CLI, Slack |
| Testing | JUnit, Mockito, k6 |
| Observability | Actuator, Micrometer, Prometheus, Grafana, Loki, Alloy |
| Infrastructure | Docker, Docker Compose |
| Documentation / Build | Springdoc OpenAPI, Swagger, Gradle Kotlin DSL |

---

## 5. System Architecture & Design

### 5.1. System Architecture

![Gudit 시스템 아키텍처](docs/images/system-architecture.svg)

Gudit은 하나의 Spring Boot 애플리케이션에서 인증·상품·판매·재고·구매·결제·CS 도메인을 처리하며,  
도메인별 패키지를 분리한 Monolithic Architecture로 구성했다.

주요 저장소와 외부 시스템은 역할에 따라 분리했다.

| 구성 요소 | 역할 |
| --- | --- |
| PostgreSQL | 사용자·상품·판매·구매·결제 상태 및 Outbox Event 영속화 |
| Redis | 실시간 재고, 사용자별 구매 수량, Redis Streams |
| Toss Payments | 외부 결제 승인·조회·취소 |
| n8n | 주문 CS Workflow 구성 |
| Slack | AI 분석 결과 및 답변 초안을 운영자에게 전달 |

### 5.2. ERD

![Gudit ERD](docs/images/erd.svg)

Purchase와 Payment는 별도의 상태를 관리한다.

외부 결제 상태와 내부 구매 확정 상태를 분리하고,   
결제 승인·취소·Webhook 처리 시 현재 상태를 검증한 뒤 허용되는 상태 전이만 수행하도록 구성했다.

### 5.3. 장애 복구 흐름

```text
Purchase / Payment 상태 변경
              +
       OutboxEvent 저장
              │
              │ 동일 DB Transaction
              ▼
           DB COMMIT
              │
              ▼
       OutboxPublisher
              │
              ▼
        Redis Streams
              │
              ▼
        Consumer Group
              │
              ▼
        Side Effect 처리
              │
              ▼
             ACK
```

Publisher 발행 실패 시 Outbox Event는 `PENDING` 상태로 남아 재발행되며,  
Consumer 처리 실패 시 ACK하지 않아 Redis Streams의 Pending 메시지로 남는다.

이후 Pending 메시지를 다시 claim해 처리함으로써   
DB Commit 이후 발생한 Redis 재고 복구 및 결제 보상 실패를 재처리할 수 있도록 구성했다.

---

## 6. Core Backend Implementation

### 6.1. 구매 및 결제 처리

구매 생성 시 Redis 재고를 먼저 차감한 뒤 Purchase와 Payment를 생성한다.

```text
구매 요청
   ↓
중복 구매 확인
   ↓
Redis 재고 차감
   ↓
Purchase / Payment 생성
   ↓
PENDING_PAYMENT
```

Redis 선차감 이후 DB Transaction이 rollback되는 경우에는   
동일 Transaction의 Outbox 역시 함께 rollback되므로 기존 동기 보상 방식을 유지했다.

```text
Redis 선차감 성공
        ↓
DB Transaction 실패
        ↓
TransactionSynchronization
        ↓
Redis 재고 복구
```

반면 DB Commit 이후 구매 취소·결제 실패·Timeout으로 재고를 복구해야 하는 경우에는  
복구 요청을 Outbox에 남긴 뒤 비동기로 처리하도록 역할을 분리했다.

### 6.2. Transactional Outbox + Redis Streams

구매 취소·결제 실패·Timeout 처리에서는  
Purchase·Payment 상태 변경과 `STOCK_RESTORE_REQUESTED` 이벤트 저장을 동일 DB Transaction에서 수행한다.

```text
Purchase / Payment 상태 변경
              +
STOCK_RESTORE_REQUESTED 저장
              ↓
           COMMIT
              ↓
       OutboxPublisher
              ↓
        Redis Streams
              ↓
    StockRestoreConsumer
              ↓
        Redis 재고 복구
```

Outbox Event에는 `eventId`, `traceId`, `eventType`, `payload`, `status`를 저장한다.

Publisher 발행 실패 시 이벤트는 `PENDING` 상태로 남아 다시 시도되며,   
Consumer 처리 실패 시 ACK하지 않아 Pending 메시지로 남긴다.

이를 통해 DB Commit 이후 Redis 장애가 발생해도 **복구 요청 자체를 잃지 않고 다시 처리할 수 있도록 구성했다.**

### 6.3. `eventId` 기반 멱등 재고 복구

Consumer 재처리 과정에서는 ACK 이전 장애로 동일 이벤트가 다시 전달될 수 있다.

```text
재고 복구 성공
    ↓
ACK 전 장애
    ↓
동일 이벤트 재처리
```

중복 복구를 방지하기 위해 Redis Lua Script에서 다음 작업을 하나의 원자적 연산으로 처리한다.

```text
processed eventId 확인
        ↓
복구 가능 수량 계산
        ↓
재고 및 사용자 구매 수량 복구
        ↓
processed eventId 저장
```

이미 처리된 `eventId`는 다시 재고를 변경하지 않는다.

처리 완료 Key는 다음과 같이 저장하며 현재 24시간 TTL을 사용한다.

```text
stock-restore:processed:{eventId}
```

### 6.4. Toss Payments 보상 처리

외부 Toss 승인과 내부 DB 상태 변경은 하나의 Transaction으로 묶을 수 없다.

Toss 승인 이후 내부 완료 처리에 실패하면 즉시 보상 취소를 시도하고,  
보상 취소까지 실패한 경우 `PAYMENT_COMPENSATION_REQUIRED` 이벤트를 Outbox에 저장해 재처리한다.

```text
Toss 승인
    ↓
내부 DB 처리 실패
    ↓
즉시 보상 취소
    ↓
보상 실패
    ↓
Outbox 저장
    ↓
Redis Streams 재처리
```

재처리 시에는 Toss의 실제 상태를 먼저 조회한다.

- `DONE` → 보상 취소 재시도
- `CANCELED` → 외부 취소 호출 없이 내부 상태만 보정
- 그 외 상태 → 처리 중단 및 재시도 대상으로 유지

이를 통해 외부 상태를 확인하지 않은 채 동일한 취소 요청을 반복하지 않도록 구성했다.

### 6.5. AI 기반 주문 CS 자동화

결제 관련 문의 발생 시 운영자가 Purchase·Payment 상태를 직접 조회하는 과정을 줄이기 위해   
AI 기반 CS Workflow를 구현했다.

```text
구매 문의
   ↓
Spring Boot
   ↓
n8n
   ↓
내부 결제 상태 API
   ↓
Claude Code CLI
   ↓
Slack
```

사용자가 문의를 등록하면 본인의 Purchase인지 검증한 뒤 연결된 `orderId`를 n8n으로 전달한다.

n8n은 `X-INTERNAL-KEY`로 보호된 내부 API를 통해 Purchase·Payment 상태와 결제 금액을 조회하고 AI에 전달한다.

AI Workflow에는 결제 취소·환불·DB 수정 등의 실행 API를 연결하지 않았으며,  
프롬프트에서도 파일 수정·명령 실행·외부 API 호출 등 실제 조치를 수행하지 않도록 제한했다.

따라서 AI는 백엔드가 검증한 주문 상태를 기반으로 상태 분석·권장 조치·고객 답변 초안을 생성하는 보조 역할로 사용했다.

---

## 7. Technical Challenges

### 7.1. DB Commit 이후 Redis 재고 복구 요청 유실

#### 문제 상황

Purchase·Payment 상태와 실시간 재고는 서로 다른 시스템에 저장한다.

```text
PostgreSQL
→ Purchase / Payment 상태

Redis
→ 판매 재고 / 사용자별 구매 수량
```

기존에는 구매 취소 등의 상태 변경을 DB에 반영한 뒤 Redis 재고를 복구했다.

```text
DB 상태 변경
    ↓
DB COMMIT
    ↓
Redis 재고 복구
```

이 구조에서는 DB Commit 이후 Redis 장애가 발생하면   
Purchase는 이미 `CANCELED`이지만 Redis 재고는 복구되지 않은 상태로 남을 수 있다.

더 큰 문제는 Transaction이 종료된 뒤이므로 **실패한 Redis 복구 요청 자체를 다시 실행할 영속적인 정보가 없다는 점**이었다.

#### 요구사항

한정 판매 서비스에서 재고 복구 누락이 반복되면 실제 판매 가능 수량보다 Redis 재고가 작아져 정상 구매 기회를 잃을 수 있다.

따라서 다음 조건이 필요했다.

1. 복구 요청이 유실되지 않아야 한다.
2. Publisher·Consumer 장애 이후 다시 처리할 수 있어야 한다.
3. 동일 이벤트가 재처리되어도 재고가 중복 복구되면 안 된다.

#### 대안 검토

- **직접 Redis 호출 유지**
  
  구현은 단순하지만 DB Commit 이후 Redis 호출이 실패하면 복구 요청이 사라지는 기존 문제를 해결하지 못한다.

- **Kafka**
  
  높은 처리량과 장기 보관, Replay, 다수 Consumer 확장에 강점이 있다.
  
  하지만 현재 비동기 처리 대상은 전체 구매 이벤트가 아니라 구매 취소·결제 실패·Timeout 이후의 복구 이벤트였고, 서비스에서 이미 Redis를 사용하고 있었다.
  
  현재 요구사항에는 Consumer Group과 실패 메시지 재처리가 필요했기 때문에 별도 Kafka 인프라를 추가하기보다 Redis Streams를 선택했다.

- **Redis Pub/Sub**
  
  Subscriber가 처리하지 못한 메시지를 보존하고 다시 처리하기 어렵기 때문에 실패 메시지 재처리가 중요한 현재 요구사항에는 적합하지 않았다.

#### 해결

비즈니스 상태 변경과 Outbox Event 저장을 동일 DB Transaction으로 묶었다.

```text
Purchase / Payment 상태 변경
              +
       OutboxEvent 저장
              ↓
           COMMIT
              ↓
       Redis Streams
              ↓
          Consumer
```

Transaction이 rollback되면 상태 변경과 Outbox가 함께 rollback되고, Commit됐다면 복구 요청은 PostgreSQL에 남는다.

Publisher 발행 실패 시 Outbox는 `PENDING` 상태로 남고,   
Consumer 처리 실패 시 메시지는 ACK되지 않아 Pending 상태에서 다시 처리된다.

#### 새로운 문제 — 재처리에 따른 중복 복구

재시도를 허용하면 다음 상황이 발생할 수 있다.

```text
재고 복구 성공
    ↓
ACK 전 장애
    ↓
동일 이벤트 재처리
    ↓
재고 중복 복구
```

Inbox Pattern도 검토했지만 실제 Side Effect는 Redis에서 발생하고  
처리 기록은 PostgreSQL에 저장되므로 두 작업을 하나의 Transaction으로 묶을 수 없었다.

따라서 별도의 Inbox 기록보다 **Redis 작업 자체를 멱등하게 만드는 방식**을 선택했다.

Redis Lua Script에서 `eventId` 확인, 재고 복구, 처리 완료 기록을 하나의 원자적 연산으로 수행한다.

#### 검증 결과

동일 `eventId`를 반복 처리해도 Redis 재고가 한 번만 복구되는 것을 확인했다.

```text
초기 재고: 99

첫 번째 처리
→ 100

동일 eventId 재처리
→ 100 유지
```

또한 Consumer 처리 실패로 Pending 상태가 된 메시지를 다시 claim한 뒤 정상 처리·ACK되는 흐름을 확인했다.

#### 기존 동기 보상을 유지한 이유

Outbox 적용 이후에도 모든 보상 로직을 비동기로 변경하지 않았다.

```text
Redis 선차감 이후 DB rollback
→ TransactionSynchronization 기반 동기 보상

DB Commit 이후 Redis 복구 필요
→ Transactional Outbox + Redis Streams
```

DB Transaction이 rollback되는 구간에서는 같은 Transaction의 Outbox도 함께 rollback되기 때문이다.

따라서 하나의 복구 방식을 일괄 적용하지 않고 **실패가 발생하는 Transaction 경계에 따라 복구 책임을 분리했다.**

---

### 7.2. Toss 보상 취소 실패 이후 재처리

#### 문제 상황

Toss 승인 이후 내부 Purchase·Payment 완료 처리에 실패하면 즉시 보상 취소를 요청하고 있었다.

```text
Toss 승인
   ↓
내부 처리 실패
   ↓
Toss 보상 취소
```

하지만 보상 취소 자체가 실패하면 외부에서는 결제가 승인됐지만 내부에서는 실패한 상태가 남고,   
이를 다시 처리할 영속적인 요청이 없었다.

#### 해결

즉시 보상 취소를 우선 수행하고,  
해당 요청까지 실패한 경우 `PAYMENT_COMPENSATION_REQUIRED` 이벤트를 별도의 Outbox에 저장하도록 구성했다.

기존 실패 Transaction과 함께 rollback되지 않도록 이벤트 저장에는 `REQUIRES_NEW` Transaction을 사용했다.

```text
Toss 승인 성공
      ↓
내부 처리 실패
      ↓
즉시 보상 취소
      ↓
보상 실패
      ↓
PAYMENT_COMPENSATION_REQUIRED 저장
      ↓
Redis Streams
      ↓
PaymentCompensationConsumer
```

#### 외부 상태 기반 재동기화

재처리 시에는 취소 API를 바로 다시 호출하지 않고 Toss의 실제 현재 상태를 먼저 조회한다.

```text
Toss = DONE
→ 아직 승인 상태
→ 보상 취소 재시도

Toss = CANCELED
→ 외부 취소 완료
→ 외부 API 호출 생략
→ 내부 상태만 보정
```

이를 통해 취소 요청 응답을 받지 못했지만 외부에서는 실제 취소가 완료된 상황에서도 상태를 복구할 수 있도록 했다.

#### 중복 보상 방지

Toss 취소 요청에는 `paymentKey` 기반 Idempotency-Key를 사용한다.

```text
GUDIT_CANCEL_{paymentKey}
```

또한 재처리 전에 Toss 상태를 조회하고, 내부 Payment가 이미 `CANCELED`라면 보상 로직을 종료하도록 구성했다.

따라서 동일 이벤트가 다시 전달되더라도 외부 요청과 내부 상태 변경의 중복 가능성을 낮췄다.

#### 트레이드오프와 한계

재고 복구에서 사용하던 Outbox Publisher와 Redis Streams 인프라를 재사용해 별도의 메시징 인프라를 추가하지 않았다.

대신 이벤트별 Stream과 Consumer를 분리해 재고 복구와 결제 보상의 처리 로직은 독립적으로 유지했다.

현재는 최대 재시도 횟수와 DLQ가 없어 지속적으로 실패하는 이벤트를 별도로 격리하는 정책이 필요하다.

---

## 8. Kotlin Migration & Regression Validation

### 8.1. 전환 목표

기존 비즈니스 기능을 재설계하지 않고   
Java에서 동작하던 Transaction 경계, 상태 전이, 외부 API 계약을 유지한 상태로 Kotlin으로 전환했다.

담당 범위에서는 Purchase, Payment, Outbox, CS와 관련 테스트를 중심으로 전환했다.

Kotlin 문법 적용 자체보다 **기존 동작과 도메인 계약을 깨뜨리지 않는 점진적 전환**을 우선했다.

### 8.2. 전환 범위

| 항목 | Java Before | Kotlin After |
| --- | ---: | ---: |
| 전체 Production 파일 | 158 | 158 |
| Java 파일 | 157 | 3 |
| Kotlin 파일 | 1 | 155 |
| Kotlin 파일 비율 | - | **98.1%** |
| Production LOC | 8,399 | 7,338 |
| LOC 변화 | - | **-1,061 / -12.6%** |

최종 코드에 남아 있는 Java 파일은 성능 테스트 시 서버 도달 여부를 측정하기 위한 `PerformanceProbe` 관련 파일이며,   
Kotlin 전환 누락 파일로 보지 않았다.

### 8.3. 전환 과정에서 해결한 문제

#### 필수 Production 의존성을 nullable로 완화한 문제

Kotlin 전환 과정에서 Mockito 테스트의 Mock 누락으로 인해 필수 Production 의존성을 nullable로 변경한 부분을 발견했다.

테스트를 통과시키기 위해 Production 계약을 약화시키는 대신   
의존성을 다시 non-null로 복원하고 누락된 Mock을 테스트에 추가했다.

이를 통해 **테스트 편의를 위해 Production 타입 계약을 완화하지 않는 원칙**을 유지했다.

#### Mockito와 Kotlin Nullability 충돌

Java용 Mockito matcher는 내부적으로 `null`을 반환할 수 있어 Kotlin의 non-null 파라미터와 충돌했다.

Production 코드를 nullable로 변경하거나 기존 검증을 제거하지 않고  
테스트 전용 matcher wrapper를 사용해 기존 Assertion과 호출 횟수 검증을 유지했다.

#### Nullable과 도메인 상태 불변식

`paymentKey`는 결제 시작 전에는 존재하지 않을 수 있기 때문에 `String?`으로 유지했다.

그러나 보상 이벤트 생성 시점에는 결제 시작이 완료돼 반드시 값이 존재해야 한다.

필드 자체를 non-null로 변경하거나 `!!`를 사용하는 대신 다음과 같이 해당 처리 시점의 상태 불변식을 명시적으로 검증했다.

```kotlin
checkNotNull(payment.paymentKey)
```

#### JPA Entity와 Kotlin 특성

DTO와 값 객체는 Kotlin의 간결한 표현을 활용할 수 있었지만 JPA Entity에는 동일한 방식을 그대로 적용하지 않았다.

Entity는 `data class` 대신 일반 클래스로 유지하고,  
Persistence lifecycle에 따라 필요한 nullable 상태와 생성자 조건을 구분했다.

또한 Kotlin JPA Plugin을 사용해 JPA Proxy 생성과 Entity 동작을 유지했다.

#### Java/Kotlin 상호 운용성

도메인별로 순차 전환하면서 Java와 Kotlin 코드가 일정 기간 공존했다.

이 과정에서 다음 항목을 확인했다.

- Kotlin property와 Java getter 호출 방식 차이
- 기존 Builder 호출부와 Kotlin 생성자 간 호환
- 필요한 Java 호출부의 `@JvmStatic` 처리
- Java 라이브러리와 Kotlin의 Generic 타입 추론
- `Optional` 등 기존 Java API 계약과의 호환

Kotlin 문법을 최대한 사용하는 것보다  
**기존 도메인 계약과 Java/Kotlin 혼합 환경의 호환성을 유지하는 것**을 우선했다.

### 8.4. 전환 결과

| 지표 | Java Before | Kotlin After | 변화 |
| --- | ---: | ---: | ---: |
| Production LOC | 8,399 | 7,338 | **-12.6%** |
| Clean Build 평균 | 176.58s | 193.64s | **+9.7%** |
| Startup 평균 | 22.11s | 19.39s | 측정상 감소 |

Production 코드량은 12.6% 감소했지만 Clean Build 시간은 오히려 증가했다.

Startup 시간은 Kotlin 전환 후 더 짧게 측정됐지만   
Java 측정 당시의 정확한 JDK 패치 버전과 DB·Redis 상태가 완전히 동일하지 않았고  
측정 횟수도 제한적이므로 **Kotlin 전환으로 기동 성능이 개선됐다고 해석하지 않았다.**

따라서 이번 전환의 핵심 성과는 성능 향상이 아니라  
**기존 비즈니스 계약을 유지한 채 Production 코드 대부분을 Kotlin으로 전환하고 회귀 여부를 검증한 것**이다.

### 8.5. 회귀 검증

Kotlin 전환 측정 시점에는 283개 테스트, 이후 최신 통합 코드에서는 290개 테스트를 재실행해 실패 0건을 확인했다.

또한 기존 동시성 시나리오를 Java/Kotlin 환경에서 반복 실행해   
검증 범위 내에서 주요 기능·정합성 동작이 유지되는지 확인했다.

상세한 동시성 검증은 [Testing & Performance](#9-testing--performance)에 정리했다.

---

## 9. Testing & Performance

### 9.1. 동시성 및 회귀 검증

팀에서 k6를 활용해 재고·구매·결제에 대한 동시성 시나리오를 구성하고,  
HTTP 응답뿐 아니라 PostgreSQL·Redis의 최종 상태까지 함께 검증했다.

Kotlin 전환 이후에는 기존 7개 동시성 시나리오를 Java/Kotlin 환경에서 반복 실행해  
검증 범위 내에서 초과 판매, 중복 처리, 최종 상태 불일치 등의 회귀가 발생하지 않음을 확인했다.

결제 승인 및 승인·취소 경합 시나리오 자체는 Gudit v1에서 직접 구현·검증한 항목이며,  
v2에서는 이를 포함한 기존 시나리오를 전환 후 회귀 검증에 활용했다.

### 9.2. 부하 테스트

팀에서 Warm-up, Baseline, Stress, Spike, Soak 등 단계별 k6 부하 테스트를 수행했다.

Kotlin 환경의 Soak Test에서는 약 100 RPS로 30분 동안 요청을 처리했으며  
HTTP 오류는 발생하지 않았지만 일부 `dropped_iterations`가 확인됐다.

따라서 해당 결과를 안정적인 운영 처리량으로 단정하지 않았다.

### 9.3. 테스트 한계

부하 발생기와 애플리케이션을 동일한 로컬 환경에서 실행했기 때문에   
현재 결과만으로 실제 운영 환경의 처리 가능한 VU·RPS를 보장하기는 어렵다.

향후 Load Generator와 Application 환경을 분리하고 JVM·GC·DB Lock 등의 지표를 함께 측정할 필요가 있다.

---

## 10. Limitations & Evolution

현재 구조는 Redis와 PostgreSQL 또는 Toss를 하나의 ACID Transaction으로 묶는 방식이 아니라,  
**eventual consistency와 재처리를 통해 실패 이후 상태를 복구하는 구조**다.

따라서 다음과 같은 운영 수준의 보완 과제가 남아 있다.

### 반복 실패 이벤트의 최종 처리 정책

현재 Consumer 실패 시 Pending 메시지를 다시 처리할 수 있지만,   
지속적으로 실패하는 이벤트를 별도로 격리하는 최종 실패 정책은 없다.

운영 환경에서는 다음 항목을 추가로 검토할 필요가 있다.

- 최대 Retry 횟수
- DLQ 또는 최종 실패 상태
- 운영자 알림
- 수동 복구 절차

### Outbox 발행 및 보관 정책

현재 Publisher는 `PENDING` 이벤트를 주기적으로 조회해 Redis Streams에 발행한다.

이벤트 양이 증가할 경우 다음 항목을 검토할 필요가 있다.

- Batch 조회 및 적절한 Index 구성
- 다중 Publisher 실행 시 경쟁 제어
- 오래된 Outbox Event의 보관·삭제 정책

### 멱등 처리 기록의 보존 기간

재고 복구 완료 여부를 기록하는 Redis Key에는 현재 24시간 TTL을 사용한다.

이는 메모리 사용량과 중복 처리 방지 기간 사이의 타협이며,  
실제 운영에서는 이벤트가 재전달될 수 있는 최대 기간과 Redis 메모리 사용량을 측정한 뒤 적절한 TTL을 결정할 필요가 있다.

### AI CS 장애 처리

현재 n8n Workflow에서는 내부 주문 조회 실패와 AI 응답 JSON 파싱 실패를 구분해 처리한다.

다만 운영 수준에서는 다음 항목을 추가로 보완할 필요가 있다.

- Claude 실행 Timeout 및 실패 정책
- Slack Webhook 실패 재시도
- Workflow 실행 이력 관리
- 반복 실패 이벤트 알림

---

## 11. References

### Related Repositories

| Repository | Description |
| --- | --- |
| [Gudit 2차 팀 저장소](https://github.com/prgrms-be-devcourse/NBE11-13-2-Team03) | 최초 기능 구현 및 동시성 검증 |
| [Gudit v1](https://github.com/dPdms21/gudit-v1) | 2차 프로젝트 개인 포트폴리오 |
| [Gudit 3차 팀 저장소](https://github.com/prgrms-be-devcourse/NBE11-13-3-Team03) | Outbox·Redis Streams 및 Kotlin 전환 |
| [Gudit v2](https://github.com/dPdms21/gudit-v2) | 3차 프로젝트 개인 포트폴리오 |

### Author

**박예은 | Yeeun Park**

Backend Developer

GitHub: [dPdms21](https://github.com/dPdms21)