---
title: "REFACTOR — AbstractIntegrationTest 추출 (16 테스트 cleanup 통합) 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-refactor
created_at: 2026-05-20T13:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[task-log-b7b-personal-statement]]"
tags: [task-log, refactor, test-helper]
---

# 작업

GitHub issue [#35](https://github.com/seum-running/bkurs-be/issues/35) — AbstractIntegrationTest 추출.

# 배경 / 회수 누적

- V11 → V15 누적 (B5/B6A/B6B/B7A/B7B) 으로 16 테스트 cleanup 매 PR 보강 부담.
- B6B (10 보강), B7A (14 보강), B7B (15 보강) — 매번 동일 패턴 반복.
- Handover 회수 노트 누적 → "BaseAcademyTest 헬퍼 추출 강력 필요" 3 PR 연속 기록.
- B7C 도입 전 본 refactor 우선 진행 결정.

# 진행

## 2026-05-20

### 1. 설계
- abstract base class (vs @TestComponent helper) — 단일 상속 OK, parent @BeforeEach 자동 실행.
- JUnit 5 / Spring Boot 의 @SpringBootTest / @ActiveProfiles 상속 검증.

### 2. 코드 (commit 단위)
- C1: AbstractIntegrationTest 작성 (133 lines)
- C2: 16 테스트 migration — extends + 중복 제거

### 3. 결과
- diff: 16 files, +122 / -1377 lines = **1270 줄 감소**
- 회귀 152/152 GREEN 유지
- 신규 entity 시 base class 한 곳만 보강

### 4. PR
- PR [#36](https://github.com/seum-running/bkurs-be/pull/36) (feature/issue-35 → feature/issue-33-b7b-personal-statement, stacked)

# 인수인계 (Handover)

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| abstract base class (vs @TestComponent / @ExtendWith) | 단일 상속 OK, JUnit parent @BeforeEach 자동 실행, @SpringBootTest 상속 표준. | TestExecutionListener 도 가능했지만 boilerplate 더 큼. abstract 가 가장 단순. |
| @BeforeEach cleanupAllRepositories — parent 자동 실행 | JUnit 5 명시: parent → child 순서. child 의 fixture setup 전 깨끗한 DB 보장. | manual @BeforeEach setup 호출 패턴은 누락 위험. annotation 이 정답. |
| protected 필드 (vs private + getter) | child 가 동일 패키지 아니라도 protected 접근 가능 + 코드 단순. | private + 다중 abstract getter 는 boilerplate. |
| 22 repo @Autowired 전체 포함 | 일부 테스트만 필요한 repo 도 모두 — child 가 선택 사용. 추가 비용 < 사용성. | 도메인 별 abstract (AcademyTestBase / CourseTestBase 등) 도 가능. 단일 base 가 단순. |
| FK 의존 역순 cleanup 한 메서드 | 신규 entity 추가 시 한 줄만 보강 (correct order). | 16 곳 보강 vs 1 곳 보강 = 16x reduction. |
| 헬퍼 (bearer/pw/saveAccount) | 16 곳 동일 helper 중복 → 단일화. saveAccount 가 tx.execute 보장. | 향후 saveOwner / saveStudent 같은 도메인 specific 헬퍼도 base 에 추가 검토. |
| AuthApi / RefreshTokenApi 의 @BeforeEach 메서드 삭제 | 본인 fixture 없이 cleanup 만 — parent 가 처리하므로 child 메서드 불필요. | empty @BeforeEach 유지보다 삭제가 깔끔. |
| StudentProfileApi 의 profileRepository → studentProfileRepository rename | parent 의 표준 이름 (studentProfileRepository) 통일 — alias 혼선 회피. | local alias 유지하면 grep 시 두 이름 동시 검색 부담. 표준화 정답. |
| RefreshTokenApiTest 의 in-test repo.deleteAll() 유지 | parent cleanup 은 @BeforeEach (테스트 시작 시), in-test 는 assertion 사이 isolation. 의도 다름. | 두 패턴 공존 명확. |

# 다음 단계 (예정)

- B7C 예상질문/기출 — 신규 entity 추가 시 AbstractIntegrationTest 만 보강
- B7D 모의면접 / B7E 생기부 / B7F 토론·수행평가·독서·시뮬

# 회수 / pitfall

- (refactor PR 자체가 누적 회수의 종결)

# 참고

- [[task-log-b7b-personal-statement]]
- JUnit 5 parent/child @BeforeEach 순서 표준
- Spring Boot @SpringBootTest 상속 표준
