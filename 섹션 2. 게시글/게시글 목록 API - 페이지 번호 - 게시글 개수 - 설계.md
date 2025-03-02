# 페이징 성능 향상 - Count 쿼리 성능 개선

페이징 기능에서 Count 쿼리는 성능 병목의 주범이 될 수 있다. 특히 데이터가 많아질수록 Count 쿼리 최적화는 필수적이다.

## 1. 문제 분석: 왜 Count 쿼리가 느릴까?

*   **전체 데이터 스캔:** Count 쿼리는 특정 조건에 맞는 전체 데이터 개수를 파악하기 위해 테이블 전체를 스캔해야 할 수 있다.
*   **복잡한 조인:** 여러 테이블을 조인해야 하는 경우 Count 쿼리는 더욱 복잡해지고, 인덱스를 효율적으로 활용하기 어려워진다.
*   **불필요한 전체 Count:** 게시글 목록 페이징 시 실제로 필요한 것은 전체 게시글 수가 아닐 수 있다. 이동 가능한 페이지 번호 활성화에 필요한 일부 데이터만으로도 충분할 때가 많다.

## 2. Count 쿼리 최적화 전략

### 2.1. Covering Index 활용

*   Count 쿼리에 필요한 모든 컬럼을 포함하는 Covering Index를 생성한다.
*   테이블에 직접 접근하지 않고 인덱스만 사용하여 Count 쿼리를 처리하므로 I/O 비용을 줄이고 성능을 향상시킬 수 있다.

### 2.2. 부분 Count: 필요한 만큼만 카운트하여 Subquery에서 LIMIT 활용

#### 2.2.1. 필요한 만큼만 카운트
*   이동 가능한 페이지 번호 활성화에 필요한 만큼만 게시글 수를 카운트하는 방법다.
*   사용자가 현재 보고 있는 페이지에 따라 다음 페이지 그룹 활성화 여부를 판단할 수 있는 최소한의 게시글 수만 확인한다.
*   **공식:** `(((n - 1) / k) + 1) * m * k + 1`
    *   `n`: 현재 페이지 번호
    *   `m`: 페이지 당 게시글 개수
    *   `k`: 이동 가능한 페이지 개수
*   **예시:**
    *   현재 페이지 7, 페이지 당 30개 게시글, 10개 페이지씩 이동 가능: `(((7-1)/10)+1)*30*10+1 = 301` 301개 게시글 유무만 확인
    *   현재 페이지 12, 페이지 당 30개 게시글, 10개 페이지씩 이동 가능: `(((12-1)/10)+1)*30*10+1 = 601` 601개 게시글 유무만 확인

#### 2.2.2. Subquery에서 LIMIT 활용
*   Count 쿼리에서 `LIMIT`은 무의미하므로, Subquery에서 Covering Index와 `LIMIT`을 함께 사용하여 부분 Count를 구현할 수 있다.
*   **예시:**

    ```sql
        SELECT COUNT(*)
        FROM (
            SELECT article_id -- 다음 페이지에서 필요한 article_id
            FROM article
            WHERE board_id = {board_id}
            ORDER BY article_id -- MySQL과 달리 PostgreSQL은 서브쿼리에서 무작위로 데이터를 선택할 수 있으므로, 정렬 기준을 명시하면 인덱스를 활용한 최적화가 가능하다.
            LIMIT {limit} -- 위 공식으로부터 도출된 개수만큼 제한
        ) AS t;
    ```
*  위 쿼리에서는 `board_id`와 `article_id`를 포함하는 Covering Index가 필요하다.


### 2.3. Slice 기반 페이징: Total Count 생략

*   `org.springframework.data.domain`에 포함된 Interface로 대표적인 구현체로는 `PageImpl`, `SliceImpl`이 있다.
*   총 레코드 개수를 반환할 필요가 없다면 (이상적인 커서 페이징), 다음 페이지 존재 여부만 확인하고 전체 Count 쿼리를 생략하는 `SliceImpl`을 활용할 수 있다.

    ```java
    // [기존] PageImpl 사용
      @Transactional(readOnly = true)
        fun search(param: Search): Mono<Page<Worklist>> {
            val pageNumber = param.page ?: 0
            val pageSize = param.limit ?: 10
            val sortBy = param.sortBy?.let(::property) ?: "create_at"
            val sort = (param.asc?.let { if(it) Sort.Order.asc(sortBy) else Sort.Order.desc(sortBy) } ?: Sort.Order.desc(sortBy)).let { Sort.by(it) }
            val pageable = PageRequest.of(pageNumber, pageSize, sort)
            return template.search(SqlIdentifier.unquoted("mrd.worklist"), param.filters, WorklistEntity::class.java, pageable).mapNotNull { page ->
                if(page.content.isEmpty()) null
                else PageImpl(page.content.map(::map), page.pageable, page.totalElements)
            }
        }
    ```
    
    ```java
    // [수정] SliceImpl 사용
     @Transactional(readOnly = true)
        fun search(param: Search): Mono<Slice<Worklist>> {
            val pageNumber = param.page ?: 0
            val pageSize = param.limit ?: 10
            val sortBy = param.sortBy?.let(::property) ?: "create_at"
            val sort = (param.asc?.let { if(it) Sort.Order.asc(sortBy) else Sort.Order.desc(sortBy) } ?: Sort.Order.desc(sortBy)).let { Sort.by(it) }
            val pageable = PageRequest.of(pageNumber, pageSize, sort)
            return template.search(SqlIdentifier.unquoted("mrd.worklist"), param.filters, WorklistEntity::class.java, pageable).mapNotNull { page ->
                if(page.content.isEmpty()) null
                else SliceImpl(page.content.map(::map), page.pageable, page.hasNext())
            }
        }
    ```

### 2.4. Count와 Content 로직 분리 및 병렬 처리

*   `PageImpl`의 생성자에는 `content`와 `count`가 필수 인자로 요구되며, Content 조회와 Count 쿼리를 분리하여 병렬로 처리하면 전체 응답 시간을 단축할 수 있다.

### 2.5. 캐싱 활용

*   게시글 수가 자주 변경되지 않는 경우, Count 결과를 캐싱하여 재사용할 수는 있으나, 동시성 문제에도 데이터 정합성이 깨지지 않는 방법을 고안해야 한다.



## 3. 페이징 전략 선택 가이드

| 요구사항                                  | 최적의 페이징 전략 |
| :---------------------------------------- | :----------------- |
| 빠른 Count 쿼리 필요                          | Covering Index 활용  |
| 이동 가능한 페이지 범위만 필요한 경우             | 부분 Count         |
| 전체 게시글 수 불필요                         | Slice 기반 페이징  |
| Content와 Count 모두 사용 필요           | 로직 분리 및 병렬 처리    |
| 실시간 Count 불필요            | 캐싱 활용          |


## References
- [JPA 페이징 Performance 향상 방법](https://cheese10yun.github.io/page-performance/)
- [[Spring Boot] 커서 페이징(no offset)에서 Page 대신 Slice 사용하기](https://zorbathegeek.tistory.com/48#%E2%AD%90%20Slice%20%EB%B0%A9%EC%8B%9D%20%EA%B5%AC%ED%98%84%20%3A%20SliceExecutionUtils%20%EB%A7%8C%EB%93%A4%EC%96%B4%EB%B3%B4%EA%B8%B0-1)
- [Speeding up partial COUNT in Postgres by using LIMIT in a subquery](https://pganalyze.com/blog/5mins-postgres-limited-count)
