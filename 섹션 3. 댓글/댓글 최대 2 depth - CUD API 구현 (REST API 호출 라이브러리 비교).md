## RestClient vs WebClient vs RestTemplate: 심층 비교 및 Kotlin 환경 고려사항

### 1. RestTemplate

RestTemplate은 동기식 HTTP 클라이언트로 Spring Framework 3.0부터 제공된 전통적인 HTTP 요청 방식이다.

#### 특징

- **동기식 요청 처리**: 요청이 완료될 때까지 스레드가 차단된다.
- **템플릿 기반 API 제공**: `getForObject()`, `getForEntity()`, `exchange()`, `execute()` 등 다양한 메서드 지원한다.
- **레거시 코드 호환성**: Spring의 이전 버전에서도 사용 가능하다.
- **선언적 HTTP 인터페이스 지원**: `@HttpExchange`를 활용하여 인터페이스 기반 호출 가능하다.

#### 사용 예시

```java
RestTemplate restTemplate = new RestTemplate();
String response = restTemplate.getForObject("https://api.example.com/data/{id}", String.class, 42);
System.out.println("응답 데이터: " + response);
```

### 2. WebClient

WebClient는 Spring Framework 5.0에서 도입된 비동기 및 리액티브 HTTP 클라이언트로, 함수형 API를 제공한다.

#### 특징

- **비동기 및 논블로킹 I/O 지원**: `Mono`와 `Flux`를 활용하여 Reactive Streams 기반 동작한다.
- **함수형 및 플루언트 API 제공**: 메서드 체이닝 및 람다 활용 가능하다.
- **WebFlux와 최적화된 통합**: 리액티브 프로그래밍 환경에서 효율적인 처리 가능하다.
- **Kotlin WebFlux와의 자연스러운 통합**: 리액티브 프로그래밍을 위한 `Mono`, `Flux` 기반의 API 활용 가능하다.
- **선언적 HTTP 인터페이스 지원**: `@HttpExchange`를 통한 인터페이스 기반 호출 가능하다.

#### 사용 예시

```java
WebClient webClient = WebClient.builder().baseUrl("https://api.example.com").build();
String result = webClient.get().uri("/data/{id}", 42).retrieve().bodyToMono(String.class).block();
System.out.println("응답: " + result);
```

### 3. RestClient

Spring Boot 3.2에서 도입된 최신 HTTP 클라이언트로, RestTemplate과 WebClient의 장점을 결합한 API를 제공한다.

#### 특징

- **동기 및 비동기 요청 지원**: WebFlux 없이도 비동기 호출 가능하다.
- **플루언트 API 제공**: WebClient와 유사한 함수형 스타일 지원한다.
- **선언적 HTTP 인터페이스 지원**: `@HttpExchange`를 활용한 인터페이스 기반 호출 가능하다.
- **Virtual Threads와의 자연스러운 통합**: JDK 21 이상의 Virtual Threads와 결합 가능하다.

#### 사용 예시

```java
RestClient restClient = RestClient.builder().baseUrl("https://api.example.com").build();
String result = restClient.get().uri("/data/{id}", 42).retrieve().body(String.class);
System.out.println("응답: " + result);
```

### 4. Kotlin 환경에서의 고려사항

### WebClient + WebFlux (병렬 처리 및 백프레셔 관리)

- WebFlux는 논블로킹 방식의 리액티브 프로그래밍을 지원하며, `Mono` 및 `Flux`를 활용하여 병렬 요청을 효율적으로 처리할 수 있다.
- `flatMap()`을 사용하여 여러 개의 비동기 요청을 동시에 실행하고 결과를 조합할 수 있다.
- **백프레셔 관리**: `onBackpressureBuffer()` 또는 `onBackpressureDrop()`을 활용하여 데이터 과부하를 방지하고, 소비 속도에 맞춰 데이터 처리를 조절할 수 있다.
- `parallel()`과 `runOn(Schedulers.parallel())`을 조합하여 CPU 활용을 극대화할 수 있다.

```kotlin
fun fetchParallelData(): Flux<String> {
    return Flux.just("/api/1", "/api/2", "/api/3")
        .flatMap { uri ->
            webClient.get().uri(uri).retrieve().bodyToMono(String::class.java)
        }
        .onBackpressureBuffer(10) // 백프레셔 버퍼 설정
        .subscribeOn(Schedulers.parallel())
}

fetchParallelData().subscribe {
    println("응답 데이터: $it")
}
```

### RestClient + Virtual Threads

- Virtual Threads를 활용하여 동기식 코드 스타일 유지 가능하다.
- Virtual Threads는 JDK 21 이상에서 지원되는 경량 스레드 모델로, 스레드 풀 없이 수천 개의 동시 요청하며, I/O 바운드 작업에서 효율적이다. (예: 네트워크 요청, DB 조회).)
- 그러나, Threads는 백프레셔 관리 기능 부족으로, 요청이 너무 많으면 과부하 발생할 수 있다.
- 또한, 스레드 컨텍스트 스위칭 비용으로 인해 CPU 바운드 작업에는 적합하지 않다.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> results = List.of(
        executor.submit(() -> restClient.get().uri("/api/1").retrieve().body(String.class)),
        executor.submit(() -> restClient.get().uri("/api/2").retrieve().body(String.class)),
        executor.submit(() -> restClient.get().uri("/api/3").retrieve().body(String.class))
    );
    results.forEach(result -> System.out.println(result.get()));
}
```

### 5. 비교

| 특징                | RestTemplate | WebClient (WebFlux) | RestClient (Virtual Threads) |
| ----------------- | ------------ | ------------------- | ---------------------------- |
| Spring 도입 버전      | Spring 3.0   | Spring 5.0          | Spring Boot 3.2              |
| 동기식 지원            | O            | 제한적                 | O                            |
| 비동기 및 리액티브 지원     | X            | O                   | 제한적                          |
| 병렬 요청 최적화         | X            | O                   | O                            |
| 백프레셔 관리           | X            | O                   | X                            |
| 플루언트 및 함수형 API    | X            | O                   | O                            |
| 선언적 HTTP 인터페이스 지원 | O            | O                   | O                            |

### 6. 결론

- **WebClient + WebFlux**: Kotlin 환경에서 고성능 논블로킹 요청을 병렬 처리하기에 적합하며, 백프레셔 관리가 가능하여, 대량의 데이터를 효율적으로 처리해야 하는 경우 적합하다.
- **RestClient + Virtual Threads**: 기존 동기식 코드 스타일을 선호하거나 I/O 바운드 작업 위주의 애플리케이션에서 유용하다.
- **RestTemplate**: 레거시 코드 유지 필요 시 사용되며, 새로운 프로젝트에서는 WebClient나 RestClient 사용이 권장된다.

### References
- [RestClient vs. WebClient vs RestTemplate: Choosing the right library to call REST API in Spring ‌Boot](https://digma.ai/restclient-vs-webclient-vs-resttemplate/)
- [Spring RestClient와 한계 (코루틴과의 호환성)](https://hui0221.tistory.com/12)
- [[Spring] Spring6에 등장한 HttpInterface에 대한 소개와 다양한 HTTP 도구들](https://mangkyu.tistory.com/291)
