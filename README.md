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


second/
├── 🔧 Common Modules (공통 라이브러리)
│   ├── common-core/           # 핵심 공통 기능
│   ├── common-database/       # DB 관련 공통 설정
│   └── common-log/           # 로깅 공통 설정
│
├── 🌐 Backend Services (Spring Boot)
│   ├── server-cloud/         # API Gateway (게이트웨이) :20200
│   ├── server-batch/         # 배치 스케줄러 :20290
│   ├── server-member/        # 회원 서비스 :20260
│   ├── server-file/          # 파일 서비스 :20280
│   ├── server-api/           # API 서비스
│   ├── service-cocoin/       # 코코인 비즈니스 서비스 :20190
│   └── service-batch/        # 배치 워커 서비스 :20190
│
├── 🖥️ Frontend Applications (Next.js)
│   ├── front-cocoin/         # 코코인 웹 애플리케이션
│   └── front-subuk/          # 서북 웹 애플리케이션
│
└── 🔌 Real-time Services (Node.js)
└── server-socket/        # WebSocket 서버 :20270


## 🏛️ 아키텍처 패턴
### **1. 마이크로서비스 아키텍처**
- 각 서비스가 독립적으로 배포 가능
- API Gateway를 통한 중앙집중식 라우팅
- 서비스 간 HTTP/REST 통신

### **2. 배치 처리 시스템**
``` 
server-batch (스케줄러) → server-cloud (게이트웨이) → service-batch (워커)
     :20290                    :20200                   :20190
```
## 🛠️ 기술 스택
### **Backend (Spring Boot)**
- **Java 17** + **Spring Boot 3.2.4**
- **Jakarta EE** (jakarta imports)
- **Spring Data JPA** + **Lombok**
- **MySQL 8.0.32** + **Redis 7.0.8**
- **JWT 인증** + **Circuit Breaker**

### **Frontend (Next.js)**
- **Next.js 14.2.16** + **React 18.3.1**
- **TypeScript** + **Tailwind CSS**
- **Redux Toolkit** + **Redux Saga**
- **React Query** + **Axios**
- **Socket.io** (실시간 통신)

### **Infrastructure**
- **Docker 24.0.6** + **Docker Compose**
- **Ubuntu 22.04.3 LTS**
- **Node.js 20.12.2**

## 🔄 서비스 간 통신 흐름
### **1. API 요청 흐름**
``` 
Frontend → server-cloud (Gateway) → 각 Backend Service
```
### **2. 배치 처리 흐름**
``` 
@Scheduled → server-batch → server-cloud → service-batch
```
### **3. 실시간 통신**
``` 
Frontend ←→ server-socket (WebSocket)
```
## 📊 주요 기능 도메인
### **코코인 (Cocoin) 서비스**
- 암호화폐 관련 서비스
- 코인 데이터 수집 및 처리
- 실시간 가격 정보 제공

### **배치 처리 시스템**
- 정기적인 데이터 처리
- 코인 데이터 수집/삭제/전송
- Circuit Breaker 패턴 적용

### **파일 관리 시스템**
- 파일 업로드/다운로드
- 이미지 처리 및 관리

## 🎯 핵심 설계 원칙
1. **관심사의 분리**: 각 서비스가 특정 도메인에 집중
2. **확장성**: 독립적인 스케일링 가능
3. **장애 격리**: Circuit Breaker를 통한 장애 전파 방지
4. **모니터링**: 각 서비스별 헬스체크 및 메트릭스
5. **보안**: JWT 기반 인증/인가 시스템
