---

# JPQL 벌크 연산(@Modifying 사용) 시 데이터 불일치 문제 및 해결방법

---

## 1. `@Query` 어노테이션

- Spring Data JPA에서는 기본적으로 JpaRepository를 통해서 제공되는 findById와 같은 메서드도 있고, 메서드 네이밍만을 통해서 쿼리를 실행할 수 있도록 기능을 제공해주고 있다.
- 하지만, 이 두가지 방법으로도 만들 수 없는 쿼리가 필요하다면, 쿼리를 직접 작성해야 한다.
- 그 때 커스텀 Reopository의 메서드에 붙이는 annotation이 `@Query`이다.
- 기본적으로는 JPQL로 작성할 수 있고, `nativeQuery=true` 옵션으로 네이티브 쿼리도 사용 가능하다.

## 2. `@Modifying` 어노테이션과 문제점
- `@Query` Annotation으로 작성 된 변경, 삭제 쿼리 메서드를 사용할 때 필요하다.
- 즉, 조회 쿼리를 제외하고, 데이터에 변경이 일어나는 INSERT, UPDATE, DELETE, DDL 에서 사용하며, 주로 벌크 연산 시에 사용한다.
- **JPA Entity LifeCycle을 무시하고 쿼리가 실행되기 때문에 데이터 불일치 문제가 발생하여, `clearAutomatically`와 `flushAutomatically` 속성을 적절히 활용해야 한다.**
### 벌크 연산의 동작 방식
- JPQL UPDATE/DELETE 쿼리는 업데이트된 엔티티가 영속성 컨텍스트에 이미 존재할 경우, 캐시된 데이터는 갱신되지 않는다.

**예시**:  
```java
@Modifying
@Query("UPDATE Member m SET m.name = 'newName' WHERE m.id = :id")
void updateNameById(@Param("id") Long id);
```
- DB에서는 값이 변경되지만, 영속성 컨텍스트에 캐시된 엔티티가 있으면 캐시된 값이 반환된다.

## 3. **해결 방법 1: `clearAutomatically = true`**  
### 3.1.1 동작 원리  
```java
@Modifying(clearAutomatically = true)
@Query("UPDATE Member m SET m.name = 'newName' WHERE m.id = :id")
void updateNameById(@Param("id") Long id);
```
- 벌크 연산 실행 후 **영속성 컨텍스트를 강제 초기화**하여 캐시 삭제 
- 이후 조회 시 DB에서 최신 데이터를 가져온다.

### 3.1.2 주의사항  
- **모든 영속성 컨텍스트 데이터 삭제** → `flush()` 미호출 시 변경 사항 손실 가능성이 있다.
- 해결책: 트랜잭션 내에서 명시적 `flush()` 호출하면 된다.
  ```java
  @Transactional
  public void updateMember(Long id) {
      memberRepository.updateNameById(id); // clearAutomatically=true
      entityManager.flush(); // 변경 사항 보존
  }
  ```

---

## 3. **해결 방법 2: `flushAutomatically`와 Hibernate FlushMode**  
### 3.2.1 `flushAutomatically`의 역할  
- **쿼리 실행 전 영속성 컨텍스트 변경 사항을 DB에 반영할지 결정** (기본값: `false`).  
- **Hibernate의 `FlushModeType.AUTO`** 가 기본값이라 `flushAutomatically=false`여도 쿼리 전 flush 발생한다.  

### 3.2.2 `FlushModeType` 조정  
- **`FlushModeType.COMMIT`로 변경 시**:  
  ```properties
  # application.properties
  spring.jpa.properties.hibernate.flushMode=COMMIT
  ```
  - 쿼리 실행 전 자동 flush 비활성화 → `flushAutomatically=true`가 필요하다.  
  - 예시:  
    ```java
    @Modifying(flushAutomatically = true)
    @Query("DELETE FROM Article a WHERE a.isPublished = true")
    int deletePublished();
    ```

## 4. **통합 전략 비교**  
| 전략                   | 동작 시점                | 사용 시나리오                  | 주의사항                     |
|------------------------|-------------------------|-------------------------------|-----------------------------|
| `clearAutomatically=true` | 벌크 연산 직후         | 단일 트랜잭션 내 후속 조회 필요 | 영속성 컨텍스트 전체 초기화 |
| `flushAutomatically=true` | 벌크 연산 전          | Hibernate `FlushMode=COMMIT` 환경 | 불필요한 flush 방지         |
| `FlushMode=COMMIT`      | 트랜잭션 커밋 시        | 대량 작업 성능 최적화          | 수동 flush 관리 필요        |


## 5. 결론  
- **`clearAutomatically=true`** 는 **영속성 컨텍스트 갱신 문제**를 해결하는 기본 방안이다.  
- **`flushAutomatically`** 는 Hibernate의 `FlushMode` 설정에 따라 동작이 달라지므로,`FlushMode=COMMIT` 환경에서만 유의미하다.
- **대량 작업 시**:  
  - `clearAutomatically=true` + `FlushMode=COMMIT` + 배치 크기 설정(`batch_size`)으로 성능 극대화할 수 있다. 

## References
- [Spring Data JPA @Modifying (1) - clearAutomatically](https://devhyogeon.tistory.com/4)
- [Spring Data JPA @Modifying (2) - flushAutomatically](https://devhyogeon.tistory.com/5)
- [Sring Data JPA @Modifying 알아보기](https://joojimin.tistory.com/71)
