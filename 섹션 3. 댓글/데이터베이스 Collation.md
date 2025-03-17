# Collation이란?

<img width="443" alt="image" src="https://github.com/user-attachments/assets/72d6b6b5-977e-41cd-ac82-c3a5a6ce78f9" />

- 문자열을 정렬하고 비교하는 방법을 정의하는 설정
    - 대소문자 구분, 악센트 구분, 언어별 정렬 순서등을 포함
      
<img width="408" alt="image" src="https://github.com/user-attachments/assets/600a5faf-04bc-4fbe-8ef2-31c8688ec9df" />

디폴트 설정은 utf8mb4_0900_ai_ci

이게 과연 무엇을 의미하는걸까?

## Charset이란?

<img width="403" alt="image" src="https://github.com/user-attachments/assets/4c3bc255-054b-4592-9ffc-1985329dff26" />

- 문자들과 그 문자들을 코드화한 인코딩들의 조합, 즉 문자 집합
    - utf8mb4: 각 문자를 최대 4바이트, UTF-8 형식으로 지원, 이모지, 유니코드 문자 해결됨(4byte를 사용함으로써)

### 많이 사용하는 정렬 방식

- bin: 바이너리 저장 값을 그대로 정렬하기 때문에 대소문자를 구분한다
- general_ci: 대소문자를 구분하지 않으나, 간단한 정렬 규칙 때문에 속도가 빠름, 일부 언어에 따라 정확한 정렬이 제공되지 않음
- unicode_ci: 유니코드의 표준에 따라 정렬이 가능해서 다양한 언어에서 정확한 정렬이 가능. 그러다보니 general보다는 속도가 느림
- 0900_ai_ci: 최신 유니코드의 표준(9.0)을 따르는 방식으로, 유니코드 정렬을 지원하고 ai(accent-insentive)로 발음 구별 기호를 무시함

### 실제 사용 사례

기존의 테이블은 **utf8mb4_0900_ai_ci** 방식이었기 때문에 대소문자의 순서를 구분할 수 없어서 bin을 활용해 특정 컬럼만 가능하도록 테이블 생성

```sql
create table comment_v2 (
	comment_id bigint not null primary key,
	content varchar(3000) not null,
	article_id bigint not null,
	writer_id bigint not null,
	path varchar(25) character set utf8mb4 collate utf8mb4_bin not null,
	deleted bool not null,
	created_at datetime not null
);
```

<img width="396" alt="image (1)" src="https://github.com/user-attachments/assets/208b2ee5-435c-4f5c-805a-33eb492f19e5" />

<img width="595" alt="image (2)" src="https://github.com/user-attachments/assets/0cdbd8c6-bad5-4b7a-8c46-73f77e45bed7" />


무한 depth를 위해 path값을 키로 사용하기 때문에 path만 bin으로 정렬방식 설정

### bin의 사용이 필요한 경우

- 사용ID, 비밀번호, 토큰, 파일명 URL등 대소문자 구분이 필요한 경우
- 해시값 / 암호화 데이터 비교 (ex. SHA256, JWT 등)
- JSON데이터에서 키의 대소문자 구별이 필요한 경우 (ex. Key vs key)
- 정확한 문자열 매칭이 필요한 경우 (ex. Binary 비교가 필요할때)
- 정렬이 필요 없는 단순 데이터 저장 (ex. 로그 원본 데이터 저장)

출처

[Charset과 Collation에 대한 개념](https://sshkim.tistory.com/128)
