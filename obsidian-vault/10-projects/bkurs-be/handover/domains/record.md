---
title: "도메인 — record (학생부 8 카테고리 + 자동 bridge)"
kind: handover
status: stable
created_at: 2026-05-21T23:51:00+09:00
tags: [bkurs-be, handover, domain, record, bridge]
related:
  - "[[../../bkurs-be]]"
  - "[[../overview]]"
  - "[[../decisions/decision-student-record-categories]]"
  - "[[../decisions/decision-record-bridge]]"
  - "[[../decisions/decision-academic-period]]"
home_hub: "[[../../bkurs-be]]"
---

# record 도메인

> 고등 학생부 (생기부) 8 카테고리. **강사 수동 입력 + 4 도메인 자동 bridge** (performance /
> discussion / simulation / reading) 가 공존. bridge 는 same-TX `@EventListener` — 원본 grade /
> evaluate 이 롤백되면 record 도 롤백.

## 1. 책임

- 학생부 entry CRUD — 강사 수동 입력, 학생 본인 조회, 학원장 학원 내 학생 조회
- 4 도메인의 event 를 listen 해서 record entry auto-upsert
- source 도메인 DELETE 시 record 보존 (이미 생기부에 적힌 것)
- KST 학년도 기반 period 자동 결정

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `StudentRecordEntry` | id / studentAccountId / category (8) / period (year+sem string) / content / sourceType (4 + MANUAL) / sourceId (nullable for MANUAL) / authorAccountId / createdAt + uk(sourceType, sourceId) — V23 ALTER |
| `RecordCategory` | AWARD / AUTONOMY / CLUB / CAREER / SERVICE / SUBJECT / BEHAVIOR / READING (학종 표준 8종) |
| `RecordSourceType` | MANUAL / PERFORMANCE / DISCUSSION / SIMULATION / READING |

## 3. 패키지 구성

```
record/
├── controller/
│   ├── TeacherRecordController     # /api/v1/teachers/me/students/{id}/records — 수동 CRUD
│   ├── StudentRecordController     # /api/v1/students/me/records — 본인 조회
│   └── OwnerRecordController       # /api/v1/academies/me/students/{id}/records — 학원장 조회
├── service/
│   └── StudentRecordService        # ownership 2계층 (강사/학생/학원장)
├── bridge/                         # ★ 자동 bridge
│   ├── PerformanceGradedEvent.java
│   ├── DiscussionEvaluatedEvent.java
│   ├── SimulationReviewedEvent.java
│   ├── ReadingFeedbackedEvent.java
│   └── StudentRecordBridgeService.java   # 4 @EventListener
├── domain/
├── dto/{request,response}/
├── enums/                          # RecordErrorCode (REC-)
├── exception/
└── handler/
```

## 4. API endpoint

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| POST | `/api/v1/teachers/me/students/{studentId}/records` | TEACHER | 수동 record 추가 (sourceType=MANUAL) |
| GET | `/api/v1/teachers/me/students/{studentId}/records` | TEACHER | 학생 record 목록 |
| PATCH | `/api/v1/teachers/me/records/{id}` | TEACHER | 본인이 추가한 MANUAL record 만 |
| DELETE | `/api/v1/teachers/me/records/{id}` | TEACHER | MANUAL only |
| GET | `/api/v1/students/me/records` | STUDENT_HIGH | 본인 record 통합 |
| GET | `/api/v1/academies/me/students/{studentId}/records` | OWNER | 학원 내 학생 record |

> bridge 로 생성된 record (sourceType != MANUAL) 는 PATCH / DELETE 불가 — 원본 도메인 (PE / Discussion / Sim / Reading) 의 grade / feedback 변경이 source-of-truth.

## 5. Migration

| V | 내용 |
| --- | --- |
| V18 | `student_record_entry` (8 카테고리 enum + period varchar) |
| V23 | `ALTER ADD source_type / source_id + uk(source_type, source_id)` |

## 6. 자동 bridge 흐름 (B9)

source service 가 grade / evaluate / feedback 처리 후 event publish → bridge 가 listen → record
upsert.

### 매핑

| source | event | category |
| --- | --- | --- |
| `PerformanceEvaluationService.grade()` | `PerformanceGradedEvent` | SUBJECT |
| `DiscussionService.evaluate()` | `DiscussionEvaluatedEvent` | AUTONOMY |
| `SimulationService.review()` | `SimulationReviewedEvent` | CAREER |
| `ReadingService.feedback()` | `ReadingFeedbackedEvent` | READING |

### 코드

```java
@Service
@RequiredArgsConstructor
public class StudentRecordBridgeService {

    private final StudentRecordEntryRepository repo;

    @EventListener
    @Transactional(propagation = Propagation.REQUIRED)
    public void onPerformance(PerformanceGradedEvent ev) {
        upsert(ev.studentId(), PERFORMANCE, ev.submissionId(), SUBJECT, ev.content());
    }

    private void upsert(long studentId, RecordSourceType type, long sourceId,
                         RecordCategory category, String content) {
        AcademicPeriod period = AcademicPeriod.from(LocalDate.now(KST));
        repo.findBySourceTypeAndSourceId(type, sourceId)
            .ifPresentOrElse(
                e -> e.overwriteFromSource(category, period, content),
                () -> repo.save(new StudentRecordEntry(studentId, category, period.toString(),
                                                       content, type, sourceId, /*author*/ null))
            );
    }
}
```

### 같은 TX

`@EventListener` (NOT `@TransactionalEventListener(AFTER_COMMIT)`) — bridge 실패 시 원본 grade 도
롤백. 의도. 변경 시 주의.

### period 자동 (B12)

```java
public record AcademicPeriod(int year, int semester) {
    public static AcademicPeriod from(LocalDate date) {
        // 3-7월 → 그 해 1학기
        // 8-12월 → 그 해 2학기
        // 1-2월 → 전년도 2학기
    }
    public String format() { return year + "-" + semester; }
}
```

`KST today` 기준. `TimeConfig.Clock` 으로 테스트 시 고정 가능. 단위 8 + 통합.

## 7. 의존

- **의존**: auth, course (학생 권한 검증), academy, student
- **의존받음**: performance / discussion / simulation / reading (event publish 측), saenggibu (중등은 별 도메인 — 본 도메인은 고등 한정)
- **bridge listener** 가 본 도메인에서 위 4 도메인 event 받음 — 결합도는 event record 형식만 의존

## 8. saenggibu 와 차이

- `record`: 고등 (학종 8 카테고리, 자동 bridge)
- `saenggibu`: 중등 (학기 free-form, 학기 lock)

[[saenggibu]] 참조.

## 서비스 로직 (`record/service/` + `record/bridge/`)

1 service + bridge listener. **수동 entry CRUD + 4 도메인 자동 upsert** 가 공존.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `StudentRecordService` | `@Transactional` (write) / `readOnly` (조회) | 강사 수동 record CRUD (sourceType=MANUAL) + role 별 listing |
| `bridge.StudentRecordBridgeService` | `@EventListener` — caller TX | 4 event 받아서 record upsert (sourceType ≠ MANUAL) |

### `StudentRecordService` 핵심 메서드

```java
public StudentRecordEntry createByTeacher(Long teacherAccountId, Long studentAccountId, RecordCategory category, String content, AcademicPeriod period)
                                                          // 강사 본인 수업 학생인지 검증 + sourceType=MANUAL INSERT
public StudentRecordEntry updateByTeacher(Long teacherAccountId, Long recordId, String content)
                                                          // MANUAL only — bridge 생성 record 는 거부
public void deleteByTeacher(Long teacherAccountId, Long recordId)        // MANUAL only
public List<StudentRecordEntry> listByTeacherForStudent(Long teacherAccountId, Long studentAccountId, RecordCategory categoryFilter)
public List<StudentRecordEntry> listByStudent(Long studentAccountId, RecordCategory categoryFilter)
public List<StudentRecordEntry> listByOwnerForStudent(Long ownerAccountId, Long studentAccountId, RecordCategory categoryFilter)
                                                          // 학원장 권한 — 본인 학원 학생인지 검증
```

### `StudentRecordBridgeService` 4 @EventListener

```java
@EventListener  // NOT @TransactionalEventListener — same TX
public void onPerformance(PerformanceGradedEvent ev) {
    upsert(ev.studentId(), PERFORMANCE, ev.submissionId(), SUBJECT, ev.content());
}
@EventListener public void onDiscussion(DiscussionEvaluatedEvent ev)   { upsert(..., DISCUSSION, ..., AUTONOMY, ...); }
@EventListener public void onSimulation(SimulationReviewedEvent ev)    { upsert(..., SIMULATION, ..., CAREER, ...); }
@EventListener public void onReading(ReadingFeedbackedEvent ev)        { upsert(..., READING, ..., READING, ...); }

private void upsert(long studentId, RecordSourceType type, long sourceId, RecordCategory category, String content) {
    AcademicPeriod period = AcademicPeriod.from(LocalDate.now(KST));
    repo.findBySourceTypeAndSourceId(type, sourceId).ifPresentOrElse(
        e -> e.overwriteFromSource(category, period, content),
        () -> repo.save(new StudentRecordEntry(studentId, category, period.toString(), content, type, sourceId, null /*author*/)));
}
```

자세히 → [[../architecture/overview]] §3.

## cross-link

- [[performance]] / [[discussion]] / [[simulation]] / [[reading]] — bridge source
- [[saenggibu]] — 중등 대응
- [[../overview#3]] — bridge 흐름 상세
- [[../decisions/decision-student-record-categories]]
- [[../decisions/decision-record-bridge]]
- [[../decisions/decision-academic-period]]
