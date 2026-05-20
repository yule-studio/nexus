---
title: "hompage data sources — 각 화면이 어디서 데이터를 가져오는지"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T10:30:00+09:00
tags: [project, hompage, data, automation]
home_hub: hompage
related:
  - "[[hompage]]"
  - "[[architecture]]"
  - "[[deploy]]"
---

# hompage data sources

**[[hompage|↑ hompage hub]]**

> 화면에 보이는 각 데이터가 자동인지 수동인지 + 어디서 오는지.

---

## 1. 한눈에 보는 매핑표

| 화면 영역 | 데이터 출처 | 자동/수동 | 수동 편집 파일 |
| --- | --- | --- | --- |
| Hero 이름·역할·SNS | `src/data/profile.ts` | 수동 | `profile.ts` |
| Topbar status badge ("Available · Q3 2026") | `profile.ts` | 수동 | `profile.ts` `status:` |
| Home GitHub 카드 KPI (stars/PRs/commits) | GitHub GraphQL | **자동** | (없음) |
| Home Awards 카드 latest | `src/data/awards.ts` | 수동 | `awards.ts` |
| Home Certs 카드 list | `src/data/certs.ts` | 수동 | `certs.ts` |
| Home Projects 카드 selected | GitHub repos (active+shipped) | **자동** | (없음) |
| Home Blog 카드 featured | Tistory RSS 최신 글 | **자동** | (없음) |
| Home Stack 카드 top langs | GitHub `/languages` | **자동** | (없음) |
| Home Homelab 카드 hosts | `src/data/homelab.ts` | 수동 | `homelab.ts` |
| Home Calendar 카드 next event | `src/data/events.ts` | 수동 | `events.ts` |
| Home Services 카드 | `src/data/homelab.ts` `services` | 수동 | `homelab.ts` |
| Projects 페이지 전체 목록 | GitHub repos | **자동** | (없음) |
| Project 상세 README | GitHub `/markdown` 렌더 | **자동** | (GitHub 의 README.md) |
| Skills Languages 비율 | GitHub `/languages` bytes | **자동** | (없음) |
| Skills Backend/DevOps/AI | GitHub repo topics + manifest + README + path + code search | **자동** | (없음, 또는 [[#3-skills-자동-규칙]]) |
| Awards 페이지 | `awards.ts` | 수동 | `awards.ts` |
| Certs 페이지 | `certs.ts` | 수동 | `certs.ts` |
| Homelab 페이지 | `homelab.ts` | 수동 | `homelab.ts` |
| Calendar 페이지 | `events.ts` | 수동 | `events.ts` |
| Contact 페이지 | `profile.ts` `links` | 수동 | `profile.ts` |

---

## 2. 자동 데이터 — GitHub

### 2.1 Projects (목록 + 상세)

**스크립트**: `scripts/fetch-github-projects.mjs`
**저장 위치**: `public/projects.json`
**수집 시점**: 매 deploy (push to main 시 + 외부 dispatch 시)

#### 수집 규칙
- 소스: `user:codwithyc` + `org:yule-studio` (workflow 의 `GITHUB_PROJECTS_SOURCES` 환경변수)
- 포함: public repo
- 제외: private / fork / archived / `.` 으로 시작하는 이름
- 최신 push 순으로 정렬, 최대 24개

#### 각 repo 에서 가져오는 것
- 이름, full_name, description, primary language
- 모든 topics
- stars, forks, created/updated/pushed 날짜
- README.md → GitHub `/markdown` API 로 HTML 렌더링해서 저장
- `homepage` 필드 → Live 배지 결정

#### 새 repo 가 자동 등장하려면
1. public repo 만들기
2. README.md 작성
3. 다음 deploy 트리거 (~5분)

### 2.2 Languages (Skills 페이지)

**스크립트**: `scripts/fetch-github-languages.mjs`
**저장 위치**: `public/github-languages.json`

#### 작동 원리
1. user + org 의 모든 repo 목록 조회
2. 각 repo 에서 `/repos/{full_name}/languages` 호출 → 언어별 바이트 수
3. 전체 합산 → 언어별 percent 계산
4. 0.01% 이상인 언어만 노출 (workflow 의 `GITHUB_LANGUAGES_MIN_PERCENT`)

#### Python 이 압도적으로 큰 이유
- yule-studio-agent repo 가 9.7MB Python → 전체 99% 가까이
- 다른 repo 들은 KB 단위라 1% 미만으로 등장

#### 비율 균형을 조정하려면
- repo 에 코드 더 push 하기 (가장 정직)
- 또는 `GITHUB_LANGUAGES_EXCLUDE` 에 큰 repo 추가 (편향 보정)

### 2.3 Topics (Skills 의 Backend/DevOps/AI 카테고리)

**스크립트**: `scripts/fetch-github-topics.mjs`
**저장 위치**: `public/github-topics.json`

이게 가장 복잡함. 5-pass 로 신호를 누적합니다:

| Pass | 신호원 | 비고 |
| --- | --- | --- |
| 1 | repo 의 `topics` 라벨 | 사용자가 GitHub UI 에서 직접 단 것 — 가장 강한 신호 |
| 2 | manifest 파일 (`pyproject.toml` / `package.json` / `pom.xml` / `go.mod` / `Dockerfile` 등) | declared 의존성 매칭 |
| 3 | README.md 본문 | "Built with FastAPI, Docker" 같은 프레임워크 명 매칭 |
| 4 | 파일 경로 (`CLAUDE.md` / `agents/` / `.obsidian/` 등) | 구조적 시그널 |
| 5 | GitHub Code Search API (실제 `import` 문) | 가장 정확하지만 가장 느림 |

#### 카테고리 매핑
스크립트의 `CATEGORIES` 객체에 정의 — Backend / DevOps / AI · Agents 3 카테고리에 각각 ~20 개 framework 명 등록.

#### 새 framework 가 안 잡힐 때
1. **권장**: 해당 repo 의 GitHub UI 에서 Topics 에 직접 추가 (예: `fastapi`, `kubernetes`) — 1순위 신호라 즉시 잡힘
2. **자동**: README 에 framework 명 명시 → 다음 deploy 시 README 스캔에 잡힘
3. **고급**: `scripts/fetch-github-topics.mjs` 의 `CATEGORIES` 와 `MANIFEST_PATTERNS` 에 새 항목 추가 (agent 에게 부탁하는 게 안전)

### 2.4 GitHub Stats (Home 의 KPI)

**스크립트**: `scripts/fetch-github-stats.mjs`
**저장 위치**: `public/github-stats.json`

GraphQL 한 번 + commit count 검색 으로 다음 값 수집:
- stars (총합), prs (총 PR 수), issues (open+closed), repos (수), commits (총합), contributedTo (contributing 한 repo 수)

---

## 3. Skills 자동 규칙

Skills 페이지의 각 카테고리는 다음 우선순위로 채워집니다:

1. GitHub repo 의 declared topics
2. 수동 fallback (`src/data/skills.ts` 의 같은 그룹) — topics 가 0개일 때만 표시

즉 **GitHub repo 에 topic 만 잘 달면 skills.ts 수동 편집이 불필요**. topic 안 단 카테고리만 자기평가 fallback 으로 보임.

### 어떤 topic 이 어떤 카테고리로?
스크립트 안 `CATEGORIES` 객체 참고. 자주 쓰는 매핑:

**Backend**: `fastapi`, `spring-boot`, `django`, `flask`, `express`, `nestjs`, `postgresql`, `redis`, `kafka`, `graphql`, `mongodb`, `sqlalchemy`, `pydantic`, `discord-py`, `caldav`, `icalendar`, `sqlite`

**DevOps**: `docker`, `kubernetes`, `k8s`, `k3s`, `terraform`, `ansible`, `helm`, `github-actions`, `prometheus`, `grafana`, `nginx`, `traefik`, `argocd`, `self-hosted`, `homelab`

**AI · Agents**: `llm`, `openai`, `anthropic`, `claude`, `gpt`, `langchain`, `rag`, `vector-db`, `embedding`, `agent`, `mcp`, `claude-code`, `obsidian`

---

## 4. 자동 데이터 — Tistory Blog

### Blog (Home 의 Featured 카드)

**스크립트**: `scripts/fetch-blog.mjs`
**저장 위치**: `public/blog.json`

#### 작동 원리
1. `https://codingtips.tistory.com/rss` → 최신 글 30개 메타
2. 각 글 페이지 fetch → 읽기 시간 추정
3. **최신 글** 을 featured 로 선택 (조회수 X — Tistory 가 글별 조회수를 public 으로 안 줌)

#### 다른 블로그 URL 로 바꾸려면
workflow `deploy.yml` 의:
```yaml
- name: Sync Tistory blog (RSS + per-post view scrape)
  run: npm run blog:sync
  env:
    BLOG_URL: https://codingtips.tistory.com   ← 여기 바꾸기
```

또는 로컬 실행:
```bash
BLOG_URL=https://newblog.tistory.com npm run blog:sync
```

#### Tistory 가 아닌 다른 블로그 (Velog, Medium 등)
RSS URL 만 같으면 작동. 단 view count 는 안 잡힘 (테마/플랫폼 의존).

---

## 5. 수동 데이터 — `src/data/*.ts`

### 공통 패턴
TypeScript 의 `export const` 배열/객체. 사람이 직접 항목 추가/삭제/수정.

| 파일 | 무엇 |
| --- | --- |
| `profile.ts` | 자기 소개 (이름 / 역할 / 위치 / SNS 링크) |
| `awards.ts` | 수상 이력 |
| `certs.ts` | 자격증 |
| `homelab.ts` | 자가호스팅 서버·서비스 |
| `events.ts` | 캘린더 일정 |
| `posts.ts` | (deprecated — 지금 안 씀, blog.json 으로 대체) |
| `skills.ts` | 자기평가 스킬 (GitHub topic fallback) |
| `projects.ts` | (deprecated — projects.json 으로 대체) |
| `githubStats.ts` | GitHub stats 타입 정의만 (값은 자동) |
| `githubLanguages.ts` | 동일 (값 자동) |
| `githubTopics.ts` | 동일 (값 자동) |
| `blog.ts` | 동일 (값 자동) |

각 파일의 편집 방법은 [[how-to-edit]] 참고.

---

## 6. 데이터 흐름 시퀀스

```
[Day 1, 10:00] codwithyc 가 새 GitHub repo "fastapi-toy" push
[Day 1, 10:00] hompage 의 deploy.yml 는 트리거 안 됨 (그 repo 가 hompage 와 무관하므로)
[Day 2, 09:00] codwithyc 가 hompage 의 README 사소한 수정 push
                  ↓
            deploy workflow 시작
                  ↓
            fetch-github-projects 가 fastapi-toy 도 새로 수집
                  ↓
            fetch-github-languages / topics / blog 도 다시
                  ↓
            ~5분 후 사이트에 fastapi-toy 카드 등장
```

**실시간 트리거** 가 필요하면 `.github/notify-hompage.example.yml` 을 다른 repo 에 배포 + HOMPAGE_DISPATCH_TOKEN 시크릿 설정. 자세한 셋업은 [[deploy#실시간-트리거]].

---

## 7. 데이터 갱신 강제하기

### 로컬에서 (테스트 / 미리보기)
```bash
cd ~/local-dev/hompage
export GH_STATS_TOKEN="ghp_xxxxxx"   # PAT 로

npm run github:stats
npm run github:projects
npm run github:languages
npm run github:topics
npm run blog:sync

npm run dev  # 로컬 미리보기
```

### Live 사이트
```bash
cd ~/local-dev/hompage
git pull origin main
git commit --allow-empty -m "🌏 데이터 강제 갱신"
git push origin main
# ~5분 후 갱신됨
```

---

연결: [[hompage]] · [[architecture]] · [[how-to-edit]] · [[deploy]]
