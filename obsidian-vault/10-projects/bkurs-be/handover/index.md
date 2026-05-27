---
title: "bkurs-be — 인수인계 문서 인덱스"
kind: handover
status: stable
created_at: 2026-05-22T00:35:00+09:00
tags: [bkurs-be, handover, index]
home_hub: "[[../bkurs-be]]"
related:
  - "[[../bkurs-be]]"
  - "[[../bkurs-be-log]]"
---

# bkurs-be — 인수인계 문서 인덱스

> 신규 합류자 / 백엔드 코드 작업자의 1순위 진입점. 본 인덱스는 `handover/` 내부의 9 가이드 +
> 29 도메인 doc + 기존 운영 가이드 + ADR 디렉토리를 모두 표면화한다.
>
> 프로젝트 정의 / 기술 결정 / 로컬 환경은 [[../bkurs-be]] 가 SSoT, PR 누적은 [[../bkurs-be-log]].

## 0. 빠른 진입

| 역할 | 1순위 |
| --- | --- |
| 처음 합류 | [[getting-started]] (역할별 5분 read path) |
| 도메인 코드 작업 | [[directory-structure]] + [[domains/]] 의 해당 도메인 |
| 인증 / Security 관련 | [[auth-and-security]] |
| DB / migration 작업 | [[database-and-migrations]] |
| 외부 SaaS plug-in | [[external-gateways]] |
| 새 회귀 추가 | [[testing-guide]] |
| 새 PR 시작 | [[stacked-pr]] |
| 운영 / 배포 | [[env-deployment]] |

## 1. 코어 9 가이드 (구조 / 룰 / 워크플로)

진입 순서대로:

1. [[directory-structure]] — 패키지 29 + 7 sub-layer + controller role prefix 규칙
2. [[catalog]] — 29 도메인 한 줄 책임 + sub doc 인덱스 + 의존 그래프
3. [[overview]] — 요청 7 layer + bridge / async / batch
4. [[common]] — `common/{api,error,exception,handler,web,messaging,config}` 사용법
5. [[auth-and-security]] — JWT + Refresh rotation + Cookie + CORS + `@PreAuthorize` matrix
6. [[database-and-migrations]] — Flyway V1~V29 + 명명 규칙 + invariant 위치
7. [[external-gateways]] — file / notification / pdf / ai / stt / payment gateway 추상화
8. [[testing-guide]] — AbstractIntegrationTest + 새 도메인 회귀 추가 절차
9. [[stacked-pr]] — B1~Phase C cycle + dev cascade + 사이클 2 dev 배포

## 2. 29 도메인 별 코드 설명 (`domains/`)

### core (6)
- [[domains/auth]] / [[domains/academy]] / [[domains/announcement]] / [[domains/student]] / [[domains/plan]] / [[domains/course]]

### learning (9)
- [[domains/homework]] / [[domains/statement]] / [[domains/question]] / [[domains/interview]]
- [[domains/record]] / [[domains/discussion]] / [[domains/reading]] / [[domains/performance]] / [[domains/simulation]]

### aggregator (3)
- [[domains/search]] / [[domains/stats]] / [[domains/dashboard]]

### content (4)
- [[domains/concept]] (공용) / [[domains/suhaeng]] / [[domains/saenggibu]] (중등) / [[domains/admission]] (MASTER)

### gateway (5)
- [[domains/file]] / [[domains/pdf]] / [[domains/notification]] / [[domains/ai]] / [[domains/stt]]

### infra (2)
- [[domains/batch]] / [[domains/common]]

## 3. 기존 운영 / 설계 가이드 (유지)

| 문서 | 책임 |
| --- | --- |
| [[getting-started]] | 역할별 5분 진입 가이드 + 문서 지도 |
| [[services-guide]] | 외부 SaaS 7+ 종 발급/설정 절차 |
| [[services-accounts]] | SaaS 자격 inventory (gitignored 참조) |
| [[env-deployment]] | AWS ECS on EC2 토폴로지 + GHCR 배포 + SSM secret + 백업 + 모니터링 |
| [[async-mq-batch]] | async / mq / batch 깊은 SSoT |
| [[hybrid-external-services]] | 외부 SaaS hybrid 전략 |
| [[envelope-migration]] | 도메인별 envelope migration 상태 |
| [[fe-be-gap]] | FE/BE 매핑 gap 추적 |
| [[cycle-3]] | 사이클 3 회고 |
| [[cycle-5-envelope]] | envelope migration 완료 회고 |

## 4. ADR / 사이클 작업 로그

- `decisions/` — 도메인별 ADR 35개 (`decision-{topic}.md`). 도메인 doc 마다 cross-link.
- `task-logs/` — PR 별 작업 로그 (B2A → cycle 2). PR 검색 시 task-log 확인.
- `research/` — 사전 조사 / 비교

## 5. 외부 참고

- [[../../bkurs-fe/bkurs-fe]] — 짝 프론트엔드 hub
- [[../../bkurs-fe/screen-logic-and-api-map]] — FE 화면별 hook + endpoint
- [[../../../20-areas/backend/spring/java-spring/java-spring]] — Spring 일반 hub
- [[../../../20-areas/database/database]] — DB 일반
- [[../../../memory/project_bkurs_be_setup]] — 프로젝트 memory

## cross-link

- [[../bkurs-be]] — 프로젝트 hub
- [[../bkurs-be-log]] — PR / 사이클 진행 로그
