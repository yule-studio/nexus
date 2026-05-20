---
title: "hompage guide — Skills (언어 비율 / 토픽 / 카테고리 매핑)"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:55:00+09:00
tags: [project, hompage, guide, skills, topics, languages]
home_hub: hompage
---

# Skills 관리

**[[../hompage|↑ hompage hub]]**

> Skills 페이지가 어떻게 채워지는지 + 직접 통제하는 방법.

---

## 0. 두 종류

| 섹션 | 자동 출처 | 수동 fallback |
| --- | --- | --- |
| Languages | GitHub `/repos/{}/languages` (코드 바이트) | (없음 — 항상 자동) |
| Backend / DevOps / AI · Agents | GitHub repo topics + manifest + README + path + code-search | `data/skills.ts` 의 같은 그룹 (자동 hit 0 일 때만) |

---

## 1. Languages — 언어 비율

### 작동
- 모든 추적 repo 의 `/languages` 응답 합산
- 언어별 byte count 기준 % 계산
- `MIN_PERCENT` (현재 0.01%) 이상만 노출

### Python 이 압도적인 이유
yule-studio-agent 한 repo 가 ~9.7MB Python → 전체 코드의 97~98%. 다른 언어는 KB 단위.

### 비율 조정 방법

#### A. 다른 언어 repo 에 코드 더 push
가장 정직. JS / Go / Rust repo 에 실제 코드 늘리면 자동으로 비율 등장.

#### B. 특정 repo 제외 (편향 완화)
`.github/workflows/deploy.yml` 의 "Generate GitHub languages" step env 에:
```yaml
GITHUB_LANGUAGES_EXCLUDE: yule-studio-agent
```
→ yule-studio-agent 의 Python 9.7MB 가 빠지고 나머지 repo 들만으로 계산.

#### C. minPercent 조정
```yaml
GITHUB_LANGUAGES_MIN_PERCENT: '0.5'    # 0.5% 미만 숨김 (작은 언어 정리)
GITHUB_LANGUAGES_MIN_PERCENT: '0'      # 전부 노출 (현재 0.01)
```

#### D. 최대 언어 수
```yaml
GITHUB_LANGUAGES_MAX: '10'   # 상위 10개만
```

### 표시 변경

#### Skill 카드 안 percent bar 더 두껍게
`src/styles/layout.css`:
```css
.bar { height: 6px; }    /* 4px → 6px */
```

#### 언어 색
fetcher 의 `COLORS` 객체 (Linguist 기준 색):
```javascript
const COLORS = {
  Python: "#3572A5",
  TypeScript: "#3178c6",
  Java: "#b07219",
  Go: "#00ADD8",
  ...
};
```
새 색을 추가하려면 `scripts/fetch-github-languages.mjs` 의 `COLORS` 에 항목 추가.

---

## 2. Topics — Backend / DevOps / AI · Agents

### 작동 (5-pass)
| Pass | 신호원 | 효과 |
| --- | --- | --- |
| 1 | repo 의 `topics` (가장 강함) | 사용자가 직접 단 라벨 |
| 2 | `requirements.txt` / `package.json` / `pyproject.toml` / `pom.xml` / `Dockerfile` 등 manifest 파일 | 선언된 의존성 매칭 |
| 3 | README.md 본문 | "Built with FastAPI" 같은 산문 매칭 |
| 4 | 파일 경로 (`CLAUDE.md` / `agents/` / `.obsidian/` 등) | 구조 시그널 |
| 5 | GitHub Code Search API | `from anthropic` 같은 실제 import |

5 신호를 누적해서 카테고리별 토픽 수집.

### 매핑 정의 위치
`scripts/fetch-github-topics.mjs` 의 `CATEGORIES` 객체:

```javascript
const CATEGORIES = {
  Backend: [
    ["fastapi", "FastAPI"],
    ["spring-boot", "Spring Boot"],
    ["spring", "Spring"],
    ["postgresql", "PostgreSQL"],
    ...
  ],
  DevOps: [
    ["docker", "Docker"],
    ["kubernetes", "Kubernetes"],
    ["k3s", "k3s"],
    ["terraform", "Terraform"],
    ["github-actions", "GitHub Actions"],
    ...
  ],
  "AI · Agents": [
    ["llm", "LLM"],
    ["anthropic", "Anthropic"],
    ["claude-code", "Claude Code"],
    ["mcp", "MCP"],
    ["obsidian", "Obsidian"],
    ...
  ],
};
```

각 항목 = `[topic-slug, 표시-라벨]`.

### 새 framework 가 안 잡힐 때

#### A. repo 에 직접 topic 달기 (가장 즉시)
1. https://github.com/yule-studio/{repo} 또는 codwithyc/{repo}
2. About → ⚙️ → Topics 입력란
3. `fastapi`, `kubernetes`, `claude-code` 등 추가
4. Save

다음 deploy 후 해당 카테고리에 등장.

#### B. README 에 framework 명 추가
`README.md` 본문에 자연스럽게:
```markdown
## Tech stack
- FastAPI / Pydantic
- PostgreSQL + Redis
- Docker / k3s
```

→ Pass 3 (README scan) 에서 자동 매칭.

#### C. CATEGORIES 매핑 자체에 추가
새 framework 가 인식 사전에 없으면 → `scripts/fetch-github-topics.mjs` 의 `CATEGORIES` 에 추가:

```javascript
Backend: [
  ...,
  ["nestjs", "NestJS"],          // 새 항목
],
```

그리고 `MANIFEST_PATTERNS` 에 매니페스트 매칭 정규식:
```javascript
const MANIFEST_PATTERNS = [
  ...,
  ["nestjs", /@nestjs\//i],      // package.json 의 @nestjs/ 매칭
];
```

복잡함 — agent 부탁 권장.

### 카테고리 추가 (예: "Mobile")

#### Step 1 — fetcher 의 CATEGORIES 에 추가
```javascript
const CATEGORIES = {
  Backend: [...],
  DevOps: [...],
  "AI · Agents": [...],
  Mobile: [
    ["react-native", "React Native"],
    ["flutter", "Flutter"],
    ["swift", "Swift"],
  ],
};
```

#### Step 2 — MANIFEST_PATTERNS 에 패턴 추가
```javascript
const MANIFEST_PATTERNS = [
  ...,
  ["react-native", /"react-native"/i],
  ["flutter", /flutter:|sdk:\s*flutter/i],
];
```

#### Step 3 — (선택) 자기평가 fallback
`src/data/skills.ts` 에 새 그룹 추가:
```typescript
{
  group: "Mobile",
  items: [
    { name: "React Native", level: 60 },
    { name: "Flutter", level: 40 },
  ],
},
```

### Skill 그룹 순서 변경
Skills 페이지에서 카테고리 순서는 `topicsPayload.categories` 순서를 따름. `scripts/fetch-github-topics.mjs` 의 `CATEGORIES` 객체 정의 순서대로 → 객체 키 순서를 바꾸면 끝.

---

## 3. Home 의 Stack 카드

Home 우측 작은 Stack 카드 — 상위 언어 chip 4개.

### 보여줄 갯수 변경
`src/pages/Home/Home.tsx`:
```tsx
{(topLanguages.length
  ? topLanguages.slice(0, 4)
  //                    ↑ 4 → 5 / 6 / ...
  : skills[0].items.slice(0, 4).map((s) => ({ ... }))
).map((lang) => ...)}
```

4 너머는 카드 본문이 좁아서 잘림. 폭을 늘리려면 카드 span 변경 (sm → md).

### Stack 카드 자체를 md (8col) 로
`Home.tsx`:
```tsx
<Card span="md" hoverable className="home-stack-card">
//        ↑ "sm" → "md"
```

그러면 6~8개도 fit.

---

## 4. Skills 페이지 카드 자체 변경

`src/pages/Skills/Skills.tsx`:
- 각 카테고리 → `<section>` → `<SectionHeader>` + `<div className="section-grid">` + `<Card>` 들
- 각 Card 안에: title + percent + bar + repos 정보

#### 한 행에 보여줄 카드 수
`src/styles/layout.css`:
```css
.section-grid {
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  /*                                       ↑ 280px 키우면 카드가 더 크게, 한 행 카드 수 감소 */
}
```

#### Card 자체 디자인 변경
`.bar` 두께 / 색 / 모양 → [[../styling/cards#9-bar-progress]]

---

## 5. Code Search 토픽 추가 (5th pass)

특정 import 도 감지하고 싶으면 `scripts/fetch-github-topics.mjs` 의 `CODE_SEARCH_PATTERNS`:

```javascript
const CODE_SEARCH_PATTERNS = [
  ["anthropic",    '"from anthropic" OR "import anthropic"'],
  ["openai",       '"from openai" OR "import openai"'],
  ["langchain",    '"from langchain"'],
  ...
];
```

검색 쿼리 = quoted phrase OR phrase. GitHub Code Search 문법.

새 패턴 추가:
```javascript
["redis-py", '"import redis" OR "from redis"'],
```

CATEGORIES 에 `["redis-py", "redis-py"]` 도 같이 추가.

**주의**: code search 는 PAT 가 필요 + GitHub 가 분당 10 req 로 제한. 패턴 너무 많이 추가하면 deploy 가 느려짐 (현재 13개 패턴 × 2 source × 7s throttle = ~3분).

---

## 6. Skills 페이지 본문 카드 의미

### Languages 카드
```
GitHub 코드 바이트 기준 상위 언어.

Python                       97.7%
████████████████████████████ (bar)
1 repo · 9.7 MB
```

### Backend / DevOps / AI 카드
```
FastAPI
2 repos

(또는 자기평가 fallback)
FastAPI                          88
████████████████████             (bar)
```

---

## 7. 변경 후 검증

```bash
# Skills 데이터 재생성
GH_STATS_TOKEN=ghp_... npm run github:languages
GH_STATS_TOKEN=ghp_... npm run github:topics

# dev 서버
npm run dev
# Skills 페이지 (/skills) 열기
```

문제 없으면 commit + push.

---

연결: [[../hompage]] · [[../data-sources]] · [[projects]] · [[../styling/cards]]
