---
title: "hompage — 개인 홈페이지 hub"
kind: project
project: hompage
agent: engineering-agent/tech-lead
status: shipped
created_at: 2026-05-20T10:00:00+09:00
tags: [project, frontend, react, vite, github-pages, tistory, portfolio]
home_hub: projects
related:
  - "[[../projects]]"
  - "[[how-to-edit]]"
  - "[[data-sources]]"
  - "[[deploy]]"
  - "[[troubleshooting]]"
---

# hompage — 개인 홈페이지 hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-20 | engineering-agent/tech-lead | 최초 작성 — FE 무지 사용자가 직접 손볼 수 있게 plain 한국어 가이드 |

**[[../projects|↑ 10-projects]]**

---

## 1. 한 줄 요약

GitHub repo / Tistory blog 데이터를 자동으로 끌어와서 보여주는 **개인 홈페이지** 입니다. 다크 dashboard 디자인이고 한 페이지에 자기 소개 / 프로젝트 / 스킬 / 자격증 / Homelab 상태가 다 모여 있어요.

- **사이트**: https://yule-studio.github.io/hompage/
- **소스 코드**: https://github.com/yule-studio/hompage
- **로컬 폴더**: `~/local-dev/hompage`

## 2. "내가 직접 손대고 싶다" 시작 가이드

> 코드 한 줄 안 짜본 사람도 따라할 수 있게 단계별로 정리. **TS/JS 문법 몰라도 됩니다.**

### 자주 바꿀 것
| 하고 싶은 일 | 어떤 문서 보고 / 어떤 파일 만지기 |
| --- | --- |
| 내 이름·역할·위치 바꾸기 | [[how-to-edit#1-프로필-바꾸기]] → `src/data/profile.ts` |
| 자격증·수상 목록 바꾸기 | [[how-to-edit#3-자격증-수상-홈랩-목록-바꾸기]] → `src/data/certs.ts` / `awards.ts` |
| Homelab 호스트·서비스 목록 | [[how-to-edit#3-자격증-수상-홈랩-목록-바꾸기]] → `src/data/homelab.ts` |
| GitHub 가서 보여줄 repo 추가 | (자동) GitHub 에 push 만 하면 됨, 따로 작업 없음 — [[data-sources#projects]] |
| 프로젝트에 **LIVE 배지** 붙이기 | repo About → Website 필드 한 줄, 코드 수정 0 — [[how-to-edit#4-A-프로젝트를-LIVE-로-표시하기]] |
| 블로그 글 갱신 | (자동) Tistory 에 글 올리면 RSS 통해서 따라옴 — [[data-sources#blog]] |
| 색 / 폰트 바꾸기 | [[how-to-edit#5-색이나-폰트-바꾸기]] → `src/styles/tokens.css` |
| 페이지 메뉴 (목차) 항목 추가 | [[how-to-edit#6-목차-항목-추가]] → `src/components/Navigation/Navigation.tsx` |

### 절대 만지지 말 것 (망가짐)
- `node_modules/` 폴더 전체 — 외부 도구 캐시, 손대지 마세요
- `dist/` 폴더 — 빌드 결과물, 자동 생성됨
- `package-lock.json` — 의존성 버전 자물쇠
- `public/*.json` (`projects.json` 등) — GitHub Actions 가 자동 채움. 직접 수정해도 다음 deploy 에 덮어씌워짐

## 3. 문서 지도

- [[architecture]] — 큰 그림: 어떤 부품이 어디서 돌아가는지 (사용자 / GitHub / Tistory / Pages 흐름)
- [[how-to-edit]] — "이거 바꾸려면 어디 만져야 해?" 케이스별 직접 수정 가이드
- [[data-sources]] — 각 화면이 어떤 데이터를 어디서 가져오는지 (자동 동기화 출처)
- [[deploy]] — git push 부터 사이트 갱신까지 흐름 + 트러블슈팅
- [[troubleshooting]] — 자주 만나는 문제 + 빠른 해결

## 4. 기술 스택 (참고만, 몰라도 됩니다)

- **빌드 도구**: Vite 5 — 코드를 묶어서 정적 사이트로 만들어줌
- **UI 프레임워크**: React 18 — 화면 조각 (컴포넌트) 단위로 작성
- **언어**: TypeScript 5 — JS 에 타입 검사 붙은 것 (안전망)
- **라우팅**: React Router 6 — 페이지 사이 이동
- **스타일**: 순수 CSS (Tailwind 같은 거 안 씀 — 토큰 변수로 통일)
- **호스팅**: GitHub Pages — 무료 정적 호스팅
- **자동화**: GitHub Actions — push 하면 빌드+배포 자동
- **데이터 소스**: GitHub REST API (repo 정보) + Tistory RSS (블로그)

## 5. 페이지 구성

| 경로 | 화면 | 비고 |
| --- | --- | --- |
| `/` | Home — 자기소개 hero + 8개 정보 카드 | 첫 진입 |
| `/projects` | Projects 목록 (Live / Code 필터) | GitHub repo 자동 |
| `/projects/:slug` | Projects 상세 (README HTML 렌더) | repo 별 페이지 |
| `/skills` | Skills (Languages / Backend / DevOps / AI) | GitHub 코드 바이트 + 토픽 |
| `/awards` | 수상 이력 | `data/awards.ts` 직접 |
| `/certs` | 자격증 목록 | `data/certs.ts` 직접 |
| `/homelab` | 자가호스팅 인프라 상태 | `data/homelab.ts` 직접 |
| `/calendar` | 일정 | `data/events.ts` 직접 |
| `/contact` | 연락 / 협업 | `data/profile.ts` links |

블로그는 **외부 사이트** (https://codingtips.tistory.com/) 로 직접 이동 — hompage 안에 별도 페이지 없음.

## 6. 데이터 흐름 (한눈에)

```
[GitHub repos]   [Tistory RSS]   [data/*.ts hardcode]
       ↓               ↓                  ↓
       └──── GitHub Actions (deploy.yml) ────┘
                      ↓
                  4 fetcher 가 public/*.json 생성
                      ↓
                  Vite build → dist/
                      ↓
                  GitHub Pages 배포
                      ↓
        https://yule-studio.github.io/hompage/
```

자세한 흐름은 [[architecture]] 참고.

## 7. 자주 묻는 질문

**Q1. 코드 한 줄도 못 짜는데 정말 손댈 수 있나요?**
→ 데이터 파일 (`src/data/*.ts`) 은 `name: "..."`, `year: 2024` 같은 단순한 텍스트/숫자만 바꾸면 됩니다. JS 문법 몰라도 됩니다. 자세한 패턴은 [[how-to-edit]].

**Q2. 내가 잘못 만져서 사이트가 깨지면?**
→ git push 하기 전엔 사이트에 안 올라갑니다. `npm run dev` 로 로컬에서 먼저 미리 보기 가능. 망가지면 `git checkout .` 으로 전부 원래대로.

**Q3. GitHub 에 push 하면 바로 사이트 갱신?**
→ main 브랜치에 push 하면 ~5분 안에 자동 배포. 자세한 타이밍은 [[deploy#배포-타임라인]].

**Q4. 다른 GitHub repo (예: 새로 만든 repo) 도 Projects 페이지에 자동으로 나오나요?**
→ 네, codwithyc 개인 repo 든 yule-studio org repo 든 다 자동. 단 fork 와 archived 는 빠짐. 자세한 규칙은 [[data-sources#projects]].

**Q5. 블로그 글 수가 안 늘어나요.**
→ deploy 워크플로가 돌 때만 동기화. main 에 commit 없이는 안 도니까 빈 commit (`git commit --allow-empty`) 으로 트리거 가능. [[deploy#수동-갱신]].

---

연결: [[architecture]] · [[how-to-edit]] · [[data-sources]] · [[deploy]] · [[troubleshooting]]
