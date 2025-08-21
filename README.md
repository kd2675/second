# Getting Started

### Reference Documentation

For further reference, please consider the following sections:

* [Official Gradle documentation](https://docs.gradle.org)
* [Spring Boot Gradle Plugin Reference Guide](https://docs.spring.io/spring-boot/docs/3.2.4/gradle-plugin/reference/html/)
* [Create an OCI image](https://docs.spring.io/spring-boot/docs/3.2.4/gradle-plugin/reference/html/#build-image)

### Additional Links

These additional references should also help you:

* [Gradle Build Scans – insights for your project's build](https://scans.gradle.com#gradle)
* 

👨‍🏫 프로젝트 소개


⏲️ 개발 기간

🧑‍🤝‍🧑 개발자 소개
- 김도영(김해리)

💻 개발환경
java 17
springboot 3.2.4
node 20.12.2
ubuntu 22.04.3 lts
docker 24.0.6 build ed223bc
docker compose 2.21.0
mysql 8.0.32 x86_64 linux 1.2.13
redis 7.0.8 linux 6.8.0-generic x86_64
nextjs 14.2.3

⚙️ 기술 스택
jpa
jwt
mysql
redis

typescript
tailwindcss
axios
react-query
redux toolkit
redux-saga
sockjs-client

📝 프로젝트 아키텍쳐
second 프로젝트 하위 셋팅

-common
common-core
common-database
common-log

-front
front-cocoin
front-subuk

-backend(spring)
server-batch
server-cloud
server-file
server-member

service-batch
service-cocoin

-backend(node)
server-socket

📌 주요 기능

✒️ API

------------설명-----------------
┌─────────────┐    HTTP/REST    ┌─────────────┐    HTTP/REST    ┌─────────────┐
│             │ ──────────────► │             │ ──────────────► │             │
│ server-batch│                 │server-cloud │                 │service-batch│
│ (스케줄러)   │ ◄────────────── │(게이트웨이)   │ ◄──────────────│  (워커)      │
│   :20290    │    Response     │   :20200    │    Response     │   :20190    │
└─────────────┘                 └─────────────┘                 └─────────────┘


--목표--

1. server-batch 스케줄러 실행
   └─ @Scheduled 메서드 트리거

2. server-batch → server-cloud
   └─ POST /api/orchestration/execute
   └─ BatchExecutionRequest 전송

3. server-cloud 처리
   ├─ 요청 검증
   ├─ service-batch 인스턴스 선택 (로드밸런싱)
   ├─ 요청 큐에 저장 (Redis)
   └─ service-batch로 요청 위임

4. server-cloud → service-batch
   └─ POST /api/batch/{jobType}
   └─ Circuit Breaker, Retry 적용

5. service-batch 처리
   ├─ 실제 비즈니스 로직 실행
   ├─ 메트릭스 수집
   └─ BatchResult 반환

6. service-batch → server-cloud
   └─ BatchResult 응답

7. server-cloud → server-batch
   └─ BatchExecutionResponse 응답

8. 상태 추적 및 모니터링
   ├─ Redis에 작업 상태 업데이트
   ├─ 메트릭스 수집 (Micrometer)
   └─ 로그 기록


- **장애 격리**: Circuit Breaker, Fallback 처리
- **로드 밸런싱**: 여러 service-batch 인스턴스 간 작업 분산
- **상태 관리**: Redis 기반 작업 상태 추적
- **모니터링**: 메트릭스 수집 및 헬스체크
- **확장성**: 각 서버의 독립적 스케일링
