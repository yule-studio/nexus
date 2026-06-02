# 10-projects/ — 프로젝트별 작업

> 특정 프로젝트와 직접 관련된 모든 노트 (결정 / 리서치 / 작업 로그 /
> 미팅 / 회고). 프로젝트 1 개 = 디렉터리 1 개.

**[[../index|↑ vault 인덱스]]**

## 현재 프로젝트

| 프로젝트                  | 진입점                                     | 상태                                                                 |
| --------------------- | --------------------------------------- | ------------------------------------------------------------------ |
| **yule-studio-agent** | [[yule-studio-agent/yule-studio-agent]] | active (자동화 에이전트 본체)                                               |
| **bkurs-fe**          | [[bkurs-fe/bkurs-fe]]                   | in-progress (B-KURS 프론트 — S2 시리즈 28 PR 완료, mock 단계)                |
| **bkurs-be**          | [[bkurs-be/bkurs-be]]                   | in-progress (B-KURS 백엔드 — Spring Boot 3.5 / Java 21, 부트스트랩 중)      |
| **b-kurs-home**       | [[b-kurs-home/b-kurs-home]]             | active (B-KURS 마케팅/홈 사이트 — camshf-crypto/b-kurs 이관/정리, Vite+React) |
| **hompage**           | [[hompage/hompage]]                     | shipped (개인 홈페이지 — GitHub + Tistory 자동 동기화 / Vite + React + TS)    |
| homelab               | [[homelab/]]                            | active                                                             |
| spring-sandbox        | [[spring-sandbox/]]                     | active (학습)                                                        |
| yule-smoke-test       | [[yule-smoke-test/]]                    | smoke 테스트 산출물                                                      |

## 프로젝트 폴더 표준 구조

```
<project>/
├── README.md          ← 프로젝트 진입점 (필수)
├── _moc/              ← 주제별 hub (선택)
├── decisions/         ← 결정 노트
├── research/          ← 리서치 노트
├── task-logs/         ← 작업 로그
├── knowledge/         ← 운영 지식 (선택)
├── meeting-notes/     ← 미팅 기록 (선택)
├── prompts/           ← 프롬프트 템플릿 (선택)
├── references/        ← 외부 reference card (선택)
├── reports/           ← 자동 생성 보고서 (선택)
└── retrospectives/    ← 사이클 회고 (선택)
```

표준 구조의 완전한 예시: [[yule-studio-agent/yule-studio-agent]]

## 운영 규칙

1. **프로젝트 = 종료 시점이 있는 작업**. 종료 없는 지속 주제는 `20-areas/` 로.
2. 새 프로젝트는 **README + decisions/ + research/ + task-logs/** 최소 4
   골격을 먼저 만든다.
3. 프로젝트 종료 시 폴더를 `90-archive/old-projects/<project>/` 로 이주.
4. 같은 주제가 다른 프로젝트와 겹치면 `20-areas/` 또는 `40-patterns/`
   에 추출하고 본 프로젝트는 [[cross-link]] 만 둔다.

## 자동화 에이전트 라우팅

- ResearchPack → `<project>/research/`
- DecisionRecord → `<project>/decisions/`
- TaskLog → `<project>/task-logs/`
- MeetingNote → `<project>/meeting-notes/`
- WorkReport → `<project>/reports/`

매핑 코드: yule-studio-agent repo 의 `agents/obsidian/export.py`

## 관련

- [[../index|↑ vault 인덱스]]
- [[yule-studio-agent/README|yule-studio-agent (표준 구조 예시)]]
