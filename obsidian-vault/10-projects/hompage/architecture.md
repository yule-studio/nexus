---
title: "hompage architecture — 부품이 어디서 돌아가는지"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T10:10:00+09:00
tags: [project, hompage, architecture, frontend]
home_hub: hompage
related:
  - "[[hompage]]"
  - "[[data-sources]]"
  - "[[deploy]]"
---

# hompage architecture

**[[hompage|↑ hompage hub]]**

> "전체 그림". FE 무지 사용자 기준으로 어떤 부품이 어디서 돌아가는지만 설명.

---

## 1. 전체 흐름 한 장

```
사용자 (브라우저)
     │ https://yule-studio.github.io/hompage/
     ▼
┌────────────────────────────────────────┐
│  GitHub Pages CDN (정적 파일 호스팅)    │
│   - index.html                         │
│   - assets/index-xxx.js                │
│   - assets/index-xxx.css               │
│   - projects.json / blog.json 등 5종   │
└─────────────────▲──────────────────────┘
                  │ 자동 배포 (push to main → 5분)
                  │
┌─────────────────┴──────────────────────┐
│  GitHub Actions: Deploy workflow       │
│   1) GitHub API 로 repo / 언어 / 토픽  │
│   2) Tistory RSS 로 블로그             │
│   3) → public/*.json 5개 생성          │
│   4) Vite 가 React → dist/ 빌드        │
│   5) Pages 로 업로드                    │
└────────────────────────────────────────┘

데이터 소스 (외부)
- GitHub API: codwithyc 개인 + yule-studio org repos
- Tistory RSS: https://codingtips.tistory.com/rss
```

## 2. 폴더 구조 (필요한 것만)

```
~/local-dev/hompage/
├─ src/                    ← 모든 React/TS 소스 코드
│  ├─ App.tsx              ← URL → 페이지 매핑 (라우터)
│  ├─ main.tsx             ← 진입점 (브라우저가 처음 실행)
│  ├─ components/          ← 재사용 부품 (Card / Topbar / Nav 등)
│  ├─ pages/               ← 각 URL 별 화면 (Home / Projects ...)
│  ├─ data/                ← 하드코딩 데이터 (자격증 / 수상 등)
│  ├─ hooks/               ← React 데이터 로더 (use~)
│  └─ styles/              ← CSS (tokens.css = 색·폰트, layout.css = 레이아웃)
├─ public/                 ← 빌드 안 거치고 그대로 배포되는 파일
│  ├─ favicon.svg
│  └─ *.json               ← GitHub Actions 가 자동 생성
├─ scripts/                ← 데이터 수집 노드 스크립트 (수동 실행도 가능)
│  ├─ fetch-github-stats.mjs
│  ├─ fetch-github-projects.mjs
│  ├─ fetch-github-languages.mjs
│  ├─ fetch-github-topics.mjs
│  └─ fetch-blog.mjs
├─ .github/workflows/
│  ├─ ci.yml               ← PR / push 시 lint+typecheck+build 검사
│  ├─ deploy.yml           ← main push → 빌드 + Pages 배포
│  └─ notify-hompage.example.yml ← 다른 repo 가 hompage 갱신 트리거 (선택)
├─ package.json            ← 의존성 목록 + npm 명령 모음
├─ vite.config.ts          ← Vite 빌드 설정
└─ tsconfig.json           ← TypeScript 설정
```

## 3. 화면 ↔ 파일 매핑

| 화면 보고 싶을 때 | 어떤 파일이 그리고 있나 |
| --- | --- |
| Home 의 hero 자기소개 | `src/pages/Home/Home.tsx` 안의 `<section className="hero">` |
| Home 의 GitHub 카드 | `Home.tsx` 의 KPI 카드 + `data/githubStats.ts` |
| Home 의 Blog featured | `Home.tsx` Blog 카드 + `hooks/useBlog.ts` + `public/blog.json` |
| 상단 검은 막대 | `src/components/Topbar/Topbar.tsx` |
| 목차 (Home / Projects ...) | `src/components/Navigation/Navigation.tsx` |
| 페이지 전환 애니메이션 | `src/components/Layout/Layout.tsx` 의 `key={pathname}` + `styles/layout.css` 의 `.page-anim` |
| Projects 목록 카드 | `src/pages/Projects/Projects.tsx` |
| 개별 프로젝트 README | `src/pages/Projects/ProjectDetail.tsx` |

## 4. 데이터 소스 4종 + 1

| 데이터 | 출처 | 수집 시점 | 결과 파일 |
| --- | --- | --- | --- |
| GitHub repos 목록 | GitHub REST `/users/codwithyc/repos` + `/orgs/yule-studio/repos` | 매 deploy 마다 | `public/projects.json` |
| 코드 언어 비율 | GitHub REST `/repos/{}/languages` | 매 deploy | `public/github-languages.json` |
| Skills 토픽 (FastAPI / Docker 등) | GitHub repo 의 topics + manifest 파일 + README + 디렉터리 + code search | 매 deploy | `public/github-topics.json` |
| GitHub KPI (stars / PRs / commits) | GitHub GraphQL | 매 deploy | `public/github-stats.json` |
| Tistory 최신 블로그 | https://codingtips.tistory.com/rss | 매 deploy | `public/blog.json` |

추가로 [[data-sources]] 에 자세한 매핑 설명.

## 5. "직접 만질 수 있는 영역" 색칠

```
src/data/ ✏️              ← 자유롭게 텍스트 / 숫자 수정 (사람용 데이터)
src/styles/tokens.css ✏️  ← 색·여백·폰트만 살짝 (값만 바꾸기)
src/pages/*.tsx ⚠️        ← 구조 변경은 주의 (텍스트 변경은 OK)
src/components/*.tsx ⚠️   ← 동일 — 텍스트만 OK
public/*.json ❌          ← GitHub Actions 가 덮어씀, 직접 수정 무의미
scripts/*.mjs ❌          ← Node.js 코드, FE 무지 사용자에겐 위험
.github/workflows/ ❌     ← CI 깨질 수 있음
```

자세한 "어디를 어떻게 만져?" 는 [[how-to-edit]].

## 6. 외부 의존성 (실패 시 영향)

| 외부 서비스 | 끊기면? |
| --- | --- |
| GitHub API | deploy 실패 (이전 데이터 그대로 유지, 사이트 살아있음) |
| Tistory RSS | blog 카드 fallback (data/posts.ts 하드코딩) |
| GitHub Pages | 사이트 자체 안 보임 (드물게 발생, 대형 incident) |

자세한 트러블슈팅은 [[troubleshooting]].

---

연결: [[hompage]] · [[how-to-edit]] · [[data-sources]] · [[deploy]]
