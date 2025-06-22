## 인기글 설계 시 고려해야 할 점

Producer: 게시글/댓글/좋아요/조회수 서비스

Consumer: 인기글 서비스

<img width="996" alt="image" src="https://github.com/user-attachments/assets/cff8280a-5f7d-4e45-b966-d603b72e5524" />

- 데이터의 일관성, 원자성 관리를 위해 트랜잭션 관리 필요
- kafka는 네트워크를 사용해 복잡한 전송 과정을 처리하는데 이로 인해 장애가 발생할 수 있음 => 데이터 유실 없이 처리해야 함
- Consumer는 이벤트 처리를 정상적으로 모두 완료한 이후 offset을 commit하면, 데이터의 유실 없이 처리할 수 있음.

## Kafka에 이벤트 전달 과정에서 장애를 해결하는 방법

**Producer → Kafka에 이벤트를 전달하는 과정에 장애가 발생한다면?**

1. kafka의 모든 broker가 장애인 경우
2. 네트워크가 순단된 경우

Producer는 kafka로의 데이터 전송과 무관하게 정상 동작해야함.

그렇다면 Producer는 정상 동작하였으나, **kafka로 이벤트 전송이 실패한 경우는?**

- 생산 및 전파해야 하는 이벤트 데이터는 유실되어 데이터 일관성 깨짐
- 즉, Consumer의 게시글 생성 사실을 알지 못함
- 하지만, 비즈니스 로직은 우선 처리해야하지만, 이벤트 전송은 실시간으로 처리될 필요는 없음.
- 나중에 처리 가능 => Eventually Consistency (궁극적 일관성, 지금 당장은 아니더라도 언젠가는 반드시 된다는 보장)

## Producer의 비즈니스 로직 + kafka의 이벤트 전송의 하나의 트랜잭션으로 관리하기

### 일반적으로 트랜잭션을 수행 하는 방법

1. Transaction start
2. 비즈니스 로직 수행
3.  commit or rollback

이 방법은 @Transactional을 이용하면 손쉽게 적용 가능

### 이벤트를 처리하는 방법

- mysql과 kafka는 독립된 시스템이기 때문에 단일 DB(1개의 샤드)의 1개의 트랜잭션이기 때문에, mysql의 상태변경과 kafka로의 데이터 전송을 단일 트랜잭션으로 묶을 수 없음.

- **publishEvent()**를 사용한다고 해도 만약 kafka 장애로 인해 3초간 블로킹 된다면?
    - case 1) kafka의 장애가 mysql로 전파
    - case 2) 트랜잭션 commit이 실패했는데, 이벤트 전송은 이미 완료됐을 수 있음

- 이는 **publishEvent()가** 동기적으로 처리하기 때문임. 그렇다면 비동기인 publishEventAsync()를 사용한다면?
    - 비동기로 수행한다고 해도, 이벤트 전송이 실패한다고 해서 MySQL 트랜잭션이 자동으로 롤백되지는 않음
    - 롤백을 위한 보상 트랜잭션을 직접 수행할 수도 있지만, 역시 누락의 문제가 있어 복잡도 상승

---

## ✅ 보상 트랜잭션(Compensating Transaction)이란?

> 실패한 트랜잭션을 되돌릴 수 없을 때, 성공한 작업을 취소하는 별도의 작업을 수행하는 것.
> 

## ✅ 왜 롤백이 필요할까?

```
1. A 작업 실행
2. B 작업 실행
3. 중간에 실패 → A도 B도 롤백 (DB rollback)
```

하지만 **마이크로서비스 환경**에서는 서비스 A, B, C가 **서로 다른 DB**를 갖고 있음

- 하나는 성공하고
- 다른 하나는 실패했을 때
- 전체 트랜잭션을 **롤백 불가!**

➡ 그래서 **대신 “보상 작업(compensating action)”을 수행해야 함**

## ✅ SAGA 패턴이란?

> 마이크로서비스 환경에서 트랜잭션을 서비스 간으로 나눠서 처리할 수 있게 해주는 패턴
> 
> 
> 실패할 경우, **앞서 성공한 작업들을 보상 트랜잭션으로 되돌려서 정합성을 유지**합니다.
> 

## ✅ 전통적인 트랜잭션 vs SAGA 패턴

| 구분 | 전통 트랜잭션 | SAGA 패턴 |
| --- | --- | --- |
| 범위 | 하나의 DB, 하나의 서비스 | 여러 서비스, 여러 DB |
| 롤백 방식 | DB 롤백 (자동) | 보상 트랜잭션 (수동, 명시적) |
| 일관성 | 즉시 일관성 | 최종 일관성 (Eventually Consistent) |

## ✅ SAGA 패턴의 종류

### 1. **Choreography (이벤트 기반, 메시지 기반)** ⭐️

- 각 서비스가 Kafka 등의 메시지 브로커를 통해 이벤트를 주고받음
- 중앙 조정자 없음
- 서비스가 이벤트를 **구독하고 직접 다음 작업을 트리거**

```
Order Service → emits "OrderCreated"
  ↓
Payment Service → emits "PaymentCompleted"
  ↓
Inventory Service → emits "StockReduced"
  ↓
Shipping Service → emits "ShippingFailed"
  ↓
Inventory Service ← listens "ShippingFailed" → restores stock
Payment Service ← listens "ShippingFailed" → cancels payment
```

---

### 2. **Orchestration (중앙 집중식)**

- **SAGA orchestrator**가 전체 플로우를 지휘
- 각 서비스는 orchestrator의 명령에 따라 수행

```
SAGA Orchestrator → call Order → call Payment → call Stock
                            ↑                    ↑
                   실패 시 보상 명령         실패 시 보상 명령
```

---

## ✅ 비교: 동기 vs 비동기 이벤트 발행

| 구분 | `publishEvent()` | `publishEventAsync()` |
| --- | --- | --- |
| 처리 방식 | 동기 (synchronous) | 비동기 (asynchronous) |
| 이벤트 핸들러 호출 시점 | 즉시 호출하고 끝날 때까지 기다림 | 별도 스레드에서 나중에 처리 |
| 예외 처리 | 호출자에게 예외 전파 | 호출자와 무관, 로그로만 처리 가능 |
| 성능 | 느릴 수 있음 (리스너가 오래 걸리면) | 빠름 (리스너 실행과 무관) |
| 대표 환경 | Spring 기본 이벤트 처리 | Spring + `@Async`, Kafka, MQ 등 |

---

## 분산 시스템 간의 트랜잭션 관리 방법

### Two Phase Commit

- 분산 시스템에서 모든 시스템이 하나의 트랜잭션을 수행할 때, 모든 작업이 성공하면 commit, 하나라도 실패하면 rollback

<img width="629" alt="image (1)" src="https://github.com/user-attachments/assets/41af3ff3-ad64-4259-ac37-35bafa56cad9" />


1. Prepare phase (준비 단계)
- Coordinator는 각 참여자에게 트랜잭션을 커밋할 준비가 되었는지 물어보고, 각 참여자는 이에 응답

1. Commit phase (커밋 단계)
- 모든 참여자가 준비 완료 응답을 보내면, Coordinator는 모든 참여자에게 트랜잭션 커밋을 요청, 모든 참여자는 트랜잭션을 커밋

- 모든 참여자의 응답을 기다려야 하기 때문에 지연 발생 가능성 있음
- Coordinator or 참여자 장애가 발생하면 다른 사람들은 상태를 모른채로 대기 및 트랜잭션 복구 처리가 복잡해질 수 있음

### Transactional Outbox

- 이벤트 전송 작업을 일반 DB 트랜잭션에 포함시킬 수는 없지만, 정보를 DB 트랜잭션에 포함해 기록할 수는 있음
- 트랜잭션을 지원하는 DB에 Outbox 테이블을 생성, 서비스 로직 수행과 Outbox 테이블 이벤트 메시지 기록을 단일 트랜잭션으로 묶음

1. 비즈니스 로직 수행 및 Outbox 테이블 이벤트 기록
    1. Transaction start
    2. 비즈니스 로직 수행
    3. Outbox 테이블에 전송 할 이벤트 데이터 기록
    4. commit or abort

1. Outbox 테이블을 이용한 이벤트 전송 처리
    1. outbox 테이블 미전송 이벤트 조회
    2. 이벤트 전송
    3. Outbox 테이블 전송 완료 처리

<img width="579" alt="image (2)" src="https://github.com/user-attachments/assets/4cbb626f-dcf1-4f74-a927-9b80a2f3c59b" />

- DB 트랜잭션 커밋이 완료되었다면, outbox 테이블에 이벤트 정보가 함께 기록되었기 때문에, 이벤트가 유실되지 않음
- 추가적인 outbox 테이블 생성 및 관리가 필요
- outbox 테이블의 미전송 이벤트를 message broker로 전송하는 작업이 필요

---

## ✅ Outbox Table이란?

> 내 서비스의 DB 트랜잭션과 이벤트 발행을 일치시키기 위한 중간 저장 테이블
> 

즉, 이벤트를 Kafka에 **바로 발행하지 않고**,

**먼저 내 DB에 `outbox`라는 테이블에 저장**한 뒤,

**백그라운드 프로세스(예: Kafka Producer)가 이를 읽고 Kafka로 발행**

---

## ✅ Outbox Table이 필요한 이유

Kafka에 바로 이벤트를 발행하면 문제 발생:

### ❌ 문제 사례

1. 게시글 저장 → DB에는 저장됨
2. Kafka 발행 실패 → 이벤트 사라짐 😱

이렇게 되면 **DB에는 존재하지만, Kafka에는 정보가 없음 → 시스템 불일치 발생**

---

## ✅ Outbox 패턴의 흐름

```
1. 게시글 저장 트랜잭션 시작
2. 게시글 테이블에 INSERT
3. 동시에 Outbox 테이블에도 INSERT ← 같은 트랜잭션 내
4. 트랜잭션 COMMIT → 둘 다 저장 확정
5. 별도의 프로세스가 Outbox 테이블을 읽고 Kafka에 발행
6. 성공 시 Outbox row 삭제 or 상태 변경
```

💡 핵심: **DB 트랜잭션이 성공해야지만 Kafka 발행도 시도된다.**

---

## ✅ Outbox 테이블 예시 구조

| id | aggregate_type | aggregate_id | event_type | payload (json) | created_at | status |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | POST | 123 | POST_CREATED | {postId: 123, ...} | 2025-06-19 10:00 | PENDING |
| 2 | POST | 124 | POST_LIKED | {postId: 124, ...} | 2025-06-19 10:01 | SENT |

---

## ✅ 구성 예시

- 게시글 서비스는 Kafka Producer가 아님 → 대신 **DB에만 저장**
- Outbox 테이블에 이벤트도 같이 INSERT
- 백그라운드에서 Kafka로 **발행 후 성공 여부 관리**

이 방식은 **Transaction Outbox 패턴**이라고도 하며,

**마이크로서비스 + Kafka 환경에서 권장되는 디자인 패턴**

---

## ✅ 정리

| 항목 | 설명 |
| --- | --- |
| 목적 | DB 변경과 Kafka 발행의 **정합성 보장** |
| 사용 이유 | Kafka 발행 실패로 인한 데이터 손실 방지 |
| 구성 | 서비스 트랜잭션 + Outbox 테이블 → Kafka 발행 프로세스 |
| 관련 패턴 | Transactional Outbox, CDC(Change Data Capture), Debezium |

---

## ✅ message relay란?

> 한 시스템이 수신한 메시지를, 가공하거나 조건에 따라 다른 시스템/토픽으로 다시 전달(중계)하는 구조
> 

즉, 메시지를 직접 처리하지 않고, 메시지를 받아서 다른 쪽으로 다시 보내는 중간 역할

---

## ✅  message relay 구조

```
[Service A] → Kafka Topic A
                     ↓
             [Message Relay Service]
                     ↓
               Kafka Topic B → [Service B]
```

- Service A가 Kafka Topic A에 메시지를 발행
- **Message Relay**는 Topic A의 메시지를 구독(consume)
- 그리고 이를 **가공하거나 필터링**한 뒤 **Topic B로 다시 발행(produce)**
- Service B는 Topic B만 구독해서 처리

---

## ✅ 예시 1: 이벤트 필터링 Relay

- Kafka Topic A: 모든 게시글 이벤트 (`POST_CREATED`, `POST_UPDATED`, `POST_DELETED`)
- 인기글 서비스는 `POST_CREATED`만 필요
- 그래서 Message Relay가 `POST_CREATED`만 걸러서 Topic B로 전달

```
[post-events] → MessageRelay (filter) → [popular-post-events]
```

---

## ✅ 예시 2: 포맷 변환 Relay

- 어떤 서비스는 JSON으로 이벤트를 보내지만
- 소비자는 Avro 또는 다른 포맷을 원할 때

```
[json-topic] → Relay (convert JSON → Avro) → [avro-topic]
```

---

## ✅ 구현 예시

```java
@KafkaListener(topics = "topic-a")
public void onMessage(PostEvent event) {
    // 필요한 가공
    ModifiedEvent modified = convert(event);

    // 다시 전송
    kafkaTemplate.send("topic-b", modified);
}
```

또는 Spring Cloud Stream 등에서는 중간 `Processor`처럼 선언적으로도 사용 가능

---

## ✅ message relay를 사용하는 이유

| 이유 | 설명 |
| --- | --- |
| 포맷 변환 | JSON → Avro 등 |
| 이벤트 필터링 | 필요 없는 이벤트 제거 |
| 보안/검열 | 민감 데이터 제거 후 전달 |
| 시스템 decoupling | A 시스템에서 직접 B를 모르도록 중계 |
| 장애 차단 | 한 시스템 장애 시 중간 캐시 역할 |

---

## ✅ 비슷한 개념

| 용어 | 설명 |
| --- | --- |
| Message Broker | 메시지를 중계하고 큐잉하는 시스템 자체 (Kafka, RabbitMQ 등) |
| Message Router | 메시지를 **조건에 따라** 다른 채널로 보내는 역할 |
| Message Relay | 메시지를 **단순/조건부 중계**하는 서비스 또는 컴포넌트 |

---

## ✅ 정리

- `message relay`는 **메시지를 받아서 다른 쪽으로 다시 보내는 중간 전달자**
- Kafka에서는 Consumer + Producer 역할을 동시에 수행
- 주로 **포맷 변환, 필터링, 복제, 리디렉션, 안전한 분리**를 위해 사용됨
- 서비스 decoupling과 확장성에 매우 유용

+추가자료)

[29cm Transactional Oubox 패턴 실제 구현 사례](https://medium.com/@greg.shiny82/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%94%EB%84%90-%EC%95%84%EC%9B%83%EB%B0%95%EC%8A%A4-%ED%8C%A8%ED%84%B4%EC%9D%98-%EC%8B%A4%EC%A0%9C-%EA%B5%AC%ED%98%84-%EC%82%AC%EB%A1%80-29cm-0f822fc23edb)

---

### Transaction Log Tailing

- DB 트랜잭션 로그를 추적 및 분석하는 방법 ⇒ DB는 각 트랜잭션의 변경 사항을 로그로 기록
- 이러한 로그를 읽어서 Message Broker에 이벤트를 전송할 수 있음
    - CDC(Change Data Capture): 데이터 변경 사항을 추적하는 기술
    - CDC를 기술을 활용해 데이터의 변경 사항을 다른 시스템에 전송
  
<img width="587" alt="스크린샷 2025-06-21 16 42 28" src="https://github.com/user-attachments/assets/9b57dc96-7120-4e70-b810-049d3b30fff1" />

- DB에 저장된 트랜잭션 로그를 기반으로, Message Broker로의 이벤트 전송 작업을  구축하기 위한 방법으로 활용 가능
- 테이블을 직접 추적하면, Outbox table은 사용하지 않아도 됨
- but, 트랜잭션 로그를 추적하기 위해 CDC 기술을 활용해야 함 ⇒ 러닝커브 + 운영 비용 발생

---

## 📊 세 가지 방법 비교

| 항목 | Outbox 패턴 | CDC (로그 테일링) | 2PC |
| --- | --- | --- | --- |
| **정합성** | ✅ 높음 (DB와 같은 트랜잭션) | ✅ 높음 (DB 로그 기반) | ✅ 매우 높음 (완전 원자적) |
| **복잡도** | 보통 (코드 + 프로세스 추가) | 높음 (CDC 도구 설치, 로그 권한 등) | 매우 높음 (트랜잭션 매니저, XA 등) |
| **성능** | ✅ 빠름 | ⚠ 약간 느릴 수 있음 | ❌ 느림 (락, 대기, 분산 커밋) |
| **언어/환경 의존성** | 없음 (모든 언어 사용 가능) | DB 종류 및 CDC 툴 의존 | DB/Kafka/미들웨어 모두 지원 필요 |
| **재시도/보장** | ✅ 메시지 재발행 가능 | ✅ Kafka로 메시지 보냄 | ✅ 완전한 보장 |
| **장애 복원력** | ✅ 높음 | ✅ 높음 | ❌ 낮음 (장애 시 롤백 위험) |
| **실시간성** | ✅ 높음 (거의 즉시) | ⚠ 약간 지연 (ms~s) | ✅ 즉시 |
| **운영 난이도** | 보통 | 높음 (Debezium, Kafka 연동 등) | 매우 높음 |

## 🏆 가장 좋은 방법은?

| 상황 | 추천 방식 |
| --- | --- |
| 현대적인 마이크로서비스, Kafka 기반 | ✅ **Outbox 패턴** (가장 일반적, 안정적) |
| 레거시 시스템을 CDC 기반으로 Kafka 연동 | ✅ **CDC (Log Tailing)** |
| 정합성이 생명이고 성능 상관 없음 (금융 등) | ✅ **2PC**, 그러나 일반적으로는 **비권장** |

👉 **대부분의 경우에는 `Outbox + CDC (Debezium)` 조합이 실용성과 안정성을 모두 갖춘 조합**

---

## 💡 보너스: Outbox + CDC 조합 방식

1. 서비스는 Kafka에 직접 안 보내고 Outbox 테이블에 INSERT
2. Debezium이 Outbox 테이블만 tailing 해서 Kafka에 발행
3. 정합성 + 자동화 + 운영성 모두 확보 가능

이 방식은 대기업(쿠팡, 배민, 우아한형제들 등)에서도 널리 사용

출처:

[우리 팀은 카프카를 어떻게 사용하고 있을까](https://techblog.woowahan.com/17386/)

[회원시스템 이벤트기반 아키텍처 구축하기](https://techblog.woowahan.com/7835/)
