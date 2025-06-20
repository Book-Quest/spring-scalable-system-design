## 주요 용어

### Producer

- Kafka로 데이터를 보내는 클라이언트
- 데이터 생산 및 Topic 단위로 전송

### Consumer

- kafka에서 데이터를 읽는 클라이언트
- 데이터 소비 및 Topic 단위로 구독하여 처리

### Broker

- kafka에서 데이터를 중개 및 처리해주는 애플리케이션 실행 단위
- Produce와 Consumer 사이에서 데이터를 주고 받는 역할

### Kafka Cluster

- 여러 개의 Broker가 모여서 하나의 분산형 시스템을 구성한 것
- 대규모 데이터에 대해 고성능, 안정성, 확장성, 고가용성 등 지원 (데이터의 복제, 분산처리, 장애복구 등)

### Topic

- 데이터가 구분되는 논리적인 단위

### Partition

- Topic이 분산되는 단위
- 각 Topic은 여러 개의 Partition으로 분산 저장 및 병렬 처리
- 각 Partition 내에서 데이터가 순차적으로 기록되므로, Partition 간에는 순서가 보장되지 않음
- Partition은 여러 Broker에 분산되어 Cluster의 확장성을 높임

### Offset

- 각 데이터에 대해 고유한 위치
- 데이터는 각 Topic의 Partition 단위로 순차적으로 기록, 기록된 데이터는 offset을 가짐
- Consumer Group은 데이터를 어디까지 읽었는지 각 그룹이 처리한 Offset을 관리

### Consumer Group

- 각 Topic의 Partition 단위로 Offset 관리
- Group 내의 Consumer들은 데이터를 중복해서 읽지 않을 수 있음
- Group 별로 데이터를 병렬로 처리할 수 있음

## 설계 방식

### API

- 각 서비스의 데이터 변경이 생기면, 인기글 서비스로 API를 이용한 이벤트 전송
- 구현은 간단하지만, 타 서비스에 직접적으로 의존하게 되고, 시스템간의 결합도도 증가함
- 서버 부하를 전파할 수 있고, 장애 전파, 유실 등의 위험 높음

### Message Broker

- 서비스의 데이터 변경이 생기면 메시지 브로커로 전송하고, 인기글 서비스에서 이벤트를 가져와서 처리
- 대규모 데이터를 안전하게 처리하게 위한 시스템이 필요해 구현이 복잡함
- 타 서비스에 메시지 브로커 이용을 통한 간접적 의존성을 가지고, 시스템 간 결합도를 감소시킴
- 인기글 서비스는 메시지 브로커에서 적정하게 이벤트를 가져와 비동기 처리 가능
- 메시지 브로커로 이벤트만 전송하면 되기 때문에 서비스에 장애를 전파하거나 유실 등의 위험이 낮음

## Event Driven Architecture

- 이벤트를 주고 받으며 마이크로서비스간에 통신을 하고 서로의 행동을 야기하는 아키텍쳐
- 시스템 내에서 발생하는 이벤트를 중심으로 컴포넌트들이 비동기적으로 처리, 느슨하게 결합되는 구조

---

### 🔄 동작 흐름 예시

```scss
[사용자] → 주문 요청 → [주문 서비스]
                             ↓
                     OrderCreated 이벤트 발행
                             ↓
             Kafka (브로커를 통해 이벤트 전파)
                             ↓
     ┌─────────────┬──────────────┐
     ↓             ↓              ↓
[결제 서비스] [재고 서비스] [알림 서비스] (이벤트 소비자)
```

### 🎯 장점

- **확장성**: 새로운 소비자를 추가해도 기존 Producer 변경 없음
- **유연성**: 느슨한 결합으로 유지보수 용이
- **비동기 처리**: 빠른 응답 가능 (백그라운드 처리)
- **실시간 대응 가능**: 이벤트 스트림 기반 실시간 시스템 구축 가능

### ⚠️ 단점

- **디버깅 어려움**: 흐름이 분산돼 있어서 추적이 어려움
- **복잡성 증가**: 메시지 중복 처리, 순서 보장 등 고려 필요
- **데이터 일관성 유지 어려움**: 트랜잭션 경계가 모호해짐

---

## Sorted Set(zset)

- Set처럼 중복 없는 원소들을 저장하면서, 각 원소에 점수를 부여해 자동 정렬될 상태로 관리하는 자료구조
- Redis에서는 zset이라고 함
- value + score로 구성, value는 중복 불가, score는 중복 가능
- score 오름차순으로 정렬되며,  Skip List + Hash Table(O(log N), 삽입/삭제/조회 기준)

### 🧩 구조 예시

```
zadd leaderboard 100 "Alice"
zadd leaderboard 200 "Bob"
zadd leaderboard 150 "Charlie"
```

- 저장된 결과 (score 기준 자동 정렬):

```
Alice (100)
Charlie (150)
Bob (200)
```

- `zrange leaderboard 0 -1` → `[Alice, Charlie, Bob]`

### 💡 활용 사례

| 활용 분야 | 설명 |
| --- | --- |
| **게임 랭킹 시스템** | 사용자 점수에 따라 자동 정렬 |
| **우선순위 큐** | score를 우선순위로 활용 |
| **시간순 정렬 데이터** | score에 timestamp 사용해서 최근 순으로 정렬 |
| **SNS 피드 정렬** | 게시물 인기/시간 기준 정렬 |
