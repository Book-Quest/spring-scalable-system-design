----
# MSA 환경에서의 @EntityScan, @EnableJpaRepositories 및 Gradle implements 정리
----

## 1. implements(Gradle 의존성 관리)와 @EntityScan의 차이 및 역할

| 구분             | Gradle implements / api                      | @EntityScan                                    |
|------------------|----------------------------------------------|------------------------------------------------|
| 개념             | Gradle 빌드 시스템에서 모듈 간 의존성 노출 범위 제어 | Spring Boot의 런타임 시 JPA 엔티티 스캔 위치 지정     |
| 동작 시점        | 컴파일 및 빌드 단계                           | 런타임(Spring 컨테이너 초기화 시점)                |
| 목적             | 의존 모듈 코드 클래스 노출 범위와 접근성 제어        | 지정한 패키지 경로에서 `@Entity` 클래스 검색 및 등록 |
| 기능             | 모듈 간 public 클래스 노출 여부 결정(-> open api로 관리됨), 의존성 관리            | JPA 엔티티 자동 매핑 위해 대상 클래스 탐색           |

따라서, 다른 모듈의 엔티티를 코드에서 사용하려면 반드시 Gradle 의존성에 해당 모듈을 추가해야 하며(`implementation` 혹은 `api`),  
`@EntityScan`은 엔티티 클래스를 어디서 스캔할지 런타임에 알려주는 역할임을 명확히 구분해야 한다.

## 2. @EntityScan 어노테이션

- **필요 시점**: 엔티티가 `@SpringBootApplication`이 선언된 기본 패키지 외부에 있을 때(특히 멀티 모듈 프로젝트)  
- **기본 동작**:  
  - 아무 설정 없으면 `@SpringBootApplication` 패키지 및 하위 패키지 내의 `@Entity` 클래스만 스캔  
  - 외부 모듈 또는 별도 패키지 엔티티가 있으면 `@EntityScan(basePackages = {"패키지 경로"})` 명시 필요  
- **예시**

```java
@EntityScan("com.example.project.module.entity")
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```
- 컴파일 타임에는 아무런 동작하지 않음
- 런타임에 지정된 패키지에서 `@Entity` 클래스 동적으로 탐색 및 등록
- 만약 스캔 경로가 누락되면 런타임 JPA 매핑 오류 발생 가능

## 3. @EnableJpaRepositories 어노테이션

- **역할**: JPA 리포지토리 인터페이스(`JpaRepository`, `CrudRepository` 등)를 스캔해서 Spring Bean으로 등록  
- **Spring Boot 기본**: `@SpringBootApplication`이 이미 내부적으로 포함하고 있어 보통 별도 선언 불필요  
- **멀티 모듈/패키지 분리 시**: 리포지토리가 다른 위치에 있으면 명시적으로 `@EnableJpaRepositories(basePackages = "패키지 경로")` 설정 권장  
- **예시**

```java
@EnableJpaRepositories("com.example.project.module.repository")
@EntityScan("com.example.project.module.entity")
@Configuration
public class DataConfig {
}
```

## 4. 멀티 모듈 프로젝트에서 주의사항 및 전략

- **모듈 의존성 명확히 선언**  
  - 엔티티가 정의된 모듈을 사용하는 모듈에 `implementation project(":entity-module")` 또는 `api project(":entity-module")` 추가해야 컴파일과 런타임 모두 정상 동작  
  - `implementation`은 의존성 내부 한정 공개, `api`는 의존성 전파까지 포함  
- **스캔 대상 패키지 명확히 지정**  
  - `@EntityScan`, `@EnableJpaRepositories`, `@ComponentScan` 모두에서 별도 패키지 지정으로 누락 방지
  - 스캔 경로를 외부 설정(yaml, properties)로 관리하면 환경별 유연성 확보 가능   
- **공통 도메인 및 엔티티 모듈 분리**  
  - 여러 서비스 모듈에서 공유되는 엔티티, DTO 등을 별도 모듈로 분리하여 일관된 의존성 관리  
- **`@EntityScan`과 `implements`는 별개 개념**  
  - `implements`는 보통 Java 인터페이스 구현이나 Gradle 의존성 `implementation`을 뜻하며, `@EntityScan`은 엔티티 클래스 자동검색 설정임  
  - 둘을 혼동하지 말 것  
- **컴포넌트 및 기타 빈 스캔**  
  - 서비스, 컴포넌트가 별도 패키지라면 `@ComponentScan`으로 추가 등록 필요
