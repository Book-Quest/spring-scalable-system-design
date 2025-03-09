# 댓글 최대 2 depth 삭제 (DELETE)

## 1) Soft Delete VS Hard Delete

### (1) Soft Delete

데이터베이스에서 데이터를 삭제하지 않고, 사용자 입장에서는 데이터에 접근할 수 없게 하는 방식

- 테이블에 `deleted` 컬럼을 만들어 boolean값으로 데이터 사용 여부를 결정하는 방식
    - `deleted`가 false이면 조회 가능, `deleted`가 true이면 조회 불가능
- 데이터가 삭제된 것처럼 해당 데이터에 접근 불가능하지만, DB에는 데이터가 여전히 존재

#### **[장점]**

- **데이터 복구 쉬움** : 실수로 삭제해도 UPDATE를 사용하여 복구할 수 있음
- **참조 무결성 유지** : 다른 테이블에서 참조 중인 데이터를 유지하면서 삭제 가능
- **삭제 로그 관리 용이** : 언제, 누가 삭제했는지 삭제 이력 관리 가능

#### **[한계]**

- 실제로 삭제하지 않으므로 데이터베이스가 커질 수 있음
- `deleted = false` 조건을 넣지 않으면 삭제된 데이터도 조회될 수 있음
- **성능 저하 가능성** : 특히 대량의 데이터가 논리적으로 삭제된 경우, 해당 데이터를 매번 건너뛰어야 하므로 성능에 영향을 줄 수 있음

**Hibernate 예시:**

```java
@Where(clause = "deleted = false")
@Entity
@Table(name = "comment")
public class Comment {
    // ...
}
```

- 이 방식으로 쿼리마다 `deleted=false` 조건을 자동으로 추가해주어 실수로 삭제된 데이터를 조회하는 일을 방지할 수 있음
- @Where 어노테이션에 명시했던 조건이 쿼리에 반영되는 경우
    - ✅ 엔티티 전부를 조회 (`select * from comment;` )
    - ❌ 특정 필드만 조회 (`select comment_id, content from comment;` )

### (2) Hard Delete

데이터베이스에서 데이터를 직접 삭제하는 방식

- DELETE 쿼리를 날리는 방식

#### **[장점]**

- **공간 절약** : 데이터가 데이터베이스에서 완전히 제거되어 저장 공간을 절약할 수 있음
- `deleted=false` 조건을 추가할 필요 없이 데이터 조회가 가능

#### **[한계]**

- **복구 불가능**: 삭제된 데이터를 되돌릴 수 없으므로 실수로 삭제된 데이터를 복구할 수 없음.
- **참조 무결성 문제**: 다른 테이블에서 참조 중이면 삭제가 어렵거나, 삭제할 때 추가적인 처리(Foreign Key CASCADE 설정 등)가 필요함.

### (3) Soft Delete VS Hard Delete

|  | Soft Delete | Hard Delete |
| --- | --- | --- |
| **설정 용이성** <br> Ease of Setup | ✅ 단순히 열을 UPDATE하는방식으로 구현이 더 쉬움  | 🟥 삭제할 데이터를 **Audit table**로 복사하는 작업이 포함되어 설정이 어려움 |
| **디버깅** | ✅ `deleted`과 **Audit table**를 통해 디버깅 가능 | ✅ **Audit table**를 통해 디버깅 가능 |
| **데이터 복원** <br> Restoring data| ✅ `deleted=false` 로 삭제된 데이터를 복원하기 쉬움 | 🟥 복원이 어려움 |
| **active data 쿼리** <br>Querying for active data | 🟥 where 조건에 `deleted=false` 조건을 추가하는 것을 잊었을 때 문제가 발생할 가능성 있음 (`@where` 으로 해결 가능) | ✅ 삭제된 데이터를 조회할 가능성 없음 |
| **뷰 단순성** <br>View Simplicity | 🟥 active data와 삭제된 데이터가 한 테이블에 존재 | ✅ 삭제된 데이터는 audit table에만 있고 active data는 나머지 테이블에 존재 (분리) |
| **작업 성능**<br>Performance of operation | ✅ UPDATE는 DELETE보다 빠름(microsecond) | 🟥  DELETE는 UPDATE보다 느림 |
| **애플리케이션 속도**<br>Application Performance Speed | 🟥 모든 SELECT 쿼리에 `deleted=false` 가 추가되어 더 느림 | ✅ 조건이 더 적으므로 쿼리 속도가 더 빠름 |
| **애플리케이션 크기**<br>Application Performance Size| 🟥 삭제된 데이터와 active 데이터가 한 테이블에 존재하여 테이블 크기가 커짐 | ✅ active data만 테이블에 존재하여 공간 절약 |
| **DB UNIQUE Index** | 🟥 **UNIQUE index 사용 불가능** ex) unique index : (컬럼a, 컬럼b, deleted) | ✅ 데이터가 아예 삭제되므로 새로운 데이터를 삽입할 때 UNIQUE index가 충돌하지 않음 |
| **DB CASCADING** | 🟥 ON DELETE CASCADE 사용 불가. 대신 삭제 flag를 추적하는 UPDATE 트리거 생성 | ✅ ON DELETE CASCADE 사용 |

#### **[Audit table]**

관리 권한이 있는 경우에만 접근할 수 있는 테이블로, 특정 테이블에서 수행되는 작업을 추적하는 기능을 함

- 일반적으로 시간정보(생성일자, 수정일자, 삭제일자)와 사용자 정보(생성자, 수정자)를 저장
- audit table의 레코드는 업데이트하거나 삭제할 수 없으며, 삽입만 가능.
    - 레코드를 업데이트 및 삭제하려면 추가 승인과 같은 프로세스가 가능

예시 : 

```sql
CREATE TABLE comment_audit (
    audit_id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- 감사 로그의 고유 식별자
    table_name VARCHAR(255) NOT NULL,      --  변경이 발생한 테이블명
    record_id INT NOT NULL,                -- 변경된 레코드의 고유 식별자
    action_type VARCHAR(10),               -- 변경 유형 (INSERT, UPDATE, DELETE)
    old_values TEXT,                       -- 변경 전 레코드 값 (JSON 또는 XML 형식)
    new_values TEXT,                       -- 변경 후 레코드 값 (JSON 또흔 XML 형식)
    created_by VARCHAR(255),               -- 변경을 수행한 사용자
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- 변경된 시간
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP  -- 변경이력 수정 시각
);

```

#### **[Unique Index 문제]**

- **Unique Index :** 중복 데이터를 막기 위해 특정 컬럼 조합에 대해 유일한 값만 저장하도록 강제하는 기능
- 예시 ) Unique Index가 `(컬럼A, 컬럼B, deleted)`
    1. **soft delete** : `(A1, B1, false)` → `(A1, B1, true)`
    2. (A1,B1)가 테이블에 추가됨 : `(A1,B1,false)`
    3. `(A1,B1,false)`는 soft delete 될 수 없음. 

### (4) Soft Delete와 Hard Delete는 언제 사용할까?

**[Soft Delete]**

- 데이터 복구의 필요성이 높을 때
- 참조 무결성 문제를 피하고 싶을 때
- 삭제되더라도 데이터가 유지되어야 하는 경우

**[Hard Delete]**

- 데이터베이스의 저장 공간을 효율적으로 활용하고 싶을 때
- 성능 최적화가 필요한 경우
- 특정 상황에서만 필요하고 남겨도 되지 않는 데이터

## 2) **2-depth 댓글의 재귀적 삭제 로직**

### DELETE 케이스

- 삭제할 댓글이 자식 있으면, 삭제 표시만 한다
- 하위 댓글이 삭제되고, 삭제되지 않은 부모면, 하위 댓글만 삭제한다.
- 하위 댓글이 삭제되고, 삭제된 부모면, 재귀적으로 모두 삭제한다.

### 댓글 삭제 로직

```java
@Transactional
public void delete(Long commentId) {
    commentRepository.findById(commentId)           // 댓글 id로 댓글 찾기
            .filter(not(Comment::getDeleted))       // 아직 삭제되지 않은 댓글인지 확인
            .ifPresent(comment -> {                 // 댓글이 있으면 삭제
                if (hasChildren(comment)) {         // 하위 댓글이 있다면 삭제 표시만
                    comment.delete();
                } else {                            // 하위 댓글이 없다면 삭제
                    delete(comment);
                }
            });
}
```

1. comment_id로 댓글 조회
2. 만약 삭제되지 않은 상태라면, 자식 댓글이 있는지 확인한다.
    
    ```java
    private boolean hasChildren(Comment comment) {
        // commentId를 parentCommentId로 가지는 comment들의 수를 세기
        return commentRepository.countBy(comment.getArticleId(), comment.getCommentId(), 2L) == 2;
    }
    ```
    
    ```sql
    select count(*) from (
       select comment_id from comment
       where article_id = :articleId and parent_comment_id = :parentCommentId
       limit :limit
    )
    ```
    
    - commentId를 parentCommentId로 가지는 comment(자식)들의 수를 세기
        - root 댓글인 경우, 자식이 1개 이상인지 확인하기 위해 limit 2로 지정
3. 댓글 삭제
    
    ```java
    private void delete(Comment comment) {
        // 삭제
        commentRepository.delete(comment);
        // 하위 댓글이 삭제가 됐으면 deleted=true, 하위 댓글이 없는 상위 댓글을 재귀적으로 삭제
        if(!comment.isRoot()) {
            commentRepository.findById(comment.getParentCommentId())        // 상위 댓글
                    .filter(Comment::getDeleted)                            // 상위댓글이 삭제가 되어있는지 확인
                    .filter(not(this::hasChildren))                         // 상위댓글이 또 다른 자식을 가지고 있지 않은지 확인
                    .ifPresent(his::delete);                               // 재귀적으로 상위 댓글 삭제
        }
    }
    ```
    
    - 자식 댓글이 있다면 `deleted=true`로 Soft Delete 처리
    - 자식 댓글이 없다면 DB에서 실제 삭제 (Hard Delete)
4. 만약 부모 댓글이 삭제된 상태이고, 부모 댓글이 다른 자식 댓글을 가지고 있지 않다면, 부모도 삭제한다.

---

**참고자료**

https://velog.io/@heypop/2023.10.22-Soft-Delete-Hard-Delete

https://dgjinsu.tistory.com/77

https://abstraction.blog/2015/06/28/soft-vs-hard-delete#recommendation

[https://velog.io/@yhlee9753/soft-delete-와-hard-delete-비교](https://velog.io/@yhlee9753/soft-delete-%EC%99%80-hard-delete-%EB%B9%84%EA%B5%90)
