---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, decisions, open-questions]
related:
  - "[[00-index]]"
  - "[[05-operational-risk-register]]"
  - "[[06-deployment-readiness-roadmap]]"
---

# 07 · 결정이 필요한 질문 목록

> 본 문서는 **사람의 결정을 요구하는 항목** 만. 단정 X — 옵션과 트레이드오프만 정리.
> 각 항목 결정 후 [[06-deployment-readiness-roadmap]] / [[05-operational-risk-register]] 갱신.

---

## Q1 · Supabase 를 완전히 제거할 것인가?

**Context:**
- 현재 Supabase 직접 호출은 concept 도메인 (P0 cutover 대상) + 영상 storage placeholder.
- BE 는 이미 `StudentConceptController` / `TeacherConceptController` 보유 → FE swap 만 남음.
- `VITE_SUPABASE_*` 가 FE 번들에 포함 — 운영 빌드에 키 노출 (R12).

**옵션:**
| 옵션 | 결과 | 트레이드오프 |
| --- | --- | --- |
| A. 완전 제거 (concept cutover 직후 `supabase.ts` 삭제 + env 제거) | 운영 표면 단일화 (BE 만) | concept 이미지/파일 일부가 Supabase storage 잔류 시 별도 마이그레이션 필요 |
| B. concept 만 cutover, storage placeholder 잔류 | 영상 cutover (S3) 까지 buffer | dist 번들에 anon key 잔류 → R12 잔존 |
| C. 영구 듀얼 (Supabase + bkurs-be 병행) | 빠른 prototyping | 두 시스템 권한 / audit 분리 — 운영 부담 |

**권고 후보:** A — concept cutover 와 S3 cutover 사이 buffer 가 길면 B 도 가능. 결정 후 `06-deployment-readiness-roadmap` P0 §2.2 갱신.

**Owner:** Product + FE + BE

---

## Q2 · `/api/*` CloudFront 라우팅 vs `api.bkurs.com` 별 도메인?

**Context:**
- FE 의 `apiClient.withCredentials=true` + HttpOnly refresh cookie.
- BE 의 `SameSite=Lax` 가 cross-site 로 분류되는 도메인은 cookie 미동봉 위험.
- `application-prod.yml` CORS `allowedOriginPatterns: ${FRONTEND_URL}` — 단일 origin.

**옵션:**
| 옵션 | 도메인 | cookie SameSite | CORS |
| --- | --- | --- | --- |
| A. same-site (`bkurs.com` + CloudFront 경로 `/api/*` → ALB) | 동일 origin | `Lax` 그대로 OK | 사실상 CORS 불필요 |
| B. 별 도메인 (`bkurs.com` FE + `api.bkurs.com` BE) | cross-subdomain | `Lax` 가 cross-site 차단 ↔ `None;Secure` 필요 | CORS 필수 + credentials |
| C. 멀티 환경 (`stg.bkurs.com`, `prod.bkurs.com`, `stg-api.bkurs.com`, ...) | 더 복잡 | 동일 | multi-origin CORS 패턴 필요 |

**권고 후보:** A — 같은 origin 이면 CSRF / cookie 정책 단순화 + R9 자연 해결. B 면 BE `same-site=None; Secure` + CSP / CORS 갱신 필요.

**Owner:** Infra + BE

---

## Q3 · MASTER 계정 bootstrap 방식?

**Context:**
- R1 mitigation 후 public signup 으로 MASTER 생성 불가.
- 그러나 prod 첫 배포 시 MASTER 1명 이상 필요.

**옵션:**
| 옵션 | 방식 | 보안 |
| --- | --- | --- |
| A. SQL seed (Flyway V`x__seed_master.sql`) — 초기 비밀번호 평문 hash 박아 prod 첫 배포 후 즉시 변경 | 단순 | seed 비번이 git 에 남음 — 초기 변경 강제 |
| B. ApplicationRunner — `MASTER_BOOTSTRAP_EMAIL`, `MASTER_BOOTSTRAP_PASSWORD` ENV 가 있고 master 가 없으면 생성, 한번 생성 후 ENV 제거 | env-driven | secret rotation 필요 |
| C. admin CLI / Spring Shell — 운영자가 BE 컨테이너에 exec 해서 한 번 실행 | 명시적 / 감사 가능 | 운영자 수동 부담 |
| D. AWS Lambda one-shot — 별 service 가 IAM 자격으로 1회 호출 | 클라우드-native | 인프라 복잡 |

**권고 후보:** B — ENV 주입 + idempotent + 1회 실행 후 ENV 제거. C 도 안전하지만 자동화 비용 ↑.

**Owner:** BE + Infra

---

## Q4 · 파일 업로드는 S3 구현 전까지 닫을 것인가?

**Context:**
- 현재 `FileStorageService` = MockFileStorage only. 실 S3 impl 없음.
- FE `useUploadFlow` 의 `simulateUpload` 가 진행률 시뮬레이션.
- 영향 화면: master/lessons, concept (이미지?), record (PDF 다운로드), saenggibu, simulation (video) 등.

**옵션:**
| 옵션 | 운영 상태 | 비고 |
| --- | --- | --- |
| A. 업로드 UI 비활성 (router 차단 또는 button disabled + banner) | 데이터 일관성 OK | 일부 화면이 read-only |
| B. mock impl 잔류 + dev/stg 만 활성 | dev 검증 가능 | prod 절대 enable 금지 (R11 가드) |
| C. S3 cutover 후 활성 | 가장 안전 | cutover 일정 의존 |

**권고 후보:** A + 동시에 B17 (S3 impl) 우선순위 ↑. cutover 완료 후 활성화.

**Owner:** Product + BE + FE

---

## Q5 · concept 도메인 모델은 FE 기준 vs BE 기준?

**Context:**
- 현재 FE concept (Supabase 직접) 가 inroad-high 호환 mock 시점 모델 사용.
- BE `concept/` 도메인이 별도 정의됨 (`Flyway V25__create_concept_diagnosis.sql`).
- FE `useConceptDiagnosis` hook 은 이미 BE 계약 사용 (`src/lib/hooks/useConceptDiagnosis.ts`).

**옵션:**
| 옵션 | 결과 |
| --- | --- |
| A. BE 모델 기준 — FE 페이지 UX 일부 조정 (필드 명 / 단계 매핑) | 백엔드 단일 SSoT |
| B. FE 모델 기준 — BE entity / DTO 일부 조정 | 마이그레이션 V`n+1` 필요 |
| C. 중간 — DTO 어댑터로 호환 | 단기 OK, 장기 부담 |

**권고 후보:** A — BE V25 가 이미 prod 모델로 정착. FE UX 조정 비용 < schema 변경 비용.

**Owner:** Product + BE + FE

---

## Q6 · secret manager — AWS SSM Parameter Store vs Vault vs k8s Secret?

**Context:**
- `.env.example` 에 모든 외부 SaaS 키 placeholder.
- `application-prod.yml` 의 `${VAR:?required}` 는 누락 시 startup fail 강제.
- 그러나 secret 의 격리 / rotation / audit 흐름 미정.

**옵션:**
| 옵션 | 적합 환경 | 비고 |
| --- | --- | --- |
| A. AWS SSM Parameter Store | ECS / EC2 + IAM | 단순, 비용 ↓, rotation 수동 |
| B. AWS Secrets Manager | 동일 | 자동 rotation, 비용 ↑ |
| C. HashiCorp Vault | cloud-agnostic | 운영 부담 ↑ |
| D. k8s Secret + KMS | k8s 선택 시 | 클러스터 종속 |

**권고 후보:** A (단순 시작) → 결제 키 같은 high-sensitivity 만 B 로 승격.

**Owner:** Infra

---

## Q7 · Container registry — GHCR vs ECR?

**Context:**
- 현재 `deploy-dev.yml` 이 GHCR 에 image push.
- AWS ECS 사용 시 ECR 이 IAM 통합 더 자연.

**옵션:**
| 옵션 | 비고 |
| --- | --- |
| A. ECR 만 사용 — deploy workflow 가 ECR push | AWS-native, IAM 통합 |
| B. GHCR 만 사용 — ECS task pull credential 수동 | 무료 (public repo), AWS 외 사용 가능 |
| C. 둘 다 (CI 가 GHCR, prod 가 ECR) | 복잡 |

**권고 후보:** A — prod 단순화. dev 만 GHCR 잔류 가능.

**Owner:** Infra

---

## Q8 · Master role 운영 노출 정책 — block vs banner vs swap?

**Context:** R2. 현재 Master 9/10 페이지 mock.

**옵션:**
| 옵션 | 운영 결과 |
| --- | --- |
| A. 전체 차단 — `/master/*` router 가 prod 빌드에서 비활성 | 마스터가 prod 에서 아무것도 못 봄 |
| B. banner read-only — "베타 — mock 데이터" 표시 후 노출 | 사용자 혼란 risk |
| C. 페이지별 점진 swap — questions 만 OK, 나머지 banner | 가장 점진적 |

**권고 후보:** C — 이미 questions 가 BE OK. 다른 페이지는 banner + 사용 가이드.

**Owner:** Product

---

## Q9 · Spring Batch / RabbitMQ — prod 활성화 시점?

**Context:**
- 로컬 / dev profile 은 batch enabled=false / mq=in-memory.
- prod 진입 후 미납 알림 / 일일 집계 / 비동기 mail 등 정기 job 필요.

**옵션:**
| 옵션 | 비고 |
| --- | --- |
| A. prod 초기에는 비활성, 사용자 base 100+ 시점에 활성 | 단순 시작 |
| B. prod 초기부터 활성 (AmazonMQ for Rabbit + Spring Batch) | 초기 운영 부담 ↑ |
| C. cron 만 활성, MQ 는 후행 | 알림은 동기 처리 (느릴 수 있음) |

**권고 후보:** A — 운영 stress 가 낮은 초기엔 단순 cron 만.

**Owner:** Product + BE

---

## Q10 · 회귀 테스트 정책 — vitest + Playwright e2e?

**Context:** R14. FE CI 가 lint + typecheck + build 만.

**옵션:**
| 옵션 | 비고 |
| --- | --- |
| A. vitest 단위 + Playwright critical-path 5 시나리오 | 표준 |
| B. vitest 단위 만 | e2e 부재 — staging smoke 에 의존 |
| C. Playwright e2e 만 | 단위 부재 — slow |

**권고 후보:** A — 단위 (lib/api unwrap 등) + e2e (login → record).

**Owner:** FE

---

## 결정 추적 표 (요약)

| ID | 질문 | Owner | 차단 단계 |
| --- | --- | --- | --- |
| Q1 | Supabase 완전 제거? | Product+FE+BE | P0 cutover |
| Q2 | same-site vs cross-site? | Infra+BE | P1 staging |
| Q3 | MASTER bootstrap? | BE+Infra | P0/P1 |
| Q4 | 파일 업로드 닫음? | Product+BE+FE | P1 |
| Q5 | concept 모델 기준? | Product+BE+FE | P0 cutover |
| Q6 | secret manager? | Infra | P1 staging |
| Q7 | container registry? | Infra | P1 CI/CD |
| Q8 | Master 표면 정책? | Product | P0 |
| Q9 | Batch / MQ 활성 시점? | Product+BE | P2-P3 |
| Q10 | 회귀 테스트 정책? | FE | P1 staging |
