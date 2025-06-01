---

# Kafka Replication, ISR, 장애복구 메커니즘 이해

---

## 1. Kafka Replication

- `replication-factor`라는 옵션값을 통해 카프카 내 몇 개의 replication을 유지할 것인지 관리자가 옵션 조정 가능하다.
- Repliaction factor 수를 4, 5 이상으로 설정할 수는 있지만 상황에 맞춰서 아래와 같이 설정하는 것을 권장하고 있다.
  | **환경**                     | **권장되는 replication-factor**                          |
  |------------------------------|-----------------------------------|
  | 테스트나 개발 환경         | 1               | 
  | 운영환경 (로그성 메세지로서 약간의 유실 허용) | 2              | 
  | 운영환경 (유실 허용하지 않음) | 3              | 
- 실제로 복제되는 단위는 "토픽"이 아니라 "파티션"이다.
## 2. Leader&Follower
- rabbitmq의 경우 복제본이 2개 있는 경우 하나는 master, 나머지는 mirrored라고 표현하는데, 이를 kafka에서는 leader와 follower라고 표현한다.
- 각 파티션의 복제본 중 하나가 leader, 나머지는 follower가 된다.
- 모든 read/write는 leader를 통해서만 이루어진다. 컨슈머와 프로듀서 모두 leader와만 통신한다.
- follower는 leader의 데이터를 주기적으로 pull(복제)하며, 리더 장애 시 ISR 내 follower 중 하나가 새로운 leader가 된다.
## 3. ISR(In-Sync Replica)
- ISR은 leader와 동기화가 잘 되고 있는 follower들의 집합이다.
- ISR에 속한 follower만 leader가 될 자격이 있다.
- follower는 leader를 계속 바라보면서 consumer들이 kafka에서 데이터를 pull로 가져는 것과 동일한 방법으로 주기적으로 데이터를 pull한다.
- follower가 일정 시간 이상 leader와 동기화하지 못하면, ISR에서 제외되어 leader 승격 자격을 잃는다.
- 프로듀서의 acks=all 설정 시, ISR 내 모든 복제본에 데이터가 기록되어야만 메시지가 커밋된다.
- 커밋된 메시지의 오프셋(high-water mark)은 leader가 follower에게 전달하며, follower는 이 값을 기준으로 커밋 완료 여부를 판단한다.
- 커밋 전 메시지는 컨슈머가 읽을 수 없으며, 데이터 정합성을 보장한다.
<img src="https://github.com/user-attachments/assets/d1b1346a-9436-4984-aa00-1b904f56d1e6" alt="image" width="500">

## 4.Leader와 Follower 간의 Replication 동작
<img src="https://github.com/user-attachments/assets/88effdab-ed28-40be-af16-07e4b841c3e1" alt="image" width="500">

- 동일한 ISR 내에서 leader와 follower는 데이터 정합성을 맞추기 위해 각자의 방식으로 서로를 계속 체크하고 있는 것이라고 볼 수 있다.

<img src="https://github.com/user-attachments/assets/2f44d50f-4ba5-4743-9a36-bf4c2f8ad196" alt="image" width="500">

1. Leader -> 1번 Offset 위치에 두 번째 새로운 메시지인 message2를 Producer로부터 받은 뒤 저장
2. Followers -> 0번 메시지에 대해 replication을 마치면, Leader에게 1번 Offset에 대한 Replication을 요청
3. Leader -> 0번 Offset에 대한 Repliaction 동작이 성공했음을 인지하고 high water mark 증가

<img src="https://github.com/user-attachments/assets/d6389baa-150d-4249-8136-6c5cca0585b1" alt="image" width="500">

4. Leader ->  Leader는 응답에 0번 Offset 메시지가 커밋되었다는 내용 전달
5. Followers -> 0번 Offset 메시지가 커밋되었다는 사실을 알고 Leader와 동일하게 커밋을 표시 및  1번 Offset에 있던 메시지를 Replication
## 3. **장애 복구 및 리더 선출**
- ISR 내 follower가 모두 사라지면 해당 파티션은 일시적으로 leader가 없어 read/write가 불가하다.
- 모든 브로커가 다운된 후 재기동 시, 가장 먼저 올라온 복제본이 leader가 되며(kafka 기본값), 이 경우 빠르게 장애 대응은 가능하지만 일부 데이터 손실이 발생할 수 있다.
- 그러나, 옵션 변경으로 (모든 데이터를 가지고 있을 가능성이 높은) 마지막 leader가 복구될 때까지 대기하도록 할 수도 있다.
## References
- [Kafka 운영자가 말하는 Topic Replication](https://www.popit.kr/kafka-%EC%9A%B4%EC%98%81%EC%9E%90%EA%B0%80-%EB%A7%90%ED%95%98%EB%8A%94-topic-replication/)
- [[Kafka] Replication(리플리케이션)과 Leader(리더)와 Follower(팔로워)](https://colevelup.tistory.com/19)
