---
title: "hompage guide — 프로젝트 (LIVE / Code-only / 필터)"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:50:00+09:00
tags: [project, hompage, guide, projects, live]
home_hub: hompage
---

# 프로젝트 관리

**[[../hompage|↑ hompage hub]]**

> Projects 페이지의 카드 / LIVE 배지 / 필터 / 정렬을 어떻게 제어하는지.

---

## 0. 기본 원칙

**프로젝트는 GitHub repos 에서 자동.** 코드에서 일일이 추가/제거할 필요 없음.

- 추적 source: `user:codwithyc` + `org:yule-studio` (workflow 환경변수)
- 매 deploy 마다 자동 갱신
- 추적 규칙은 fetcher 가 결정 — public + non-fork + non-archived

---

## 1. 새 프로젝트가 등장하려면

GitHub 에서:
1. **Public** repo 만들기 (private 는 자동 제외)
2. README.md 작성 (없으면 카드의 sub 가 비게 됨)
3. (선택) About → Topics 에 `fastapi`, `docker` 등 추가 → Skills 페이지에도 카테고리화
4. (선택) About → Website 에 배포 URL 추가 → **LIVE 배지** 부여
5. main 에 push

hompage 의 다음 deploy (약 5분) 후 사이트에 등장.

### 즉시 갱신
```bash
cd ~/local-dev/hompage
git pull
git commit --allow-empty -m "🌏 데이터 강제 갱신"
git push origin main
```

또는 https://github.com/yule-studio/hompage/actions → "Run workflow".

---

## 2. LIVE 배지 설정 — 핵심

### 원리
fetcher 가 GitHub API 의 `repo.homepage` 필드를 읽어 다음과 같이 처리:
- `homepage` 비어 있음 → Code-only
- `homepage` 채워져 있음 → **LIVE** (배지 + accent border)

### 설정 방법

#### A. GitHub UI (가장 쉬움)
1. `https://github.com/yule-studio/{repo}` 또는 `https://github.com/codwithyc/{repo}`
2. 우측 상단 **About** 박스의 ⚙️ 톱니바퀴
3. **Website** 필드에 배포 URL 입력
   - 예: `https://yule-studio.github.io/hompage/`
   - 예: `https://my-app.vercel.app`
   - 예: `https://my-domain.com`
4. **Save changes**

#### B. gh CLI (한 번에 여러 개)
```bash
gh repo edit yule-studio/hompage --homepage "https://yule-studio.github.io/hompage/"
gh repo edit yule-studio/yule-studio-agent --homepage "https://yule-agent.example.com"
gh repo edit codwithyc/spring-sandbox --homepage ""   # 비우면 LIVE 해제
```

### 확인
```bash
gh repo list yule-studio --json name,homepageUrl --jq '.[] | select(.homepageUrl != "")'
gh repo list codwithyc  --json name,homepageUrl --jq '.[] | select(.homepageUrl != "")'
```

### 다음 deploy 후 사이트 반영 (5분)

---

## 3. 필터 (All / Live / Code-only)

Projects 페이지 상단의 3 탭. 코드 위치: `src/pages/Projects/Projects.tsx`.

### 현재
```tsx
const FILTERS: Array<{ id: Filter; label: string }> = [
  { id: "all", label: "All" },
  { id: "live", label: "Live · 실행 가능" },
  { id: "code", label: "Code-only · GitHub" },
];
```

### 라벨 바꾸기
```tsx
{ id: "live", label: "🚀 Production" },
{ id: "code", label: "📦 Source" },
```

### 탭 추가 (예: "Active" — 최근 6개월 내 push)
복잡함 — 새 Filter 타입 + 필터 함수 추가 필요. agent 부탁 권장.

### 기본 선택 탭 바꾸기 (예: 처음부터 LIVE만)
```tsx
const [filter, setFilter] = useState<Filter>("live");
//                                          ↑ "all" → "live"
```

---

## 4. 카드 순서

기본 정렬: `pushed_at` 최신순 (fetcher 에서).

### 다른 기준으로 정렬
`scripts/fetch-github-projects.mjs` 의 마지막 부분:
```javascript
const filteredRepos = filterRepos(dedupedRepos)
  .sort((a, b) => new Date(b.pushed_at ?? 0).getTime() - new Date(a.pushed_at ?? 0).getTime());
```

#### stars 순
```javascript
.sort((a, b) => (b.stargazers_count ?? 0) - (a.stargazers_count ?? 0));
```

#### 알파벳 순
```javascript
.sort((a, b) => a.name.localeCompare(b.name));
```

#### LIVE 가 먼저
```javascript
.sort((a, b) => {
  const aLive = a.homepage ? 1 : 0;
  const bLive = b.homepage ? 1 : 0;
  if (aLive !== bLive) return bLive - aLive;
  return new Date(b.pushed_at ?? 0).getTime() - new Date(a.pushed_at ?? 0).getTime();
});
```

---

## 5. 특정 repo 만 강제 포함 / 제외

### 환경변수로 (workflow 또는 로컬)

#### 포함만
```yaml
GITHUB_PROJECTS_INCLUDE: hompage,yule-studio-agent,spring-sandbox
```
→ 이 3개만 표시.

#### 제외
```yaml
GITHUB_PROJECTS_EXCLUDE: github-stats-box,productive-box
```
→ 이 2개는 절대 안 나옴.

#### 최대 개수
```yaml
GITHUB_PROJECTS_MAX: 12
```
→ 최대 12개만.

`.github/workflows/deploy.yml` 의 "Generate GitHub projects" step env 에 추가.

#### 로컬 테스트
```bash
GH_STATS_TOKEN=ghp_... \
GITHUB_PROJECTS_INCLUDE="hompage,spring-sandbox" \
npm run github:projects
```

---

## 6. Fork / Archived repo 포함

기본은 제외. 포함하려면 환경변수:
```yaml
GITHUB_PROJECTS_INCLUDE_FORKS: "true"
GITHUB_PROJECTS_INCLUDE_ARCHIVED: "true"
```

---

## 7. 다른 owner 추가

기본 source: `user:codwithyc + org:yule-studio`. 추가하려면:
```yaml
GITHUB_PROJECTS_SOURCES: user:codwithyc,org:yule-studio,org:my-other-org,user:friend-handle
```

---

## 8. LIVE 카드 시각 변경

`src/styles/layout.css`:
```css
.project-card--live {
  border-color: color-mix(in srgb, var(--accent) 60%, var(--border));
  background: linear-gradient(
    180deg,
    color-mix(in srgb, var(--accent) 6%, var(--bg-card)) 0%,
    var(--bg-card) 100%
  );
}
.project-card--live:hover { border-color: var(--accent); }
```

### LIVE 만 더 두드러지게
```css
.project-card--live {
  border-color: var(--accent);
  background: color-mix(in srgb, var(--accent) 10%, var(--bg-card));
  box-shadow: 0 0 20px var(--accent-soft);
}
```

### LIVE 라벨 텍스트 바꾸기
`src/pages/Projects/Projects.tsx`:
```tsx
{live ? (
  <StatusBadge status="ok" label="LIVE" />   ← "LIVE" 를 다른 단어로
) : (
  <StatusBadge status={statusTone[p.status]} label={p.status} />
)}
```

예: `label="🚀 SHIPPED"` 또는 `label="실행 가능"`.

---

## 9. Card 안에 어떤 정보 보이는지

```tsx
<Card>
  <header>
    <Link>{p.name}</Link>             ← repo 이름
    <StatusBadge status>{label}</StatusBadge>   ← LIVE 또는 상태
  </header>
  <p className="card-sub">{p.summary}</p>      ← repo description
  <div className="chip-row">
    {p.tags.map(t => <span className="chip">{t}</span>)}   ← language + topics
  </div>
  <footer>
    <Link to={`/projects/${p.slug}`}>open project</Link>
    <span>{p.homepageUrl ? <a>↗ live</a> : null} · {p.year}</span>
  </footer>
</Card>
```

### 표시 항목 변경
- `summary` 안 보이게 → `<p>` 삭제
- 태그 갯수 제한 → `p.tags.slice(0, 5)`
- 별 수 표시 추가 → `<span>★ {p.stars}</span>`

---

## 10. Project Detail 페이지

각 카드 클릭 시 `/projects/{slug}` 로 이동.

### 내용
- 위쪽 카드 (메타) — 날짜, full_name, GitHub / Live 버튼, 태그
- 아래 — README.md 의 HTML 렌더링

### 코드
`src/pages/Projects/ProjectDetail.tsx`.

### 자주 손 댈 곳
- 메타 라벨 변경
- 표시 안 할 필드 제거

### README 가 너무 길면
GitHub 의 README.md 자체를 수정. hompage 측 코드 안 만짐.

### README 안 이미지 / 텍스트 잘리면
`src/styles/layout.css` 의 `.markdown-body` 룰 손봐야. 이미 `min-width: 0` + `grid-template-columns: minmax(0, 1fr)` 적용됨 — 일반적으로 안 잘림.

---

## 11. 빌드 검증

```bash
npm run typecheck
npm run lint
npm run build
```

다 통과해야 push.

---

연결: [[../hompage]] · [[../data-sources]] · [[../styling/cards]] · [[skills]]
