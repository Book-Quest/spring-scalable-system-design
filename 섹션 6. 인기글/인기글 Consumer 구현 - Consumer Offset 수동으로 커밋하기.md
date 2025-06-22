----
# 인기글 Consumer 구현 - Consumer Offset 수동으로 커밋하기  
----
## 1. 자동 커밋(`enable.auto.commit=true`)의 동작과 한계
- **커밋 트리거 메커니즘**
  - Kafka의 컨슈머는 읽은 메시지의 위치를 추적하기 위해 offset을 commit 하는 역할을 담당한다.
  - 이 offset은 Kafka의 내부 토픽인 __consumer_offsets에 저장되며, 컨슈머는 commit이 발생할 때마다 이 offset 값을 갱신 후, 이후 컨슈머는 이 offset 값을 참조하여 다음에 처리할 레코드를 읽어온다.
  - spring Boot 2.2에서 사용하는 버전인 2.3부터 기본적으로 `spring.kafka.consumer.enable-auto-commit: false`로 되어있으나, true로 옵션 사용 시 poll() 메서드가 실행되었을 때 수행 시간이 auto.commit.interval.ms 설정값을 초과하는 경우 자동으로 offset이 commit 된다.
   <img src="https://github.com/user-attachments/assets/4ab0d455-b2a8-44b5-b17b-fbb15876eb38" width="600"/>

  ```java
    public void maybeAutoCommitOffsetsAsync(long now) {
        if (autoCommitEnabled) {
            nextAutoCommitTimer.update(now);
            if (nextAutoCommitTimer.isExpired()) {
                nextAutoCommitTimer.reset(autoCommitIntervalMs);
                doAutoCommitOffsetsAsync(); // 비동기 커밋 실행
            }
        }
    }
    ```
    - `poll()` 호출 시마다 `maybeAutoCommitOffsetsAsync()` 실행
    - `auto.commit.interval.ms`(기본 5초)마다 커밋

- **문제점**
    | 시나리오         | 메커니즘                                         | 결과                   |
    |------------------|--------------------------------------------------|------------------------|
    | 유실 발생        | 처리 시간 > 커밋 간격 → 미처리 메시지 커밋        | 장애 시 데이터 유실    |
    | 중복 발생        | 리밸런싱 시 커밋되지 않은 오프셋 재할당           | 동일 메시지 재처리     |

---

## 2. Spring Kafka 수동 커밋 구현

### 2.1 설정 단계

- **컨슈머 팩토리**
    ```java
    @Bean
    public ConsumerFactory consumerFactory() {
        Map configs = new HashMap<>();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configs.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        configs.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // 자동 커밋 비활성화
        return new DefaultKafkaConsumerFactory<>(configs);
    }
    ```

- **컨테이너 팩토리**
    ```java
    @Bean
    public ConcurrentKafkaListenerContainerFactory kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AckMode.MANUAL); // 수동 커밋 활성화
        factory.setConcurrency(3); // 병렬 처리
        return factory;
    }
    ```

### 2.2 리스너 구현 패턴

- **단일 레코드 처리**
    ```java
    @KafkaListener(topics = "test-topic")
    public void listen(
        ConsumerRecord record,
        Acknowledgment acknowledgment
    ) {
        try {
            processMessage(record.value());
            acknowledgment.acknowledge(); // 수동 커밋
        } catch (Exception e) {
            log.error("Message processing failed", e);
        }
    }
    ```

- **배치 처리**
    ```java
    @KafkaListener(topics = "test-topic", batch = "true")
    public void listenBatch(
        List records,
        Acknowledgment acknowledgment
    ) {
        for (ConsumerRecord record : records) {
            processMessage(record.value());
        }
        acknowledgment.acknowledge(); // 전체 배치 성공 시 커밋
    }
    ```

---

## 3. AckMode 상세 동작 비교

| 모드               | 커밋 시점                       | 오프셋 전송 주체         | 사용 사례           |
|--------------------|---------------------------------|--------------------------|---------------------|
| MANUAL             | `acknowledge()` 호출 후 다음 poll| ConsumerCoordinator      | 안전성 중시 시스템  |
| MANUAL_IMMEDIATE   | `acknowledge()` 즉시            | KafkaConsumer.commitSync | 저지연 요구 시스템  |
| BATCH              | 모든 레코드 처리 후 자동         | Container                | 단순 배치 처리      |
| RECORD             | 레코드 단위 처리 직후           | Container                | 실시간 스트림       |

---

## 4. 주의사항 및 리밸런싱 대응

- **커밋 누락 위험**
    ```
    @KafkaListener(...)
    public void listen(..., Acknowledgment ack) {
        processMessage();
        // ack.acknowledge() 누락 → 오프셋 미갱신
    }
    ```
    - 동일 메시지 무한 재처리 → Consumer Lag 증가

- **리밸런싱 대응**
    ```
    factory.getContainerProperties().setConsumerRebalanceListener(
        new ConsumerAwareRebalanceListener() {
            @Override
            public void onPartitionsRevoked(
                Collection partitions,
                Consumer consumer
            ) {
                consumer.commitSync(); // 리밸런싱 직전 강제 커밋
            }
        }
    );
    ```
    - 파티션 해제 시점에 명시적 커밋 → 중복 처리 최소화

---

### References
- https://lsj8367.tistory.com/entry/Kafka-Offset-Commit%EC%9D%98-%EC%A4%91%EC%9A%94%EC%84%B1
- https://dkswnkk.tistory.com/744
- https://oliveyoung.tech/2024-10-16/oliveyoung-scm-oms-kafka/
