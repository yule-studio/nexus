---
title: bkurs-be — envelope 마이그레이션 진척표
kind: tracker
status: active
created_at: 2026-05-21
tags: [bkurs-be, envelope, migration, refactor]
related:
  - "[[bkurs-be]]"
  - "[[../bkurs-fe/architecture-api-foundation]]"
  - "[[architecture-async-mq-batch]]"
home_hub: "[[bkurs-be]]"
---

# bkurs-be — envelope 마이그레이션 진척표

## 1. 핵심 요약

bkurs-be 전체 endpoint (~153) 를 `ApiEnvelope<T> {success, data, error, meta}`
표준으로 마이그레이션. phase 1 (안전한 3 도메인) 완료 → phase 2~5 점진.

| Phase | 도메인 | PR | 상태 |
|-------|--------|-----|------|
| **1-A** | auth (AuthController + AuthExceptionHandler) | [#101](https://github.com/seum-running/bkurs-be/pull/101) | **DONE** |
| **1-B** | student-profile (Student + Owner) | [#103](https://github.com/seum-running/bkurs-be/pull/103) | **DONE** |
| **1-C** | dashboard (Owner + Teacher, read-only) | [#105](https://github.com/seum-running/bkurs-be/pull/105) | **DONE** |
| **C3-신규** | suhaeng (PR #109) + saenggibu (PR #111) — 신규는 envelope 표준부터 | [#109](https://github.com/seum-running/bkurs-be/pull/109) [#111](https://github.com/seum-running/bkurs-be/pull/111) | **DONE** |
| **2-1A** | reading | [#113](https://github.com/seum-running/bkurs-be/pull/113) | **DONE** |
| **2-1B** | homework | [#115](https://github.com/seum-running/bkurs-be/pull/115) | **DONE** |
| **2-1C** | course (9 controller — Owner/Teacher/Student × Course/Session/Attendance) | [#117](https://github.com/seum-running/bkurs-be/pull/117) | **DONE** |
| **2-2A** | question | [#119](https://github.com/seum-running/bkurs-be/pull/119) | **DONE** |
| **2-2B** | interview | [#121](https://github.com/seum-running/bkurs-be/pull/121) | **DONE** |
| **2-2C** | simulation | [#123](https://github.com/seum-running/bkurs-be/pull/123) | **DONE** |
| **2-3A** | record (생기부 entry) | [#125](https://github.com/seum-running/bkurs-be/pull/125) | **DONE** |
| **2-3B** | performance | [#127](https://github.com/seum-running/bkurs-be/pull/127) | **DONE** |
| **2-3C** | statement (자소서) | [#129](https://github.com/seum-running/bkurs-be/pull/129) | **DONE** |
| **2-4A** | plan / payment / subscription | [#131](https://github.com/seum-running/bkurs-be/pull/131) | **DONE** |
| **2-4B** | search | [#133](https://github.com/seum-running/bkurs-be/pull/133) | **DONE** |
| **2-4C** | announcement | [#135](https://github.com/seum-running/bkurs-be/pull/135) | **DONE** |
| **2-4D** | discussion | [#137](https://github.com/seum-running/bkurs-be/pull/137) | **DONE** |
| **final** | concept (Student + Teacher) | [#144](https://github.com/seum-running/bkurs-be/pull/144) | **DONE** |

## 마이그레이션 종료 (2026-05-21)

phase 1 + phase 2 + 잔여 누락 도메인 전체 envelope 마이그레이션 완료. 모든 도메인
controller / handler 가 ApiEnvelope 표준. 신규 endpoint 는 처음부터 envelope.

- 총 도메인 **19개** (auth, student-profile, dashboard, suhaeng, saenggibu, reading,
  homework, course, question, interview, simulation, record, performance, statement,
  plan, search, announcement, discussion, stats, academy, concept)
  · file: 이미 envelope 설계 / pdf: 의도적 raw byte stream (envelope 제외)
- 총 PR **22개** (phase 1 × 3 + Cycle 3 신규 × 2 + phase 2 × 12 + Cycle helper-fix × 1 +
  stats × 1 + academy × 1 + concept × 1 + cleanup)
- ErrorCode prefix **20개** — AUTH/STU/SUH/SGB/READ/HW/CRS/Q/IVW/SIM/REC/PE/STM/PAY/SRC/ANN/DSC/STT/ACA/CON
- 회귀 0. 통합 test 모두 PASS (1 fail = actuator RabbitMQ DOWN, envelope 무관).

### Controller 단위 전수 점검 (PR #144 후)
- 총 controller 67개 → **65개 envelope 적용** (97%)
- 의도적 예외 2개:
  - `StudentRecordPdfController` — PDF byte stream (`Content-Type: application/pdf`)
  - `HealthController` — `/api/v1/health` 단순 ping (gateway/CDN 컨벤션상 raw)
- 모든 `*ExceptionHandler` envelope 사용
- 모든 controller 메서드 시그니처: `ResponseEntity<ApiEnvelope<...>>` 또는 raw stream / void

## 사용자 지적 (2026-05-21)
1차 phase 2 종료 후 stats / academy 누락 — PR #139 (test helper fix) + #141 (stats) + #143
(academy) 로 1차 보정. 2차 점검에서 **concept 누락** 발견 (FE concept.ts 는 envelope.unwrap
호출했지만 BE 가 raw 반환 → 호출 broken 상태) → PR #144 로 최종 보정.

> 교훈: "phase 2 끝" 시점에 "전수 controller grep" 점검을 한 번 더 했어야 함. 점검 표 (3.4)
> 추가.

(아래 옛 phase 2-4/2-5 행은 위 세부 행으로 대체됨)
| **2-4** | plan / payment / search / announcement / discussion | — | (위로 분할) |
| **2-5** | concept / suhaeng / saenggibu (Cycle 3 신규 도메인 포함) | — | (Cycle 3 / 신규는 처음부터 envelope) |

## 2. 내 해석 (왜 이 순서로)

### phase 1 = "안전한 3 도메인" 선정 기준
- **auth**: 모든 도메인이 의존 → 표준 잡으면 나머지 패턴 복제 쉬움.
- **student-profile**: 도메인 exception 3종 + happy/error 모두 있음 — handler+
  controller 양쪽 envelope 검증.
- **dashboard**: read-only + handler 없음 — controller envelope 만 패턴 검증.

이 3 조합으로 envelope 마이그레이션의 **3 가지 변형 (handler/controller/test)**
패턴을 모두 검증 → phase 2 부터는 같은 template 복제.

### phase 2~5 묶음 기준
도메인 응집도 (학습 / 평가 / 기록 / 지원 / 신규) 로 ~10 controller 씩 묶어
각 PR 4-5 commit. envelope 마이그레이션 자체는 mechanical — 한 묶음 ~1 cycle.

## 3. 적용 맥락

### 3.1. 도메인별 작업 template

각 도메인 envelope 마이그레이션은 항상 같은 3 commit:

**Commit 1** — `<Domain>ErrorCode` enum + Handler envelope
- `src/main/java/com/bkurs/<domain>/enums/<Domain>ErrorCode.java` (신규)
  - code prefix: 도메인 약자 (AUTH, STU, ACA, PAY, ...)
  - status 별 4001/4002... + 4040 (NotFound) + 4091 (Conflict) 패턴
- `<Domain>ExceptionHandler.java` 갱신
  - `ApiEnvelopes` 주입
  - 각 exception → `envelopes.fail(code.status(), ApiError.of(code, msg, null, requestId), request)`

**Commit 2** — Controller envelope
- 응답 type: `ResponseEntity<ApiEnvelope<T>>`
- `ApiEnvelopes` 주입 + `HttpServletRequest http` 파라미터 추가
- 본문: `return envelopes.ok(data, http);`
- POST + cookie 같은 hybrid 응답은 envelope.body 만 추출해서 새 ResponseEntity 빌드
- logout 등 204 No Content 는 envelope 없이 그대로

**Commit 3** — Test envelope assertion 갱신
- helper 함수 추가 (envelope.data 추출):
  ```java
  private Map<String, Object> data(ResponseEntity<Map> res) {
      return (Map<String, Object>) res.getBody().get("data");
  }
  private Map<String, Object> error(ResponseEntity<Map> res) {
      return (Map<String, Object>) res.getBody().get("error");
  }
  ```
- 또는 도메인 단일 endpoint 면 helper 안에서 envelope unwrap (dashboard 패턴)
- error 응답은 `error.get("code")` 까지 검증 (AUTH-4001 등)

### 3.2. 테스트 환경 변수

`./gradlew test` 단독은 실패 — DB_PASSWORD env 필요:
```bash
DB_PASSWORD='<로컬 mysql 비번>' ./gradlew test --tests "com.bkurs.<domain>.*"
```

`application-local.yml` 의 `password: ${DB_PASSWORD:?DB_PASSWORD env required ...}`
plac eholder 가 환경변수 없을 때 fail-fast.

### 3.4. 전수 점검 명령 (한 도메인 끝낼 때마다)
```bash
# 1) envelope import 없는 controller (handler + 의도적 raw 제외)
grep -rln "@PostMapping\|@GetMapping\|@PatchMapping\|@DeleteMapping\|@PutMapping" \
  src/main/java/com/bkurs --include="*Controller.java" \
  | xargs grep -L "ApiEnvelope"

# 2) ResponseEntity<Map> 반환하는 handler (envelope 우회)
grep -rln "ResponseEntity<Map\b" src/main/java/com/bkurs/

# 3) 컨트롤러 public 메서드 중 ResponseEntity 가 아닌 반환형
grep -rn "public [A-Za-z].* [a-z][A-Za-z]*(" src/main/java/com/bkurs \
  --include="*Controller.java" \
  | grep -v "ResponseEntity\|public void\|public class\|public enum\|public static\|public final"
```
허용되는 결과: `HealthController.health()` (raw Map), `StudentRecordPdfController`
(byte stream). 그 외는 모두 envelope 으로 마이그레이션.

### 3.3. 회귀 방지

- **OpenAPI 스펙 변화**: envelope 적용 시 OpenAPI doc 의 response schema 가 `T` →
  `ApiEnvelope<T>` 로 변함. FE 가 `apiClient` (envelope unwrap helper) 통해 호출하면
  영향 X — 단, raw fetch 가정한 코드 grep 으로 사전 점검.
- **client 호환**: bkurs-fe 의 `src/lib/api/concept.ts` 가 표준 (envelope unwrap) — 신규
  도메인 client 도 본 패턴 따름.

## 4. 관련 노트

- [[bkurs-be]] — 프로젝트 허브
- [[../bkurs-fe/architecture-api-foundation]] — FE 쪽 envelope unwrap 구조 (S1/S2)
- [[architecture-async-mq-batch]] — async / MQ / batch 인프라
- [[../yule-studio-agent/research-tech-lead-runtime-loop-issue-73|tech-lead loop research]]

## 5. 참고

- 코드 SSoT: `bkurs-be/src/main/java/com/bkurs/common/api/{ApiEnvelope,ApiEnvelopes,ApiMeta}.java`
- 도메인 ErrorCode 예: `auth/enums/AuthErrorCode.java` / `student/enums/StudentErrorCode.java`
- 마이그레이션 template: 본 노트 3.1
- 후속: Cycle 3 (suhaeng/saenggibu 신규 도메인) + phase 2~5 mechanical 복제.
