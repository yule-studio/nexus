---
title: "hompage styling — 레이아웃 / 그리드 시스템"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:20:00+09:00
tags: [project, hompage, styling, layout, grid]
home_hub: hompage
related:
  - "[[tokens]]"
  - "[[cards]]"
  - "[[responsive]]"
---

# 레이아웃 / 그리드 시스템

**[[../hompage|↑ hompage hub]]**

> Page → Container → Grid → Card 4단 위계. 폭/간격/카드 배치를 어디서 조정하는지.

---

## 1. 큰 구조

```
<body>
 └ <Topbar>                                  ← sticky 상단 (56px)
 └ <main class="page">
   └ <div class="container">                 ← 최대 1180px 가운데 정렬
     └ <div class="page-shell page-anim">    ← 720px 좁은 페이지용
        └ {children}                          ← Home / Projects ...

레이어
[bg dot grid] ← body 배경
[page padding 위/아래]
[container padding 좌/우 16~32px]
[page-shell 가운데 720px]
[grid 12 col]
[card sm 4col / md 8col / xl 12col]
```

---

## 2. Container

```css
.container {
  width: 100%;
  max-width: var(--container-max);   /* 1180px */
  margin: 0 auto;
  padding-inline: var(--container-pad);   /* clamp(16px, 4vw, 32px) */
}
```

### 사이트 폭 늘리기
`tokens.css`:
```css
--container-max: 1280px;
```

### 사이트 폭 좁히기
```css
--container-max: 960px;
```

### 좌우 패딩 항상 고정
```css
--container-pad: 24px;
```

---

## 3. Page-shell (가운데 좁은 영역)

`Layout.tsx` 의 wrapper. Home / Projects / Skills 등 모든 페이지 컨텐츠가 이 안에 들어감.

```css
.page-shell {
  width: min(100%, 720px);
  margin-inline: auto;
}
```

### 더 넓은 페이지 (예: Projects)
```css
.page-shell { width: min(100%, 960px); }
```

### Home 만 따로
Home 은 `home-shell` 로 별도 wrapper:
```css
.home-shell {
  width: min(100%, 720px);
  margin-inline: auto;
}
```

Home 만 더 넓게:
```css
.home-shell { width: min(100%, 1080px); }
```

---

## 4. Page padding (위/아래)

```css
.page { padding-block: var(--space-7) var(--space-9); }
/* 32px top, 56px bottom */
```

### 더 여유로
```css
.page { padding-block: var(--space-9) var(--space-10); }
```

---

## 5. Grid (12-column dashboard)

Home dashboard 의 카드 배열.

```css
.grid {
  display: grid;
  grid-template-columns: repeat(12, minmax(0, 1fr));
  grid-auto-flow: dense;     /* 빈칸 자동 채움 */
  grid-auto-rows: 100px;     /* 한 행 100px */
  gap: 2px;                  /* 카드 사이 간격 */
}
```

### Home 만 grid 다른 설정
home-shell 안:
```css
.home-shell .grid {
  grid-auto-rows: 68px;
  gap: 6px;
}
```
→ Home 은 행 높이 68px, gap 6px (더 컴팩트).

### 카드 사이 간격 조정
```css
.grid { gap: 8px; }   /* 2 → 8px (현재) */
.grid { gap: 16px; }  /* 더 여유 */
```

### 행 높이 조정 — 모든 카드 더 짧게
```css
.grid { grid-auto-rows: 80px; }
```

### Column 수 변경 (12 → 6)
```css
.grid { grid-template-columns: repeat(6, minmax(0, 1fr)); }
```
하지만 card span (`sm: span 4` 등) 도 같이 수정해야 의미 있음.

---

## 6. Section-grid (Projects / Skills / Certs 등)

목록 페이지의 카드 배열. 12col 이 아니라 auto-fill 형식.

```css
.section-grid {
  display: grid;
  gap: 2px;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
}
```

### 카드 최소 폭 조정
```css
.section-grid {
  grid-template-columns: repeat(auto-fill, minmax(360px, 1fr));
}
```
→ 카드가 더 커야 한 줄에 들어옴 = 한 줄당 카드 수 감소.

### 카드 간격
```css
.section-grid { gap: 12px; }
```

---

## 7. Page-header

각 페이지의 상단 (페이지 제목 + 서브 + 우측 chip).

```css
.page-header {
  display: grid;
  grid-template-columns: 1fr auto;
  align-items: end;
  gap: var(--space-4);
  margin-bottom: var(--space-6);
  padding-bottom: var(--space-4);
  border-bottom: 1px dashed var(--border);
}
```

### page-header 글씨 크기
```css
.page-title { font-size: var(--text-2xl); font-weight: 600; }
.page-subtitle { font-size: var(--text-sm); color: var(--text-muted); }
.page-eyebrow { font-size: var(--text-xs); color: var(--accent); }
```

### page-header divider 제거
```css
.page-header { border-bottom: 0; padding-bottom: 0; }
```

---

## 8. Hero (Home 의 자기소개)

```css
.hero {
  display: grid;
  gap: var(--space-3);
  padding-block: var(--space-6) var(--space-6);
  border-bottom: 1px dashed var(--border);
  margin-bottom: var(--space-6);
}
.hero h1 {
  font-size: clamp(34px, 6vw, 56px);
  font-weight: 700;
  letter-spacing: -0.02em;
  line-height: 1.1;
}
```

### Home 의 hero 만 별도
```css
.home-shell .hero {
  padding-block: var(--space-5) var(--space-4);
  margin-bottom: var(--space-3);
}
.home-shell .hero h1 {
  font-size: clamp(36px, 7vw, 48px);
}
```

### hero divider 제거
```css
.hero { border-bottom: 0; padding-bottom: 0; }
```

---

## 9. Nav (목차 pill)

```css
.nav {
  display: flex; justify-content: center;
  padding-block: var(--space-3);
  margin-block: var(--space-3) var(--space-3);
  position: sticky;
  top: var(--topbar-h);
  z-index: 20;
  background: color-mix(in srgb, var(--bg) 92%, transparent);
  backdrop-filter: blur(6px);
  border-radius: var(--radius-pill);
}
```

### Home 의 nav 는 좌정렬 + 단순화
```css
.home-shell .nav {
  justify-content: flex-start;
  padding-block: var(--space-2);
  margin-block: var(--space-2);
  background: transparent;
  backdrop-filter: none;
  border-radius: 0;
}
```

### nav pill 자체
```css
.nav-pill {
  display: inline-flex; gap: 2px; padding: 4px;
  background: var(--bg-pill);
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  max-width: 100%; overflow-x: auto;
  scrollbar-width: none;
}
.nav-item {
  flex: 0 0 auto;
  display: inline-flex; align-items: center; gap: 6px;
  padding: 6px 12px;
  border-radius: var(--radius-pill);
  font-family: var(--font-mono); font-size: var(--text-sm);
  color: var(--text-muted);
  letter-spacing: 0.01em;
  white-space: nowrap;
  transition: background var(--t-fast), color var(--t-fast);
}
.nav-item:hover { color: var(--text); background: var(--bg-pill-active); }
.nav-item[aria-current="page"] {
  color: var(--text);
  background: var(--bg-card);
  border: 1px solid var(--border-accent);
  padding: 5px 11px;
}
```

### nav 항목 폰트 더 크게
```css
.nav-item { font-size: var(--text-md); padding: 8px 16px; }
```

### nav 가 상단 고정 안 되게
```css
.nav { position: static; }
```

---

## 10. Topbar

```css
.topbar {
  position: sticky; top: 0; z-index: 30;
  background: color-mix(in srgb, var(--bg) 88%, transparent);
  backdrop-filter: blur(8px);
  border-bottom: 1px solid var(--border);
}
.topbar-inner {
  display: flex; align-items: center;
  gap: var(--space-3);
  height: var(--topbar-h);
}
```

### Topbar 높이 변경
`tokens.css`:
```css
--topbar-h: 64px;
```

### 상단 고정 해제 (스크롤 시 같이 올라감)
```css
.topbar { position: static; }
```

---

## 11. Footer

```css
.site-footer {
  border-top: 1px solid var(--border);
  padding-block: var(--space-5);
  margin-top: var(--space-9);
}
.site-footer-inner {
  display: flex; justify-content: space-between;
  gap: var(--space-4);
  flex-wrap: wrap;
  font-size: var(--text-xs);
  color: var(--text-faint);
}
```

---

## 12. 새 카드 행 추가 (Home dashboard)

Home.tsx 의 `<div className="grid">` 안에 `<Card>` 더 넣기.

```tsx
<div className="grid">
  <ProfileCard />
  <Card span="sm">...</Card>
  <Card span="sm">...</Card>
  ...
  {/* 새 카드 */}
  <Card span="sm" hoverable ariaLabel="my-new-card">
    <div className="card-head">
      <span className="label">/ my new</span>
    </div>
    <h3 className="card-title">My Card</h3>
    <p className="card-sub">설명.</p>
  </Card>
</div>
```

→ 자동으로 다음 빈자리 (4col × 3row) 에 배치 (grid-auto-flow: dense).

---

## 13. dot-grid 배경 (장식점)

```css
body {
  background:
    radial-gradient(var(--dot-color) var(--dot-size), transparent var(--dot-size))
      0 0 / var(--dot-spacing) var(--dot-spacing),
    var(--bg);
  ...
}
```

### dot 안 보이게
```css
--dot-color: transparent;
```

### dot 더 자주
```css
--dot-spacing: 16px;
```

### dot 더 진하게
```css
--dot-color: rgba(255, 255, 255, 0.08);
```

---

## 14. 반응형 — 화면 폭별

자세히 [[responsive]]. 요약:

| 폭 | grid | 효과 |
| --- | --- | --- |
| ≥ 960px | 12-col | 데스크탑, span sm 4col |
| 641 ~ 960 | 8-col | 태블릿, span 자동 축소 |
| ≤ 640 | 4-col | 모바일, 모든 카드 full width |

---

연결: [[../hompage]] · [[tokens]] · [[cards]] · [[responsive]]
