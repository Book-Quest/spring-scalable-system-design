# Transactional Outbox와 Transaction Log Tailing의 비교와 도입 사례

## [1] Transactional Outbox와 Transaction Log Tailing 개념

### Transactional Messaging이란?

- 애플리케이션 비즈니스 로직에 의해 데이터베이스를 수정하는 작업과 메시지 큐에 메시지를 발행하는 작업을 원자적으로 수행하여 데이터의 일관성을 보장하는 것
- Transactional Messaging을 달성하기 위한 분산 트랜잭션 방법
    1. Two Phase Commit
    2. Transactional Outbox
    3. Transaction Log Tailing

### Transactional Outbox

- 메시지 큐에 메시지를 바로 보내지 않고 데이터베이스의 outbox 테이블에 넣는 방식
- 데이터베이스 트랜잭션이 커밋되면 주기적으로 아웃박스 테이블의 내용을 읽어 메시지 큐에 보낸다.
- 적어도 한번 이상(at-least-once) 메시지가 성공적으로 전송되었는지 보장 가능

![image](https://github.com/user-attachments/assets/cec9e0eb-6ae1-4d0f-9375-59890c8d43b5)


1. 비즈니스 로직 수행 및 Outbox 테이블 이벤트 기록 : 
    - 비즈니스 로직 실행과 이벤트 메시지 기록을 데이터베이스의 단일 트랜잭션으로 처리한다.
2. Outbox 테이블을 이용한 이벤트 전송 처리 : 
    - Message Relay가 Outbox 테이블에서 미전송 이벤트를 조회하여 Message Broker로 전송하고, Outbox 테이블에 전송 완료 처리를 한다.

***Message Relay : Message Broker로 이벤트를 전송하는 역할*

### Transation Log Tailing

- 데이터베이스의 트랜잭션 로그를 추적 및 분석하고, 아웃박스 테이블의 데이터 변경만을 필터링(Transaction log miner)해 Message Broker에 이벤트를 전달한다.
- CDC기술을 활용하여 데이터의 변경 사항을 다른 시스템에 전송
    - CDC(Change Data Capture)  : 데이터의 변경사항을 추적하는 기술로, 데이터베이스 안에서 일어나는 모든 변화를 감지하고, 이를 각각의 이벤트로 기록하며 이를 이벤트 스트림으로 전달한다.
- MySQL binlog, PostgreSQL WAL, SQL Server Transaction Log 등

![image](https://github.com/user-attachments/assets/0e2e7d27-e2a0-46a8-8a6c-b53e36dea67a)


## [2] 장단점 및 트레이드 오프
### Transactional Outbox의 장단점
[장점]

- 데이터베이스 트랜잭션 커밋이 완료되었다면, Outbox 테이블에 이벤트 정보가 함께 기록되어 있어 이벤트가 유실되지 않는다.
- 이벤트 정보를 더욱 구체적이고 명확하게 정의 가능

[단점]

- 추가적인 outbox 테이블 생성 및 관리가 필요하다

### Transaction Log Tailing의 장단점

[장점]

- Data Table의 변경 사항을 직접 추적 가능하여 outbox table은 필요하지 않을 수 있다.
- Stream 기반으로 실시간성이 보장된다.
- 이벤트 전송 작업을 직접 개발하지 않을 수 있다.

[단점]

- Data Table은 메시지 정보가 데이터베이스 변경 사항에 종속된 구조로 한정되어 있다.
- 올바르게 활용하려면 새로운 기술(CDC 등)에 대한 학습 및 운영 비용이 필요하다.

### Transactional Outbox과 Transaction Log Tailing의 비교

| 항목 | Transactional Outbox | Transaction Log Tailing |
| --- | --- | --- |
| 구현 난이도 | 낮음 | 높음 |
| 실시간성 | 낮음 (Polling 기반) | 높음 (Stream 기반) |
| DB 부하 | Message Relay 빈도에 따라 증가 | 비교적 낮음 |
| 운영 복잡도 | Outbox 관리 필요 | CDC 도구 운영 필요 |
| 장애 복구 | 명확한 상태 관리 가능 | 복잡한 복구 로직 필요 |

**[1] 구현 난이도** : ✅Outbox < Log Tailing

- **Transactional Outbox** : 기존 애플리케이션 코드 안에서 DB 트랜잭션으로 메시지를 Outbox 테이블에 기록하고, 이후 별도 Relay 프로세스에서 메시지를 Kafka로 전송
    
    → 기존 트랜잭션 처리와 동일한 방식으로 구현 가능
    
- **Transaction Log Tailing** : DB 트랜잭션 로그를 외부에서 읽는 방식으로, CDC 도구 설정, Kafka Connect, Debezium 설정, DB 로그 파싱 처리 등이 필요

**[2] 실시간성** : Outbox < ✅Log Tailing

- **Transactional Outbox** : 주기적 polling 방식을 주로 사용하며, relay 주기에 의해 실시간성이 제한됨
- **Transaction Log Tailing** : CDC 도구가 DB로그를 거의 실시간으로 스트리밍하여 Kafka에 전달하므로 실시간성이 높음

**[3] DB 부하** : Outbox > ✅Log Tailing

- **Transactional Outbox** : 트랜잭션 내에 Outbox 테이블 insert + polling으로 인한 주기적인 select 로 인해 DB 부하 증가
- **Transaction Log Tailing** : DB의 트랜잭션 로그만 읽기 때문에, 실제 테이블을 직접 조회하지 않고 부하가 매우 낮음

**[4] 장애 복구 및 일관성 :**  ✅Outbox >  Log Tailing

- **Transactional Outbox** : Outbox 테이블은 상태필드를 포함해(예: `sent_at`, `status`) 전송 여부를 명확히 관리 가능 ⇒ 메시지 재전송, 복구 시점 파악이 쉬움
- **Transaction Log Tailing :** 로그스트림 기반이므로 장애 발생 시 Kafka topic과 DB 상태 간 offset mismatch 복구가 어려움. 일부 도구는 log position을 수동 조정해야함

**[5] 운영 복잡도** : ✅Outbox <  Log Tailing

- **Transactional Outbox** : DB테이블과 Message Relay만 있으면 되므로 시스템 구조가 단순함. relay 장애, queue 길이 모니터링 등 자체 운영 로직 필요
- **Transaction Log Tailing :** Debezium, Kafka Connect, Schema Registry 등 다수의 구성 요소와 다양한 장애 포인트가 있음

## [3] 현업 사례

### 우아한 형제들 : Transactional Outbox + Transaction Log Tailing
```
딜리버리 서비스 팀
- 배달의 민족의 배민배달 주문을 중계하고 관리하는 역할을 수행
- 분산 시스템 기반 아키텍처와 카프카를 이용해 주문과 배달을 처리
- 분산 서버는 주문/배달 서버와 분석 서버로 구성되고, 처리량과 성능 향상을 위해 각 서버 그룹은 여러 서버로 구성 
```
- 목표 : 주문이 발생하면 고객에게 배달이 완료될 때까지 안전하게 처리하는 것이 가장 큰 목표
- 순서 보장 : 주문과 배달의 이벤트 순서가 중요 → Kafka를 이벤트 브로커로 사용
    - 주문식별자, 배달 식별자 등 순서관리가 필요한 식별자를 Kafka 메시지 키로 사용해, 같은 파티션에 할당되도록하여 배달 상태 변경 이벤트의 순서를 보장
- 데이터 정합성 : 이벤트가 누락되지 않도록 관리 → Transactional Outbox Pattern 사용

**(1) 데이터 정합성 이슈**

- 비즈니스 로직을 처리하기 위한 데이터를 MySQL 데이터베이스에 저장하고, 카프카로 이벤트를 발생하는 방식으로 데이터와 이벤트를 관리
- Kafka에 장애가 생길 경우, 데이터베이스에는 변경된 배달 상태가 저장되었으나 이벤트는 발행되지 않을수도 있다.
    
    ex) 주문 취소로 인해 배달 취소가 발생하면 DB에서 해당 배달은 취소된 상태로 저장된다. 이벤트 발행에 실패하면 Consumer는 메시지를 수신하지 못해 여전히 배달을 진행할 수 있다.

**(2) 요구사항**

- DB의 상태와 메시지 발행의 상태가 불일치하지 않도록 하나의 트랜잭션으로 관리하여 데이터 정합성 보장이 필요하다.
- 메시지 발행 실패 시 재시도가 필요하다. 재시도 과정에서 메시지의 순서 보장과 다른 비즈니스 이벤트 처리에 미치는 영향 최소화가 동시에 필요하다.

![image](https://github.com/user-attachments/assets/e5827c89-d8d9-40c3-9845-7f53e0295ad5)

**(3) 해결 방법 : Transactional Outbox + CDC(Log Tailing) 방식 도입**

1. 트랜잭션 DB에 Outbox 테이블을 도입
2. 비즈니스 로직 수행 시 이벤트를 Outbox에 insert
3. DB 트랜잭션이 성공적으로 커밋되면 Outbox도 함께 반영되며, 커밋된 Outbox 테이블의 insert 작업이 데이터베이스의 Transaction log(binlog)에 기록됨
4. CDC 도구가 binlog를 감지하여, Outbox 테이블의 새로운 레코드를 메시지로 전송

**(4) 구현 방법 : Debezium 라이브러리에서 지원하는 MySQL 카프카 커넥터를 이용**

Debezium의 역할 : binlog를 CDC 방식을 사용하여 DB의 변경사항을 감지하고 이벤트 스트림으로 변환

동작 과정 :

1. binlog 변경 사항을 순서대로 읽고, 설정된 토픽으로 메시지 전송
2. 메시지 발행 실패 시 Outbox 테이블 데이터도 rollback
- Debezium에서 메시지 발행에 사용되는 MySQL source connector는 하나의 task만 동작하도록 설정되어있어 메시지 발행 순서 보장 가능

**(4) 성능 최적화 전략**

문제 : 하나의 task로 동작하면 테이블에 데이터가 쌓이는 속도보다 커넥터가 처리하는 속도가 느릴 경우, 메시지 지연 발생 가능

처리량을 높이기 위해, 토픽별로 outbox 테이블을 분리하여 만든다.

- 각 outbox 테이블은 식별자 기반으로 n개의 테이블로 구성
- 각 테이블에 connector를 연결하여 한 connector가 처리하는 양을 분산하여 처리량을 확보
- 같은 key는 같은 테이블에 저장되며, 한 테이블에서는 하나의 커넥터를 사용하므로 같은 키에 대해서 순서가 보장됨

⇒ Outbox에 쓰고, Debezium이 binlog를 통해 Kafka로 전송하는 CDC 기반 하이브리드 구성
### ridi : Transactional Outbox (polling 방식)
```
- 리디 서비스는 Kafka를 중심으로 통합
- Kafka 도입 초기에 DB 트랜잭션과 메시지 발행을 분리 처리했으나, 원자성 보장 실패, 재시도 지연, 메시지 순서 역전 등의 문제가 발생
- 이를 해결하기 위해 Transactional Outbox 패턴을 도입
```

- Polling Publisher 방식을 사용해 Message Relay를 구현
    - 선택 이유: 구조가 단순하고 구현이 간편함
    - 우려사항: polling으로 인한 DB 부하 (모니터링으로 판단하기로 결정)
    - Transaction Log Tailing 방식의 단점: CDC 도구 학습/운영 비용, 메시지 스키마 관리 복잡성
- Outbox DB 테이블에 polling 하는 것만으로 publish 할 메시지를 가져올 수 있음

**(1) Outbox 테이블 설계**

- `message` 테이블 : 비즈니스 로직과 함께 트랜잭션 안에서 Outbox 테이블에 이벤트 메시지를 저장 (Kafka에 발행할 메시지의 topic, key, payload 등 모든 정보가 포함)
- `processed_message` 테이블 : publish 되거나 publish가 불가능해서 skip하려는 메시지 ID를 저장
    - message 테이블만 존재할 때, 복수의 Message Relay 노드 운영 시 DB Lock 문제 발생하여 추가

**(2) Message Relay 프로세스**

1. 메시지 조회 : 별도의 Message Relay 서비스가 주기적으로 `message` 테이블을 polling하여 처리되지 않은 메시지를 조회
    - 처리 여부는 `processed_message` 테이블을 LEFT JOIN하여 판단
2. Kafka 발행 : 조회된 메시지들을 Kafka로 배치 발행
3. 결과 처리 : 
    - 성공한 메시지 ID와 스킵할 메시지 ID :  `processed_message` 테이블에 저장
    - 실패한 메시지는 재시도 대상으로 분

**(3) 동시성 제어 및 중복 방지**

- 이중 락 메커니즘 사용
    - **Redis Lock**: 서로 다른 노드 간 중복 처리 방지
    - **MySQL Record Lock**: 데이터베이스 레벨 동시성 제어
- **동작 방식:**
    1. Redis lock 획득한 노드만 MySQL record lock 시도 가능
    2. MySQL record lock wait로 인한 DB 부하 최소화
    3. 메시지 중복 처리 완전 방지

**(4) 메시지 삭제 및 성능 고려**

1. 처리 완료된 메시지를 processed_message 테이블에 임시 저장
2. 최근 처리된 메시지 ID로부터 일정 간격 떨어진 오래된 메시지만 삭제 (lock 경합 감소)
- insert와 delete가 접근하는 ID 대역을 분리하여 lock 경합 최소화

### 29cm : Transactional Outbox
- **기존 HTTP 통신의 한계를 넘기 위해 EDA 전환**: 마이크로서비스 고도화 과정에서 동기식 HTTP API만으로는 한계가 있어 Kafka 기반의 Event Driven Architecture 도입을 결정
- CDC가 아닌 Transactional Outbox 선택 이유
    1. 팀 내 CDC 경험 부족 → 안정적 운영에 우려
    2. CDC는 메시지 형식 커스터마이징이 어려움 → Consumer가 추가 로직이 필요
    3. 테이블 스키마 변경에 CDC는 취약 → 서비스 간 독립성 저해


---

#### 참고자료
https://yozm.wishket.com/magazine/detail/2833/

https://devocean.sk.com/blog/techBoardDetail.do?ID=165445&boardType=techBlog

https://techblog.woowahan.com/17386/

https://ridicorp.com/story/transactional-outbox-pattern-ridi/

 https://medium.com/@greg.shiny82/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%94%EB%84%90-%EC%95%84%EC%9B%83%EB%B0%95%EC%8A%A4-%ED%8C%A8%ED%84%B4%EC%9D%98-%EC%8B%A4%EC%A0%9C-%EA%B5%AC%ED%98%84-%EC%82%AC%EB%A1%80-29cm-0f822fc23edb
