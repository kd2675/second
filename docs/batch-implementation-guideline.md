# Batch Implementation Guideline

## 범위

이 문서는 일반적인 배치 프로젝트에서 배치를 구현할 때 사용하는 다음 5개 레이어만 다룬다.

- `job`
- `step`
- `reader`
- `processor`
- `writer`

목표는 하나다.

배치 프로젝트의 구조를 레이어 단위로 정리하고, 각 레이어가 자기 역할만 하도록 만들며, 그 안에서 가장 안정적이고 유지보수하기 좋은 구현 기준을 정하는 것.

현재 기준 스택은 다음과 같다.

- Java 17
- Spring Boot 3.2.4
- Spring Batch 5.x
- Spring Data JPA
- MySQL 8.x

---

## 기본 원칙

### 1. 레이어 이름보다 책임이 더 중요하다

파일이 `reader` 패키지에 있다고 해서 Reader가 되는 것이 아니다.

다음 기준을 만족해야 한다.

- `Job`은 흐름만 정의한다.
- `Step`은 조립만 한다.
- `Reader`는 읽기만 한다.
- `Processor`는 변환과 필터만 한다.
- `Writer`는 반영만 한다.

이 기준이 무너지면 배치 재시작, 중복 실행, 장애 복구가 어려워진다.

### 2. 배치는 도메인 단위로 묶는다

배치 코드는 타입별 수평 분리보다 도메인별 수직 분리가 유지보수에 훨씬 유리하다.

권장 구조:

```text
src/main/java/<base-package>/batch/
  <domain>/
    job/
    step/
    reader/
    processor/
    writer/
    support/
  <another-domain>/
    job/
    step/
    reader/
    processor/
    writer/
    support/
```

핵심은 한 도메인의 배치 흐름을 한 디렉터리 안에서 끝까지 추적할 수 있게 만드는 것이다.

### 3. 공통 추상화는 실제 공통 기능이 생길 때만 만든다

기능 없는 빈 추상 클래스는 유지하지 않는다.

예:

- `BasicReader`
- `BasicProcessor`
- `BasicWriter`

이런 타입 별칭은 추상화 비용만 만들고 설계 품질을 올리지 못한다.

현재 스택에서는 다음이 기본이다.

- `ItemReader<T>`
- `ItemProcessor<I, O>`
- `ItemWriter<T>`

### 4. Java 17 문법을 적극적으로 쓴다

구현 시 기본값으로 쓰는 문법:

- `record`
  - 검색 조건
  - cursor 상태
  - reader 입력 파라미터
  - 외부 API 파싱 후 내부 전달 모델
- `switch expression`
  - 상태 분기
- `Stream.toList()`
  - 읽기 전용 컬렉션
- text block
  - 긴 SQL / JPQL / 메시지 템플릿
- `Optional.ifPresentOrElse`
  - null 분기 축소

---

## 권장 패키지 구조

아래 구조를 기본으로 한다.

```text
batch/<domain>/
  job/
    DomainIngestJobConfig.java
    DomainSendJobConfig.java
    DomainArchiveJobConfig.java
  step/
    DomainIngestStepConfig.java
    DomainSendStepConfig.java
    DomainArchiveStepConfig.java
  reader/
    DomainSourceReader.java
    DomainSendTargetReader.java
    DomainArchiveCursorReader.java
  processor/
    DomainItemProcessor.java
    DomainSendPayloadProcessor.java
    DomainArchiveProcessor.java
  writer/
    DomainEntityWriter.java
    DomainOutboxWriter.java
    DomainArchiveWriter.java
    DomainDeleteWriter.java
  support/
    DomainQueryService.java
    DomainMessageRenderer.java
    DomainArchiveWindow.java
```

여기서 `support`는 다음 같은 코드를 넣는 곳이다.

- Reader / Processor / Writer 어디에도 직접 넣기 애매한 순수 비즈니스 로직
- SQL 조립
- 메시지 렌더링
- 계산 규칙
- 도메인 정책

---

## Job 최적 구현 방안

## Job의 역할

`Job`은 “어떤 순서로 어떤 step을 실행할지”만 정의한다.

Job 파일에서 허용되는 책임:

- step 순서 정의
- flow 분기
- listener 연결
- batch 이름 정의

Job 파일에서 금지하는 책임:

- repository 호출
- 외부 API 호출
- 데이터 변환
- 비즈니스 계산
- 메시지 조립

## Job 파일 구성 원칙

권장 파일명:

- `DomainArchiveJobConfig`
- `DomainSendJobConfig`
- `DomainCleanupJobConfig`

권장 코드 형태:

```java
@Configuration
public class DomainArchiveJobConfig {

    public static final String JOB_NAME = "domainArchiveJob";

    @Bean(name = JOB_NAME)
    public Job domainArchiveJob(
            JobRepository jobRepository,
            @Qualifier(DomainArchiveStepConfig.SELECT_STEP) Step selectStep,
            @Qualifier(DomainArchiveStepConfig.DELETE_STEP) Step deleteStep
    ) {
        return new JobBuilder(JOB_NAME, jobRepository)
                .start(selectStep)
                .next(deleteStep)
                .build();
    }
}
```

## Job 설계 기준

### Job은 use case 기준으로 나눈다

좋은 예:

- 도메인 적재 job
- 도메인 전송 job
- 도메인 아카이브 job

나쁜 예:

- 적재 + 전송 + 정리를 한 job 안에 다 넣는 것

### Job은 가능한 작게 유지한다

한 job은 한 운영 목적만 가져야 한다.

판단 기준:

- 스케줄이 다르면 job도 나눈다.
- 실패 복구 정책이 다르면 job도 나눈다.
- 트랜잭션 성격이 다르면 job도 나눈다.

---

## Step 최적 구현 방안

## Step의 역할

`Step`은 reader, processor, writer를 조립하고 실행 정책을 정한다.

Step 파일에서 허용되는 책임:

- chunk size 지정
- transaction manager 지정
- reader / processor / writer 연결
- skip / retry / listener 설정

Step 파일에서 금지하는 책임:

- 엔티티 생성
- 외부 API 호출
- 메시지 포맷팅
- repository 직접 호출

## Step 파일 구성 원칙

권장 파일명:

- `DomainArchiveStepConfig`
- `DomainSettlementStepConfig`

권장 구조:

```java
@Configuration
public class DomainArchiveStepConfig {

    public static final String SELECT_STEP = "domainArchiveSelectStep";

    @Bean(name = SELECT_STEP)
    @JobScope
    public Step domainArchiveSelectStep(
            JobRepository jobRepository,
            @Qualifier("crawlingTransactionManager") PlatformTransactionManager txManager,
            @Qualifier(DomainArchiveCursorReader.BEAN_NAME) ItemReader<SourceEntity> reader,
            @Qualifier(DomainArchiveProcessor.BEAN_NAME) ItemProcessor<SourceEntity, ArchiveEntity> processor,
            @Qualifier(DomainArchiveWriter.BEAN_NAME) ItemWriter<ArchiveEntity> writer
    ) {
        return new StepBuilder(SELECT_STEP, jobRepository)
                .<SourceEntity, ArchiveEntity>chunk(500, txManager)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }
}
```

## Step 설계 기준

### chunk step과 tasklet step을 구분한다

chunk step을 쓰는 경우:

- 여러 건 읽고 가공하고 저장
- cursor 기반 반복 처리
- skip / retry가 중요

tasklet step을 쓰는 경우:

- 단건 제어 작업
- bulk SQL 한 번 실행
- 파일 이동
- 잠금 획득 / 해제

### chunk size는 도메인별로 상수화한다

권장:

- `NEWS_ARCHIVE_CHUNK_SIZE`
- `HOTDEAL_SEND_CHUNK_SIZE`

피해야 할 것:

- 모든 step에 같은 숫자 복붙

### 실패 정책은 주석이 아니라 설정으로 남긴다

나쁜 예:

```java
// .faultTolerant()
// .skip(...)
```

좋은 예:

```java
.faultTolerant()
.retry(HttpServerErrorException.class)
.retryLimit(3)
.skip(IllegalArgumentException.class)
.skipLimit(10)
```

정책이 없다면 주석도 남기지 않는다.

---

## Reader 최적 구현 방안

## Reader의 역할

Reader는 데이터를 읽기만 한다.

허용되는 책임:

- DB 조회
- 외부 API 조회
- 파일 읽기
- cursor 계산
- 파라미터 해석

금지되는 책임:

- Mattermost 전송
- DB 저장
- 상태 갱신
- 삭제
- 비즈니스 결과 확정

Reader는 최대한 side effect가 없어야 한다.

## Reader 구현 방식

### 1. 데이터가 작을 때만 `ListItemReader`

적합한 경우:

- 최대 수십~수백 건
- 한 번에 메모리에 올려도 부담이 없음

부적합한 경우:

- 누적 데이터
- 삭제 대상
- 1만 건 이상 가능성
- 페이지 이동이 필요한 경우

### 2. 대량 데이터는 cursor 또는 안정 paging

현재 스택에서 가장 권장하는 방식:

- `JdbcClient` 또는 `JdbcTemplate` 기반 keyset reader
- `id > :lastId order by id asc limit :size`

권장 예:

```sql
select id, title, content, pub_date
from old_news
where id > :lastId
  and pub_date >= :from
order by id asc
limit :size
```

### 3. `JpaPagingItemReader`를 쓸 때 규칙

- 반드시 안정 정렬 사용
- 고유 key 포함
- 내부 page 상태를 해킹하지 않음

좋은 예:

```sql
select e
from SourceEntity e
where e.pubDate < :threshold
order by e.id asc
```

나쁜 예:

- 정렬 없음
- 계산식만 있는 정렬
- `getPage()` 오버라이드로 0 고정

### 4. 외부 API Reader는 파싱과 읽기를 분리

권장 구조:

- Reader: 호출과 page 순회
- Support: 응답 파싱
- Processor: DTO -> Entity 변환

즉 Reader에서 메시지 알림이나 repository 저장까지 하지 않는다.

## Reader 파일 구성 예시

```java
@Configuration
public class DomainArchiveCursorReader {

    public static final String BEAN_NAME = "domainArchiveCursorReader";

    @Bean(name = BEAN_NAME)
    @StepScope
    public ItemReader<SourceEntity> reader(
            DomainQueryService queryService
    ) {
        return new ItemReader<>() {
            private long lastId = 0L;
            private Iterator<SourceEntity> current = List.<SourceEntity>of().iterator();

            @Override
            public SourceEntity read() {
                if (current.hasNext()) {
                    SourceEntity item = current.next();
                    lastId = item.getId();
                    return item;
                }

                List<SourceEntity> next = queryService.findArchiveTargets(lastId, 500);
                if (next.isEmpty()) {
                    return null;
                }

                current = next.iterator();
                SourceEntity item = current.next();
                lastId = item.getId();
                return item;
            }
        };
    }
}
```

---

## Processor 최적 구현 방안

## Processor의 역할

Processor는 읽은 값을 가공하고, 다음 레이어가 쓰기 좋은 형태로 만든다.

허용되는 책임:

- DTO -> Entity 변환
- 필드 정규화
- 파생 값 계산
- 조건 불충족 시 `null` 반환으로 필터링

금지되는 책임:

- repository save
- delete
- 외부 메시지 전송
- 트랜잭션 경계 제어

## Processor 구현 기준

### 1. Processor는 최대한 순수 함수처럼 작성

입력 하나가 들어오면 결과 하나가 나오는 형태가 가장 좋다.

좋은 예:

- `SourceItem -> DomainEntity`
- `DomainEntity -> DomainSendPayload`
- `InputEntity -> ExecutableEntity`

### 2. 도메인 계산은 support/service로 분리

Processor 안에 분기와 계산이 길어지면 support로 뺀다.

권장 구조:

- `DomainExecutableProcessor`
- `DomainSettlementPolicy`

Processor는 orchestration만 하고 계산식은 policy/service가 담당한다.

### 3. snapshot이 필요한 값은 Step 시작 전에 명시적으로 주입

예를 들어 주문 정산 기준 가격처럼 “이 step 동안 동일해야 하는 값”은 Processor 내부에서 임의로 다시 읽지 않는다.

권장 방식:

- StepScope bean 생성 시점에 snapshot 값을 생성
- support record에 담아 주입

예:

```java
public record SettlementSnapshot(int basePrice, LocalDateTime baseTime) {
}
```

### 4. 단순 매핑은 람다 또는 메서드 참조 사용 가능

불필요한 익명 클래스 남발은 줄인다.

예:

```java
ItemProcessor<SourceEntity, Long> processor = SourceEntity::getId;
```

단, 도메인 이름이 중요하면 별도 named bean이 더 낫다.

---

## Writer 최적 구현 방안

## Writer의 역할

Writer는 실제 side effect를 수행한다.

허용되는 책임:

- insert
- update
- delete
- 외부 시스템 전송

Writer는 side effect를 갖는 대신, 한 writer가 한 종류의 결과만 책임져야 한다.

## Writer 설계 기준

### 1. DB writer와 외부 전송 writer를 분리한다

좋은 구조:

- `DomainEntityWriter`
- `DomainOutboxWriter`
- `ExternalMessageWriter`

나쁜 구조:

- Mattermost 전송 후 DB 상태 갱신까지 한 writer가 같이 수행

외부 전송은 롤백되지 않으므로 DB 갱신과 함께 묶으면 중복 발송 위험이 생긴다.

### 2. CompositeItemWriter는 같은 성격의 작업만 묶는다

허용:

- archive insert + archive audit insert
- local table update + local table update

주의:

- 외부 전송 + DB 상태 갱신
- delete + 네트워크 호출

이 조합은 되도록 step을 나누는 쪽이 낫다.

### 3. 대량 삭제는 row-by-row보다 bulk를 우선 검토

나쁜 예:

```java
for (Long id : chunk) {
    repository.deleteById(id);
}
```

좋은 예:

- `deleteAllByIdInBatch`
- native bulk delete
- tasklet 한 번으로 delete SQL 실행

### 4. 아카이브는 insert와 delete를 분리한다

권장 순서:

1. archive insert
2. 검증 가능 상태 저장
3. source delete

한 writer에서 “복사 후 바로 삭제”를 모두 처리하는 것보다 두 단계가 더 안전하다.

### 5. 메시지 전송은 outbox가 최선

현재 구조에서 가장 안전한 구현은 다음이다.

1. 전송 대상 payload를 outbox 테이블에 저장
2. 별도 step이 outbox pending을 읽어 외부 전송
3. 성공 시 outbox status를 sent로 변경

작은 시스템이라도 이 구조를 기본 목표로 본다.

---

## 레이어별 최적 파일 구성

아래는 `<source_table> -> <archive_table>` 아카이브 배치의 최적 구조 예시다.

```text
batch/<domain>/
  job/
    DomainArchiveJobConfig.java
  step/
    DomainArchiveInsertStepConfig.java
    DomainArchiveDeleteStepConfig.java
  reader/
    DomainArchiveCursorReader.java
    ArchivedDomainIdReader.java
  processor/
    DomainArchiveProcessor.java
  writer/
    ArchiveEntityWriter.java
    SourceEntityDeleteWriter.java
  support/
    DomainArchiveWindow.java
    DomainArchiveQueryService.java
```

### 레이어 흐름

#### insert step

- Reader: 아카이브 대상 source entity를 cursor로 읽음
- Processor: `SourceEntity -> ArchiveEntity`
- Writer: archive insert

#### delete step

- Reader: 아카이브 완료된 source id 읽음
- Writer: source delete

이 구조가 좋은 이유:

- Reader는 읽기만 함
- Processor는 변환만 함
- Writer는 한 종류의 side effect만 수행
- 재시작 시 어느 단계에서 실패했는지 명확함

---

## 구현 시 상세 규칙

### Job

- 한 job은 한 운영 목적만 가진다.
- Bean 이름은 의미 있게 짓는다.
- step 이름도 목적이 드러나야 한다.

### Step

- chunk size는 상수화한다.
- `@JobScope`는 step bean에만 사용한다.
- retry / skip은 주석이 아니라 설정으로 남긴다.

### Reader

- side effect 금지
- 안정 정렬 필수
- 대량 데이터는 keyset 우선
- `ListItemReader`는 소량 데이터만

### Processor

- 순수 함수처럼 작성
- 길어진 계산은 support로 이동
- `null` 필터링은 의도 명확히

### Writer

- 한 writer는 한 종류의 결과만 반영
- 외부 I/O와 DB 갱신 분리
- 대량 delete/update는 batch/bulk 검토

---

## 현재 구조를 이 기준으로 바꿀 때 우선순위

### 1순위

- Reader의 side effect 제거
- `DelJpaPagingItemReader` 제거
- 안정 정렬 없는 paging 제거

### 2순위

- 도메인 중심 패키지로 재배치
- `BasicReader` / `BasicProcessor` / `BasicWriter` 제거
- 외부 전송 writer와 DB writer 분리

### 3순위

- outbox 패턴 도입
- bulk delete / cursor reader 도입
- support 패키지에 정책 / 쿼리 / 렌더러 분리

---

## 최종 기준

배치 프로젝트에서 가장 좋은 구현은 다음 조건을 만족하는 구조다.

- Job은 흐름만 가진다.
- Step은 조립만 가진다.
- Reader는 읽기만 한다.
- Processor는 변환과 필터만 한다.
- Writer는 한 종류의 반영만 한다.
- 외부 전송과 DB 상태 갱신은 분리한다.
- 대량 삭제와 이관은 paging 해킹 대신 cursor 또는 bulk로 처리한다.
- 도메인별로 파일을 모아 탐색성과 변경 용이성을 높인다.

이 문서의 핵심은 단순하다.

`job / step / reader / processor / writer` 구조를 유지하되,
각 레이어가 자기 역할만 하게 만드는 것이 배치 프로젝트에서의 최적 구현 방식이다.
