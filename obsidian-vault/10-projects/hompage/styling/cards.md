---
title: "hompage styling — Card 컴포넌트 해부"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:15:00+09:00
tags: [project, hompage, styling, cards, css]
home_hub: hompage
related:
  - "[[tokens]]"
  - "[[layout-grid]]"
  - "[[../hompage]]"
---

# Card 해부

**[[../hompage|↑ hompage hub]]**

> 사이트의 모든 박스는 `.card`. 그 내부 구조 + span 변형 + scoped 클래스.

---

## 1. Card 기본 구조

### JSX (간단)
```tsx
<article className="card" data-span="sm">
  <header className="card-head">
    <span className="label">/ github</span>
    <StatusBadge status="ok" label="active" />
  </header>
  <h3 className="card-title">GitHub</h3>
  <p className="card-sub">개인 프로젝트와 실험 코드.</p>
  <div className="card-body">
    {/* 본문 */}
  </div>
  <footer className="card-footer">
    <a className="card-link">view github</a>
    <span className="mono faint">@codwithyc</span>
  </footer>
</article>
```

### 5 슬롯
| 슬롯 | 클래스 | 역할 |
| --- | --- | --- |
| 1. head | `.card-head` | 좌: 라벨 (`/ github`), 우: status badge / chip |
| 2. title | `.card-title` | 큰 글씨 제목 (15px) |
| 3. sub | `.card-sub` | 작은 회색 설명 (12.5px) |
| 4. body | `.card-body` | 자유 컨텐츠 영역 (list / KPI / chip 등) |
| 5. footer | `.card-footer` | 좌: 액션 링크, 우: 메타 정보 |

### 모든 슬롯이 필수는 아님
title 만 있어도 OK, body 만 있어도 OK. 흔한 패턴:
- KPI 카드 (GitHub): head + title + sub + body (KPI 4개) + footer
- 리스트 카드 (Projects): head + title + body (ul) + footer
- 미니 카드 (Awards): head + title + sub + body (1줄)

---

## 2. CSS 기본

```css
.card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius-lg);   /* 12px */
  padding: var(--space-5);            /* 20px */
  display: flex; flex-direction: column;
  gap: var(--space-3);                /* 내부 슬롯 간격 12px */
  position: relative;
  transition: border-color var(--t-fast), background var(--t-fast);
  min-width: 0;
  overflow: hidden;   /* 컨텐츠 넘침 잘림 */
}
.card:hover { border-color: var(--border-strong); }
.card[data-hoverable="true"]:hover { background: var(--bg-card-hover); }
```

### 카드 padding 바꾸기
```css
.card { padding: var(--space-4); }  /* 20 → 16 (더 컴팩트) */
.card { padding: var(--space-6); }  /* 20 → 24 (더 여유) */
```

### 카드 모서리 사각형
```css
.card { border-radius: 0; }
```

### 카드 그림자 추가
```css
.card {
  box-shadow: 0 2px 10px rgba(0,0,0,0.15);
}
```

### Hover 시 살짝 떠오르게
```css
.card {
  transition: transform var(--t-fast), border-color var(--t-fast);
}
.card[data-hoverable="true"]:hover {
  transform: translateY(-2px);
}
```

---

## 3. Span 변형 (크기)

Home dashboard 의 12-column grid 에서 카드의 가로/세로 칸 수.

```css
.card[data-span="sm"]   { grid-column: span 4;  grid-row: span 3; }
.card[data-span="md"]   { grid-column: span 8;  grid-row: span 3; }
.card[data-span="wide"] { grid-column: span 8;  grid-row: span 3; }
.card[data-span="lg"]   { grid-column: span 8;  grid-row: span 3; }
.card[data-span="xl"]   { grid-column: span 12; grid-row: span 3; }
.card[data-span="tall"] { grid-column: span 4;  grid-row: span 6; }
.card[data-span="hero"] { grid-column: span 4;  grid-row: span 6; }
```

### 자주 쓰는 span
- `sm` — 1/3 폭, 3행 (Home 의 대부분 카드)
- `md` — 2/3 폭, 3행 (Home 의 Blog 카드)
- `xl` — 전체 폭, 3행 (긴 공지나 배너용)

### Card 호출 방법
```tsx
<Card span="sm" hoverable ariaLabel="github">{children}</Card>
<Card span="md" ariaLabel="blog">{children}</Card>
```

### Tablet (max 960px) 자동 축소
```css
@media (max-width: 960px) {
  .card[data-span="sm"]   { grid-column: span 4; }
  .card[data-span="md"]   { grid-column: span 4; }
  .card[data-span="lg"]   { grid-column: span 8; }
  .card[data-span="xl"]   { grid-column: span 8; }
  .card[data-span="tall"] { grid-column: span 4; }
  .card[data-span="wide"] { grid-column: span 8; }
  .card[data-span="hero"] { grid-column: span 4; }
}
```

### Mobile (max 640px) — 모든 카드 full width
```css
@media (max-width: 640px) {
  .card[data-span] { grid-column: span 4; grid-row: span 3; }
}
```

자세한 grid 구조는 [[layout-grid]].

---

## 4. Scoped Card 변형

`className` 추가로 특정 카드만 별도 스타일.

### `home-stack-card` — Home Stack 카드
```css
.home-stack-card .chip-row {
  gap: 6px;
  overflow: hidden;
  min-height: 0;
}
.home-stack-card .lang-chip {
  padding: 4px 10px;
}
```

### `home-github-card` — Home GitHub 카드
```css
.home-github-card .card-sub {
  display: none;   /* sub 숨김 (KPI 만 보이게) */
}
.home-github-card .card-body { flex: 0 1 auto; }
```

### `home-services-card` — Home Services 카드
```css
.home-services-card .card-body { flex: 0 1 auto; }
.home-services-card .card-list li:nth-child(n + 3) { display: none; }
```

### `project-card` — Projects 목록 카드
모든 Projects 카드 공통. 별도 스타일은 거의 없음 (기본 card 사용).

### `project-card--live` — Projects 카드 중 LIVE
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

### `profile-card` — Home ProfileCard
```css
.profile-card { padding: var(--space-4); gap: var(--space-3); }
.profile-body {
  flex: 1; min-height: 0;
  display: grid; grid-template-columns: auto 1fr;
  align-items: center; gap: var(--space-4);
}
.profile-frame {
  position: relative;
  width: 96px; height: 96px;
  display: flex; align-items: center; justify-content: center;
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  padding: 6px;
  background: radial-gradient(circle at 50% 45%, ...);
}
```

### `project-detail-card` — Project 상세 큰 카드
```css
.project-detail-card {
  display: grid;
  gap: var(--space-5);
  padding: var(--space-5);
  border: 1px solid var(--border);
  border-radius: var(--radius-lg);
  background: var(--bg-card);
  min-width: 0;
  overflow: hidden;
}
```

---

## 5. 새 scoped 카드 만들기

내가 만든 카드만 다른 스타일을 주고 싶다면.

### Home.tsx 에서 className 추가
```tsx
<Card span="sm" hoverable className="home-myspecial-card">
  ...
</Card>
```

### layout.css 끝에 룰 추가
```css
.home-myspecial-card {
  background: linear-gradient(135deg, #1a1a2e, #16213e);
}
.home-myspecial-card .card-title {
  color: #00d9ff;
}
```

---

## 6. Card 내부 요소

### `.card-head`
```css
.card-head {
  display: flex; align-items: center; justify-content: space-between;
  gap: var(--space-2);
  flex-shrink: 0;
}
```
- 좌: `<span className="label">/ github</span>`
- 우: `<StatusBadge>` / `<span className="chip">`

`.label` 클래스 스타일:
```css
.label {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  letter-spacing: var(--tracking-label);
  text-transform: uppercase;
  color: var(--text-muted);
}
```

### `.card-title`
```css
.card-title {
  font-size: var(--text-md);
  font-weight: 600;
  line-height: var(--lh-tight);
  min-width: 0; overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;     /* 1줄 ellipsis */
}
```
긴 제목을 2줄 wrap 하려면 `.blog-featured-title` 처럼 별도 override (line-clamp 패턴 — [[typography#6-한국어-descender-잘림]]).

### `.card-sub`
```css
.card-sub {
  font-size: var(--text-sm);
  color: var(--text-muted);
  line-height: var(--lh-snug);
}
```

### `.card-body`
```css
.card-body {
  flex: 1;   /* 남는 공간 채움 */
  display: flex; flex-direction: column;
  gap: var(--space-2);
  min-height: 0;
  overflow: hidden;
}
```

### `.card-list` (안에 들어가는 ul)
```css
.card-list {
  display: flex; flex-direction: column; gap: 6px;
  min-height: 0;
}
.card-list li {
  display: flex; align-items: center; justify-content: space-between;
  gap: var(--space-3);
  padding-block: 8px;
  border-bottom: 1px dashed var(--border);
  font-size: var(--text-sm);
  line-height: var(--lh-snug);
}
.card-list li:last-child { border-bottom: 0; }
```

### `.card-footer`
```css
.card-footer {
  display: flex; align-items: center; justify-content: space-between;
  gap: var(--space-2);
  margin-top: auto;          /* 카드 맨 아래로 push */
  padding-top: var(--space-2);
  flex-shrink: 0;
}
```

### `.card-link` (footer 의 액션)
```css
.card-link {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  letter-spacing: var(--tracking-label);
  text-transform: uppercase;
  color: var(--accent);
  display: inline-flex; align-items: center; gap: 6px;
  white-space: nowrap;
}
.card-link::after { content: "→"; transition: transform var(--t-fast); }
.card-link:hover::after { transform: translateX(2px); }
```

---

## 7. KPI (큰 숫자 그리드) — Home GitHub 카드

```css
.kpi-row {
  display: grid; grid-template-columns: repeat(3, 1fr);
  gap: var(--space-3) var(--space-4);
}
.kpi { display: flex; flex-direction: column; gap: 4px; }
.kpi-value {
  font-size: var(--text-xl);
  font-weight: 600;
  font-family: var(--font-mono);
  color: var(--text);
}
.kpi-label {
  font-size: var(--text-xs);
  letter-spacing: var(--tracking-label);
  text-transform: uppercase;
  color: var(--text-muted);
}
```

KPI 컬럼 수 변경:
```css
.kpi-row { grid-template-columns: repeat(4, 1fr); }   /* 3 → 4 */
```

---

## 8. Chip (작은 라벨)

```css
.chip {
  display: inline-flex; align-items: center;
  padding: 3px 8px;
  font-family: var(--font-mono); font-size: var(--text-xs);
  color: var(--text-muted);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  background: transparent;
  white-space: nowrap;
}
.chip-row { display: flex; gap: 6px; flex-wrap: wrap; }
```

### Lang chip (Stack 카드)
```css
.lang-chip { gap: 6px; }
.lang-dot {
  display: inline-block;
  width: 8px; height: 8px;
  border-radius: 999px;
  background: var(--text-muted);
  flex-shrink: 0;
  margin-right: 4px;
}
```

---

## 9. Bar (progress)

Skills 페이지의 언어 비율 bar.

```css
.bar {
  width: 100%;
  height: 4px;
  border-radius: 999px;
  background: var(--bg-pill);
  overflow: hidden;
}
.bar > span {
  display: block;
  height: 100%;
  background: var(--accent);
  border-radius: inherit;
}
```

### Bar 더 두껍게
```css
.bar { height: 6px; }
```

### Bar 색 변경
JSX 에서 직접:
```tsx
<div className="bar"><span style={{ width: "50%", background: "#a78bfa" }} /></div>
```

---

## 10. 잘림 방지 패턴

카드 안에 컨텐츠가 넘쳐서 잘리면:

1. `min-height: 0` — flex / grid 자식이 컨텐츠 크기로 늘어나려는 걸 막음
2. `overflow: hidden` — 넘치는 부분 잘라냄
3. `text-overflow: ellipsis` + `white-space: nowrap` — 한 줄 텍스트 …
4. `-webkit-line-clamp` — 여러 줄 …
5. `padding-bottom: 2px` — descender 보호 (한국어)

자세히 [[typography#6-한국어-descender-잘림]].

---

## 11. 변경 후 확인 워크플로

1. `npm run dev` → 카드가 들어있는 페이지 (Home 추천) 열기
2. 브라우저 검사도구 (Cmd+Option+I) 로 해당 카드 element 클릭 → 적용된 클래스 확인
3. 변경하고 싶은 부분의 클래스를 `layout.css` 에서 찾아 수정
4. 저장 시 hot-reload → 즉시 화면 갱신

---

연결: [[../hompage]] · [[tokens]] · [[typography]] · [[layout-grid]] · [[animations]]
