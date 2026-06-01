---
title: "bkurs-be CI 실패 — Flyway 정책 PR (#156)"
kind: troubleshooting
project: bkurs-be
status: open
created_at: 2026-06-01T14:00:00+09:00
updated_at: 2026-06-01T14:00:00+09:00
tags: [bkurs-be, ci, troubleshooting, flyway, actuator, env]
home_hub: "[[../bkurs-be]]"
related:
  - "[[../bkurs-be]]"
  - "[[../policy/flyway-migration-policy]]"
  - "[[../handover/index]]"
---

# bkurs-be CI 실패 — Flyway 정책 PR (#156)

## 1. 발생 일시
- 2026-06-01 04:43:14 ~ 04:47:39 UTC (CI run)
- 본 troubleshooting 기록: 2026-06-01 14:00 KST

## 2. 관련 PR
- bkurs-be PR **#156** — `🚚 Flyway 마이그레이션 정책 정착 — V1~V29 timestamp rename + ops/db + 검증 자동화`
- URL: https://github.com/seum-running/bkurs-be/pull/156
- head: `chore/flyway-timestamp-migration-policy`
- **base: `dev`** ✓ (목표 base 와 일치)
- 짝 vault PR: yule-studio/yule-agent-vault#26 (이미 머지 — flyway-migration-policy.md)

## 3. 실패한 CI check
- Workflow: `CI`
- Job: `build + test (Java 21)`
- Step: `Gradle build + test` (`./gradlew clean check --no-daemon`)
- CI run: https://github.com/seum-running/bkurs-be/actions/runs/26735598457/job/78788117150

본 PR 이 추가한 step 의 결과:

| Step | 결과 |
| --- | --- |
| `Check Flyway migration file names` | ✅ **통과** (`✅ Flyway 마이그레이션 파일명 검증 통과 (versioned 29개 + repeatable 무관).`) |
| Set up JDK 21 | ✅ |
| Wait for MySQL | ✅ |
| Verify mw user + bkurs-local DB | ✅ |
| **Gradle build + test** | ❌ **1 test failed** |

## 4. 실패 로그 핵심 요약

```
BkursBeApplicationTests > Actuator /actuator/health 는 UP 상태를 반환한다 FAILED
    org.opentest4j.AssertionFailedError at BkursBeApplicationTests.java:75

399 tests completed, 1 failed
> Task :test FAILED
BUILD FAILED in 3m 47s
```

테스트 본문 (`src/test/java/com/bkurs/BkursBeApplicationTests.java:67~78`):

```java
@Test
@DisplayName("Actuator /actuator/health 는 UP 상태를 반환한다")
@SuppressWarnings("unchecked")
void actuatorHealthIsUp() {
    Map<String, Object> body = rest.getForObject(
        "http://localhost:" + port + "/actuator/health",
        Map.class
    );
    assertThat(body).isNotNull();
    assertThat(body.get("status")).isEqualTo("UP");   // <-- line 75 가 여기 추정
    Map<String, Object> components = (Map<String, Object>) body.get("components");
    assertThat(components).containsKey("db");
}
```

- 실패 지점은 `assertThat(body.get("status")).isEqualTo("UP")` — actuator 가 UP 가 아닌 상태 반환.
- 보통 actuator/health 가 DOWN 으로 응답하는 사유:
  - DB indicator 가 DOWN — MySQL 연결 불안정 / connection pool 미준비.
  - readiness probe 가 통과되기 전에 테스트가 요청.
  - 통합 테스트 컨텍스트에서 actuator-db indicator 가 stable 하기 전에 호출.

## 5. 원인 분류

다음 분류 기준 중:

| 분류 | 해당 여부 |
| --- | --- |
| Flyway migration filename script 문제 | ❌ (Check Flyway 단계 통과) |
| GitHub Actions ci.yml step 문제 | ❌ (본 PR 추가 step 정상 통과) |
| Gradle / test 문제 | ◐ (test 자체는 1개만 실패. compile / 398개 test 통과) |
| MySQL service / DB env 문제 | △ (env / DB 자체는 연결 OK — actuator timing 으로 추정) |
| application-*.yml 설정 문제 | △ (actuator readiness / liveness 설정 의존) |
| **이번 PR 범위를 벗어난 기존 실패** | ✅ **해당** |

근거:
- 동일 실패가 **직전 cycle** 의 PR #154 / #155 CI 에서도 등장 (`Actuator /actuator/health 는 UP FAILED at BkursBeApplicationTests.java:75` — 동일 line, 동일 메시지).
- 본 PR 이 추가 / 변경한 파일 중 actuator / health / test / application*.yml 에 영향 주는 것 없음:
  - `scripts/check-flyway-migration-names.sh` (신설 — bash, runtime 무관)
  - `.github/workflows/ci.yml` (1 step 추가 — script 실행만)
  - `ops/db/*/README.md` (md, runtime 무관)
  - `README.md` / `docs/BRANCH_STRATEGY.md` / `docs/CHANGELOG.md` (docs)
  - `src/main/resources/db/migration/V*.sql` 29 파일 rename (SQL 본문 변경 없음, Flyway 가 새 파일명 정상 인식 — Check Flyway step 이 양성 확인)

## 6. 이번 PR 변경과 직접 관련 있는지 — **없음**

- script step 은 통과.
- 새 step 이 후속 build / test 에 환경 변수 / classpath / 파일 시스템 부수 효과 없음.
- 동일 실패가 본 PR 이전부터 dev 의 다른 PR 들에서도 재현됨 (flaky / 환경 의존 추정).

## 7. 재현 명령

로컬:
```bash
# (DB_PASSWORD / MySQL local 준비 후)
cd /Users/masterway/masterway-dev/bkurs/bkurs-be
./gradlew test --tests "com.bkurs.BkursBeApplicationTests.actuatorHealthIsUp" --no-daemon
```

CI:
```bash
# CI run 로그 재확인
gh run view 26735598457 --log-failed -R seum-running/bkurs-be | grep -A 5 actuatorHealthIsUp
```

## 8. 확인한 명령

```bash
gh pr view 156 -R seum-running/bkurs-be \
  --json baseRefName,headRefName,state,mergeable,mergeStateStatus,statusCheckRollup
# → base=dev, head=chore/flyway-timestamp-migration-policy, MERGEABLE, UNSTABLE
# → statusCheckRollup: build + test (Java 21) FAILURE

gh pr checks 156 -R seum-running/bkurs-be
# → build + test (Java 21)  fail  4m25s

gh run view 26735598457 --log-failed -R seum-running/bkurs-be | grep "Check Flyway\|FAILED\|tests completed"
# → ✅ Flyway 마이그레이션 파일명 검증 통과 (versioned 29개 + repeatable 무관).
# → BkursBeApplicationTests > Actuator /actuator/health 는 UP 상태를 반환한다 FAILED
# → 399 tests completed, 1 failed
```

## 9. 해결 방향

본 PR 범위 외 — **본 PR 내에서 추가 수정하지 않음**.

권고되는 후속 (다음 브랜치 / 별 PR):

1. **actuator readiness probe + 통합 테스트 동기화 개선**
   - `application.yml` `management.endpoint.health.probes.enabled` + `wait-for-readiness` 옵션 검토.
   - `BkursBeApplicationTests` 에서 actuator 호출 전 readiness 대기 helper 추가 또는 retry.
2. **DB indicator 의 startup latency 처리**
   - `management.endpoint.health.show-details` + `groups` 정의로 liveness / readiness 분리.
3. **flaky 여부 확정 검증**
   - 동일 PR 의 CI 재실행 (`gh run rerun`) 으로 일관 실패인지, intermittent 인지 확인.
   - intermittent 면 retry / flakiness 차단 작업 별 PR.

## 10. 이번 PR 에서 수정한 내용

- **없음.** 본 PR 의 코드 / 문서 / script / ci.yml 추가 단계는 이미 정상 동작 (`Check Flyway migration file names` step 양성 통과).
- 실패 원인이 본 PR 범위를 벗어나 있어, 본 PR 에서 추가 commit 으로 actuator/health 또는 application*.yml 수정을 시도하지 않음.

## 11. 이번 PR 범위를 벗어나 남겨둔 후속 작업

- `application-local.yml` / `application-dev.yml` / `application-prod.yml` 의 actuator probes 설정 — **수정하지 않음**.
- 환경 변수 / `DB_PASSWORD` / MySQL service container 자체 설정 — CI 에서 정상 연결됨 (`mw user + bkurs-local DB` step 통과). actuator readiness 자체 문제.
- `BkursBeApplicationTests` 의 retry / readiness 대기 로직 — 별 PR.
- 다른 PR 의 동일 실패 (PR #154, PR #155) — 동일 후속에서 함께 해결 권고.

## 12. application-local.yml 수정 여부
- **수정하지 않음.** 사용자 지시 + 본 PR 범위 외.

## 13. 다음 브랜치에서 다룰 항목

| 항목 | 우선순위 | 비고 |
| --- | --- | --- |
| `BkursBeApplicationTests.actuatorHealthIsUp` flaky 처리 | 높음 | retry / readiness 대기 / disable until fixed |
| `application-*.yml` actuator probes 정합 | 중 | 사용자 수동 처리 영역 |
| CI workflow 의 `wait-for-readiness` 단계 검토 | 중 | actuator readiness 와 별개로 startup 시간 보장 |
| PR #154 / #155 의 동일 실패 동시 해결 | 중 | 본 troubleshooting 의 후속 |

## 14. 결론

> **이번 PR 의 Flyway 정책 / 파일명 검증 변경과 직접 관련 없는 환경 / 기존 actuator 테스트 문제이므로,
> merge 판단과 별도로 다음 브랜치에서 `BkursBeApplicationTests` actuator readiness 동기화 및
> `application-local.yml` / actuator probes 정리 작업으로 분리한다.**

- 본 PR `#156` 의 변경 자체는 의도대로 동작:
  - Flyway 파일명 검증 script ✅ CI 양성 통과
  - 29 파일 rename — Flyway 가 새 네이밍 정상 인식
  - `ops/db/*` README + 정책 문서 — runtime 영향 없음
- 1 test 실패는 **이번 PR 머지 가부 결정에 환경 / actuator 문제로 분리해 평가**해야 한다.
- required check 정책상 `--merge` (non-admin) 로는 머지 차단. 다음 중 사용자 결정:
  1. `--admin --merge` 로 본 PR 머지 + 별 PR 로 actuator 테스트 수정.
  2. 먼저 actuator 테스트 수정 후 본 PR rebase / 재 CI / 정상 머지.
  3. flaky 가능성 검증을 위해 CI rerun.
