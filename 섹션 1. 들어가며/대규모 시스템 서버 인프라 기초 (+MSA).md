
<img width="540" alt="image" src="https://github.com/user-attachments/assets/449e7596-dbd6-4d74-b82e-bcafb3237251" />

Client → Server

클라이언트 → 서버 : request

서버: 받은 request를 작업

Web Application Server가 그 처리를 함

그리고 그것들 중 하나가 바로 Spring boot

Spring Boot가 하는 일: 요청을 처리하며 상태 관리

Database가 하는 일: 데이터 베이스에 데이터를 안전하게 관리

 

⇒ 그럼 만약 클라이언트의 요청이 많아진다면? 단일 서버, 애플리케이션으로는 감당 불가

**Scale-Up:** 단일 서버의 성능을 향상시키는 것(수직 확장)

**Scale-Out:** 서버를 추가하여 성능을 향상시키는 것(수평 확장)

<img width="581" alt="image (1)" src="https://github.com/user-attachments/assets/1c230421-fa23-40a8-88c8-6ac17653be27" />

But, Scale-Up은 한계가 있어(비용 등) Scale-Out을 사용

<img width="637" alt="image (2)" src="https://github.com/user-attachments/assets/ec193a47-cc50-4040-b329-fcb5f2318c1c" />

클라이언트로부터 발생하는 트래픽을 라우팅 및 분산하기 위한 도구로 로드밸런서를 활용

<img width="782" alt="image (3)" src="https://github.com/user-attachments/assets/ff612b1a-f563-439c-9752-0ac7bf4ff02d" />

Cache: 더 느린 저장소에서 더 빠른 저장소에 데이터를 저장해두고 접근하는 기술, Scale-out도 가능

캐시의 활용법

- 브라우저 자체 캐시
- DNS 쿼리 결과 캐시
- DB보다 빠르게 데이터에 접근하기 위해 캐시 활용
- CDN기술에 활용

But, 안정성 문제가 있음

- 네트워크 문제
- 서버, 데이터의 시스템 오류나 자연재해 등으로의 중단 및 유실
- 물리적 거리

⇒ 응답 지연 발생

<img width="784" alt="image (4)" src="https://github.com/user-attachments/assets/60c275ef-5b0d-496d-9195-6e640f32c082" />

해결방법: 데이터센터 다중화

But, 단일 애플리케이션이 커지면 관리의 어려움이 생김

ex) 리소스 부족, 유지보수의 어려움 등

<img width="770" alt="image (5)" src="https://github.com/user-attachments/assets/8b0ce410-542e-4a2a-aada-7c5939a0ffe0" />

따라서, 단일 애플리케이션을 기능에 맞게 개별작업만 처리할 수 있도록 분리해서 관리 가능

ex) A: 게시글, B: 댓글

시스템 아키텍처: Monolithic Architecture VS Microservice Architecture

## Monolithic Architecture

- 애플리케이션의 모든 기능이 하나로 통합된 아키텍처
- 소규모 시스템에서 개발 및 배포가 간단하기 때문에 선택
- 특정 부분만 확장하기 어렵고, 변경 사항이 시스템 전체에 영향을 미침
- 대규모 시스템에서 복잡도가 커지고 개발이 어려워짐

![image (6)](https://github.com/user-attachments/assets/d3e8d51c-657f-45ae-b231-237ae0692b1e)

장점

- 게시판의 모든 기능포함
- 구조가 간단해 초기에 쉽고 빠르게 개발 가능

**case 1)**

게시글의 조회수와 작성 트래픽이 급증. 어떻게 대응할 수 있을까?

방법:

- Scale-Up ⇒ 서버의 기능은 향상되나 물리적, 비용적 하계가 있음
- Scale-Out ⇒ 서버 여러 대에 부하를 분산해 성능을 높임, but 리소스 낭비가 됨. 단일 애플리케이션이기 때문에 게시글 이외의 다른 기능도 같이 배포되어 리소스 점유

**case 2)**

여러 클래스에서 사용하고 있는 공통 코드를 수정하는 경우

![image (7)](https://github.com/user-attachments/assets/72d9984e-d446-447b-8c68-510380cf69bf)

문제 상황:

- 서비스 중 인기글 선정 정책의 변화가 필요해 인기글 코드를 수정하다 공통 코드도 같이 수정
- 인기글 기능에 문제가 없어서 재배포
- 그런데 갑자기 건들지 않았던 게시글, 댓글 기능도 동작하지 않음
- 게시글, 댓글 기능도 함께 공통 코드를 사용했기 때문에 호환이 안됨
- 최악의 경우에는 모든 서비스가 마비될 수도 있음

파생된 문제

1. 이처럼 단일 애플리케이션을 사용할 경우 공통 코드가 아니더라도, 연간된 기능에 대해 서로 코드가 침범해서 코드의 결합도 UP, 응집도 DOWN ⇒ 유지보수가 어려워짐
2. 인기글 기능만 변경하려고 해도 단일 애플리케이션이라 모든 기능이 다시 배포 ⇒ 애플리케이션이 무거울수록 빌드 및 배포 시간이 늘어남

## Microservice Architecture

시스템이 기능별로 독립적인 서비스로 구성

장점

- 각각의 마이크로서비스는 독립적으로 배포가능
- 서비스 단위로 유연한 확장 가능

단점

- 서비스 간 복잡한 통신 및 모니터링 필요
- 데이터 일관성 및 트랜잭션 관리의 어려움
- 어려운 개발 난이도

MSA로 구성된 서비스

![image (8)](https://github.com/user-attachments/assets/cd4e9e76-541a-4afb-9e38-cded4f5cc433)

- 각 기능 마이크로서비스로 배포되어 서버도 독립적으로 구성
- 각 서비스는 유기적으로 연결되어서 통신, 하나의 시스템을 이룸
- 독립적으로 관리 및 배포되므로, 복잡도가 낮고, 빠르게 빌드 및 배포할 수 있음

**case)**

게시글 트래픽에 대한 대응은?

- 게시글의 서버만 추가로 증설
    - 다른 서비스들과 독립적으로 배포되어 게시글의 리소스만 추가 점유
- 인기글에 대한 코드 변경 or 장애가 발생하더라도 다른 코드는 물리적으로 분리되어 있기 때문에 전파 범위가 줄어듬
- But, 서비스간 통신 비용, 트랜잭션 관리, 서비스 분리 기존, 모니터링, 개발 비용, 테스트, 설계 등 고려해야 할 부분이 많음

## Monolithic Architecture VS Microservice Architecture 비교

![image (9)](https://github.com/user-attachments/assets/2b56bba1-a02b-45c2-8cd7-4068295809b8)

