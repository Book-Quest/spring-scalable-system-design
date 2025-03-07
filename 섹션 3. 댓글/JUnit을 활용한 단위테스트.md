# JUnit이란?

Java에서 독립된 단위테스트를 지원해주는 프레임워크

- 특정 소스코드의 모듈이 의도한대로 작동하는지 검증
- 함수나 메소드에 대한 테스트 가능
- 어노테이션 기반 테스트 지원

```java
import static org.mockito.Mockito.mock;
```

## Mockito란?

- 개발자가 동작을 직접 제어할 수 있는 가짜(Mock) 객체를 지원하는 테스트 프레임워크
- Spring에 존재하는 객체간의 의존성을 단위 테스트를 위해서 가짜 객체를 주입시켜서 해결해주는 역할을 수행

### 어노테이션 예시

```java
@ExtendWith(MockitoExtension.class) // JUnit과 Mockito 연동을 위해 사용하는 어노테이션, 확장을 선언적으로 등록할 때 사용
class CommentServiceTest {
  @InjectMocks // @Monk로 생성된 가짜 객체를 자동으로 주입시켜주는 어노테이션
  CommentService commentService;
  @Mock // Mock 객체를 만들어 반환해주는 어노테이션
  CommentRepository commentRepository;
```

### BDD란?

- Behavior Driven Development의 약자로 **행동**을 기준으로 하는 개발 방법론
- 즉, 비즈니스 요구사항에 집중하여 테스트 케이스를 개발
- TDD(Test Driven Development)는 **테스트**를 기준으로 하는 개발 방법론, 테스트 자체에 집중

Mockito는 BDD 스타일로 테스트 코드를 짤 수 있게 BDD 패턴을 지원

```java
import static org.mockito.BDDMockito.given;
```

**Given-When-Then 패턴**을 주로 사용함

- Given(준비) : 테스트를 위한 초기 상태를 설정
    - Mock 객체 생성 및 기대값 정의
    - 예제 데이터를 준비
- When(실행): 테스트 대상 코드를 실행
- Then(검증): 실행 결과가 예상한 대로 나오는지 검증

```java
@Test // 해당 메소드가 테스트 메소드임을 나타냄
@DisplayName("삭제할 댓글이 자식이 있으면, 삭제 표시만 한다.") // 테스트 메소드 이름 설정(기본값 -> 메소드 이름))
void deleteShouldMarkDeletedIfHasChildren() {
    // ✅ Given (준비)  
    Long articleId = 1L;
    Long commentId = 2L;
    Comment comment = createComment(articleId, commentId);
    
    // commentRepository가 commentId에 해당하는 댓글을 반환하도록 설정
    given(commentRepository.findById(commentId))
        .willReturn(Optional.of(comment));
    
    // 자식 댓글이 2개 존재한다고 설정
    given(commentRepository.countBy(articleId, commentId, 2L)).willReturn(2L);

    // ✅ When (실행)  
    commentService.delete(commentId);

    // ✅ Then (검증)  
    verify(comment).delete(); // 실제로 delete() 메서드가 호출되었는지 확인
}

```

- given(): 특정 메서드가 호출될 때 원하는 값을 반환하도록 미리 설정가능
- verify() : 특정 메서드가 호출되었는지 검증

```java
// commentRepository.delete(comment)가 정확히 1번 호출되었는지 확인
verify(commentRepository, times(1)).delete(comment);

// commentRepository.delete(comment)가 아예 호출되지 않았는지 확인
verify(commentRepository, never()).delete(comment);

```

그렇다면 기존의 테스트 방식을 사용한 CommentApiTest 코드와 이번 CommentServiceTest의 차이는 뭘까?

## 단위 테스트 vs 통합 테스트

### **`CommentServiceTest` vs `CommentApiTest`의 차이**

|  | **CommentServiceTest** | **CommentApiTest** |
| --- | --- | --- |
| **테스트 유형** | **단위 테스트 (Unit Test)** | **통합 테스트 (Integration Test)** |
| **목적** | - `CommentService`의 메서드가 올바르게 동작하는지 검증 | - 실제 API 엔드포인트(`/v1/comments`)가 정상적으로 동작하는지 확인 |
| **테스트 대상** | - `CommentService` 클래스 | - `CommentController` (API 계층) |
| **Mock 사용 여부** | ✅ 사용 (`@Mock`, `@InjectMocks`) → `CommentRepository` 등을 가짜 객체로 만듦 | ❌ 사용 안 함 → **실제 API 호출** |
| **실제 데이터베이스 사용 여부** | ❌ 사용 안 함 (Mocking) | ✅ **로컬 서버(`localhost:9001`)에 직접 요청** |
| **속도** | 🚀 빠름 (비즈니스 로직만 테스트) | 🐢 느림 (API 요청 → DB 조회/저장 → 응답) |
| **실행 방식** | `JUnit`과 `Mockito`로 개별 메서드 실행 | 실제 `RestClient`를 사용하여 API 호출 |

### **`CommentServiceTest` (단위 테스트)**

**특징**

- `CommentService` 내부 로직만 검증
- `CommentRepository`를 **Mocking**하여 실제 DB를 사용하지 않음
- 메서드의 입력과 출력이 기대한 대로 동작하는지 확인

**장점**

✅ 빠름 (DB를 거치지 않으므로 속도가 빠름)

✅ 특정 메서드의 로직을 정밀하게 테스트 가능

✅ 유지보수가 용이함

**단점**

❌ API, DB와의 실제 연결 여부는 검증 불가

❌ 실제 환경에서 예상치 못한 문제가 발생할 수 있음

---

### **`CommentApiTest` (통합 테스트)**

**특징**

- `RestClient`를 이용해 **실제 API를 호출**
- `http://localhost:9001/v1/comments` 같은 엔드포인트를 직접 테스트
- 컨트롤러 → 서비스 → 리포지토리 → DB까지 모두 연결된 상태에서 검증

**장점**

✅ API가 제대로 동작하는지 전체적으로 확인 가능

✅ 실제 서버와 DB를 연결하여 테스트하므로, 운영 환경과 유사한 방식으로 검증 가능

**단점**

❌ 상대적으로 느림 (네트워크 및 DB IO 발생)

❌ 환경 설정이 필요함 (`localhost:9001`에서 API를 실행해야 함)

❌ 특정 테스트가 실패하면 원인 파악이 어려울 수 있음

### **언제 어떤 테스트를 사용해야 할까?**

| 상황 | 추천 테스트 |
| --- | --- |
| 특정 메서드의 로직이 올바르게 동작하는지 확인 | **단위 테스트 (`CommentServiceTest`)** |
| API가 제대로 동작하는지 확인 (요청 & 응답) | **통합 테스트 (`CommentApiTest`)** |
| 데이터베이스와의 연동을 확인하고 싶을 때 | **통합 테스트 (`CommentApiTest`)** |
| 빠르게 개발하면서 기능을 검증하고 싶을 때 | **단위 테스트 (`CommentServiceTest`)** |

---

### 출처

[JUnit 과 Mockito 기반의 단위 테스트 코드 작성](https://velog.io/@yyong3519/Mockito)

[[JUnit] 란 무엇일까?](https://velog.io/@choidongkuen/Junit-%EC%9D%B4%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%BC%EA%B9%8C-e0w6tlvp)

[[Java] Mockito 사용법 (5) - BDD 스타일 API (+ BDD란?)](https://effortguy.tistory.com/145)

[[Spring] JUnit과 Mockito 기반의 Spring 단위 테스트 코드 작성법 (3/3) 출처: https://mangkyu.tistory.com/145 [MangKyu's Diary:티스토리]](https://mangkyu.tistory.com/145)
