## 인덱스가 필요한 이유 및 실제 Query Plan
인덱스는 다음 과정을 통해 레코드를 읽는다.

1. 인덱스를 통해 PK를 찾는다.
2. PK를 통해 레코드를 찾는다.

- 인덱스를 통해 레코드 1건을 읽는 것이 4-5배 정도 비싸기 때문에, 읽어야 할 레코드의 건수가 전체 테이블 레코드의 20-25%를 넘어서면 인덱스를 이용하지 않는 것이 효율적이다.
- 이런 경우 옵티마이저가 인덱스를 조회하고 데이터에 접근하는 것보다 바로 데이터를 조회하는 비용이 더 낫다고 판단하면, Index Scan 대신 Sequential Scan을 사용하기도 한다.
- PostgreSQL의 경우 `random_page_cost`와 같은 파라미터를 조정하여 인덱스 스캔과 순차 스캔 간의 전환점을 조정할 수 있다.
- 기계적 디스크 저장소에 대한 랜덤 액세스는 일반적으로 순차 액세스보다 4배 이상 비싸다. 그러나 인덱싱된 읽기 같이 디스크에 대한 랜덤 액세스 대부분은 캐시에서 일어나므로 작은 기본값(4.0)으로 설정된다.

## 테이블 스캔 방식
PostgreSQL은 5가지 스캔 방식(Sequential Scan, Index Scan, Index Only Scan, Bitmap Scan, TID Scan)을 사용한다.

### 1. Sequential Scan
- 테이블의 모든 데이터를 하나씩 확인하는 방법으로, 주로 인덱스가 없는 column을 조건으로 검색할 경우에 사용된다.
- Parallel Seq Scan으로 여러 워커 프로세스가 동시에 테이블을 스캔 가능하다.

### 2. Index Scan
- 인덱스를 사용하여 필요한 데이터를 찾은 후, 해당 데이터의 실제 위치로 접근하는 방식이다.
- 인덱스는 항상 정렬되어있기 때문에 역순으로 조회가 가능하다. 따라서 Index Scan Backward 방식을 이용해서 id를 역순으로 조회하여 쿼리를 효율적으로 처리하는 것을 확인할 수 있다.

### 3. Index Only Scan
- Index Only Scan은 인덱스에 필요한 데이터가 있는 경우 사용되는 방식으로, 실제 테이블 데이터에 접근하지 않는다.
- 필요한 값이 인덱스에 모두 있는 것을 커버링 인덱스라고 하며, 커버링 인덱스를 활용하여 테이블 접근 없이 인덱스만으로 쿼리를 처리하는 방식이라고 할 수 있다.

### 4. Bitmap Scan
- Index Scan과 Sequential Scan의 중간 형태로, 두 단계로 처리된다.
  - 1단계(Bitmap Index Scan): 인덱스를 스캔하여 조건에 맞는 데이터의 위치 정보를 비트맵으로 생성
  - 2단계(Bitmap Heap Scan): 생성된 비트맵을 이용해 실제 데이터에 접근
- 다수의 행을 검색할 때 무작위 접근의 오버헤드를 줄이는 방식이다.
- 흥미로운 점은 가져와야 할 데이터가 줄어들면 데이터베이스가 Index Scan을 사용한다는 것이다. 즉, 데이터베이스가 조건문에 사용할 column의 인덱스 유무뿐만 아니라 실제로 가져와야 할 데이터의 크기도 고려해서 Table Scan 방법을 정한다는 것을 확인할 수 있다.

### 5. TID Scan
- 테이블의 물리적 위치 식별자(TID)를 사용하여 직접 데이터에 접근하는 방식으로, Tuple Indecator의 줄임말이다.
- 튜플 식별자(ctid)를 이용한 직접 접근한다.
  ```sql
  EXPLAIN ANALYZE
  SELECT id FROM post WHERE ctid = '(1, 1)';
  ```
### 테이블 스캔 방식 도식화

<img src="https://github.com/user-attachments/assets/6774a693-da9c-40f2-ab61-8f0119bac29d" width="700" height="auto">

- 옵티마이저는 특정 조건에 따라 동작하는 패턴을 위처럼 도식화할 수 있다.
- 그러나 실제로 옵티마이저가 스캔 방식을 선택할 때는 훨씬 다양한 요소들을 종합적으로 고려하기 때문에, 항상 위 패턴을 따르지는 않는다.

## 인덱스를 활용한 Sort 오퍼레이션 생략하기

- ORDER BY가 포함된 쿼리는 일반적으로 Sort 오퍼레이션이 발생하지만, 적절한 인덱스를 사용하면 이를 생략할 수 있다.

### ORDER BY Sort 오퍼레이션 생략 규칙
- ORDER BY절에 WHERE 조건절이 같이 쓰일 경우, WHERE 조건절에 포함된 컬럼이 인덱스의 선두 컬럼으로 연속되어야 한다.
- WHERE 조건절에 포함된 컬럼은 등치(=) 조건이어야 한다.

### Sort 오퍼레이션 생략 예시

예를 들어, 다음과 같은 쿼리가 있을 때,
```sql
SELECT * FROM employee ORDER BY begin_date, end_date;
```

아래와 같은 인덱스를 생성하면 Sort 오퍼레이션을 생략할 수 있다.
```sql
CREATE INDEX idx_employee_begin_end ON employee(begin_date, end_date);
```

WHERE 절이 포함된 쿼리에서도 인덱스를 활용하여 Sort 오퍼레이션을 생략할 수 있다.
```sql
SELECT * FROM employee WHERE begin_date = '20231119' ORDER BY end_date;
```

이 경우 (begin_date, end_date) 인덱스는 begin_date로 정렬된 후 end_date로 정렬되어 있기 때문에, begin_date가 '20231119'인 데이터들 사이에서도 end_date로 정렬되어 있음을 보장한다.

## Reference
- [프로젝트 삽질기1 (feat Table Scan 실행계획)](https://overcome-the-limits.tistory.com/698#%EB%93%A4%EC%96%B4%EA%B0%80%EB%A9%B0-2)
- [[PostgreSQL 공식문서] Query Plan](https://postgresql.kr/docs/current/runtime-config-query.html)
- [PostgreSQL 실행계획 분석하기 2편 (Table Scan)](https://hyunwook.dev/226)
- [인덱스를 활용한 Sort 오퍼레이션 생략하기 (With PostgreSQL)](https://hyunwook.dev/237)

