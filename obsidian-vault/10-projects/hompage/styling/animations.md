---
title: "hompage styling — 애니메이션 / 모션"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:25:00+09:00
tags: [project, hompage, styling, animations, motion]
home_hub: hompage
related:
  - "[[tokens]]"
  - "[[cards]]"
---

# 애니메이션 / 모션

**[[../hompage|↑ hompage hub]]**

> 페이지 전환 stagger, hover 효과, 트랜지션. 어느 부분이 어떻게 움직이는지 + 끄거나 더 화려하게.

---

## 1. 두 가지 모션 시스템

| 종류 | 동작 시점 | 정의 위치 |
| --- | --- | --- |
| **Page transition (stagger)** | 라우터 변경 시 (Home → Projects 등) | `Layout.tsx` + `layout.css` `@keyframes pageSlideUp` |
| **Hover / state transition** | 마우스 올림 / 클릭 / focus 시 | 각 컴포넌트 CSS `transition: ...` |

---

## 2. Page transition (slide-up + stagger)

### 작동 원리
1. URL 바뀌면 React Router 가 새 페이지 렌더
2. `Layout.tsx` 의 `<div key={pathname}>` 가 remount → CSS 애니메이션 재발사
3. `.page-anim > *` 의 직속 자식들이 80ms 간격으로 슬라이드 업
4. 그 안의 `.grid > *` / `.section-grid > *` (카드들) 이 다시 60ms 간격으로 stagger

### Layout.tsx 측
```tsx
const location = useLocation();
return (
  ...
  <div className="page-shell page-anim" key={location.pathname}>
    {children}
  </div>
  ...
);
```

### CSS 정의
```css
@keyframes pageSlideUp {
  from { opacity: 0; transform: translateY(14px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* Level 1 — page 의 직속 자식 */
.page-anim > * {
  animation: pageSlideUp 360ms cubic-bezier(0.22, 1, 0.36, 1) both;
  opacity: 0;
  will-change: transform, opacity;
}
.page-anim > *:nth-child(1) { animation-delay: 0ms; }
.page-anim > *:nth-child(2) { animation-delay: 80ms; }
.page-anim > *:nth-child(3) { animation-delay: 160ms; }
.page-anim > *:nth-child(4) { animation-delay: 240ms; }
.page-anim > *:nth-child(n+5) { animation-delay: 320ms; }

/* Level 2 — grid 안 카드들 */
.page-anim .grid > *,
.page-anim .section-grid > * {
  animation: pageSlideUp 320ms cubic-bezier(0.22, 1, 0.36, 1) both;
  opacity: 0;
}
.page-anim .grid > *:nth-child(1) { animation-delay: 200ms; }
.page-anim .grid > *:nth-child(2) { animation-delay: 260ms; }
.page-anim .grid > *:nth-child(3) { animation-delay: 320ms; }
...
.page-anim .grid > *:nth-child(10) { animation-delay: 740ms; }
.page-anim .grid > *:nth-child(n+11) { animation-delay: 800ms; }
```

### 커스터마이즈

#### 속도 빠르게
```css
.page-anim > * {
  animation: pageSlideUp 200ms cubic-bezier(0.22, 1, 0.36, 1) both;
}
.page-anim .grid > * {
  animation: pageSlideUp 200ms cubic-bezier(0.22, 1, 0.36, 1) both;
}
```

#### Stagger 간격 좁히기 (더 빠르게 다 등장)
delay 값들을 절반으로:
```css
.page-anim > *:nth-child(2) { animation-delay: 40ms; }
.page-anim > *:nth-child(3) { animation-delay: 80ms; }
...
.page-anim .grid > *:nth-child(2) { animation-delay: 130ms; }
.page-anim .grid > *:nth-child(3) { animation-delay: 160ms; }
...
```

#### Stagger 간격 넓히기 (더 천천히)
delay 값들을 2배로.

#### 방향을 위→아래로 (slide-down)
```css
@keyframes pageSlideUp {
  from { opacity: 0; transform: translateY(-14px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

#### 좌→우로 슬라이드
```css
@keyframes pageSlideUp {
  from { opacity: 0; transform: translateX(-14px); }
  to   { opacity: 1; transform: translateX(0); }
}
```

#### Scale 효과 추가
```css
@keyframes pageSlideUp {
  from { opacity: 0; transform: translateY(14px) scale(0.97); }
  to   { opacity: 1; transform: translateY(0) scale(1); }
}
```

#### 애니메이션 완전히 끄기
`layout.css` 에서 `.page-anim` 관련 룰 다 삭제, 또는:
```css
.page-anim > *,
.page-anim .grid > *,
.page-anim .section-grid > * {
  animation: none !important;
  opacity: 1 !important;
}
```

### Reduced motion 사용자 (시스템 설정으로 모션 줄임)
이미 적용됨:
```css
@media (prefers-reduced-motion: reduce) {
  .page-anim > *,
  .page-anim .grid > *,
  .page-anim .section-grid > * {
    animation: none;
    opacity: 1;
  }
}
```
손대지 않는 게 좋음 (접근성).

---

## 3. Hover transitions

### 카드 hover
```css
.card {
  transition: border-color var(--t-fast), background var(--t-fast);
}
.card:hover { border-color: var(--border-strong); }
.card[data-hoverable="true"]:hover { background: var(--bg-card-hover); }
```

### Hero CTA 버튼
```css
.hero-cta {
  transition: border-color var(--t-fast), background var(--t-fast), color var(--t-fast);
}
.hero-cta:hover {
  border-color: var(--border-accent);
  color: var(--accent);
  background: var(--bg-card-hover);
}
```

### 카드 링크 화살표 hover (→ 가 살짝 오른쪽으로)
```css
.card-link::after { content: "→"; transition: transform var(--t-fast); }
.card-link:hover::after { transform: translateX(2px); }
```

### 액션 버튼 hover 시 살짝 떠오르게
```css
.hero-cta {
  transition: transform var(--t-fast), border-color var(--t-fast);
}
.hero-cta:hover {
  transform: translateY(-1px);
}
```

### Filter tab hover
```css
.filter-tab {
  transition: color var(--t-fast), background var(--t-fast), border-color var(--t-fast);
}
.filter-tab:hover { color: var(--text); background: var(--bg-pill-active); }
```

---

## 4. Transition 토큰

`tokens.css`:
```css
--t-fast:  120ms ease;    ← hover / 작은 변화
--t-base:  180ms ease;    ← 카드 hover 강조 등 (현재 거의 미사용)
```

### 모든 hover 더 빠르게
```css
--t-fast: 80ms ease;
```

### 모든 hover 더 부드럽게
```css
--t-fast: 200ms cubic-bezier(0.22, 1, 0.36, 1);
```

---

## 5. 새 애니메이션 추가 예제

### 카드를 hover 시 살짝 들어올리기
`layout.css` 끝에:
```css
.card {
  transition: transform var(--t-fast), border-color var(--t-fast);
}
.card[data-hoverable="true"]:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.18);
}
```

### LIVE 배지에 pulse 효과
```css
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50%      { opacity: 0.5; }
}
.project-card--live .badge[data-status="ok"] {
  animation: pulse 2s ease-in-out infinite;
}
```

### Topbar 의 brand-dot 깜박임
```css
@keyframes blink {
  0%, 100% { opacity: 1; }
  50%      { opacity: 0.4; }
}
.brand-dot {
  animation: blink 1.5s ease-in-out infinite;
}
```

---

## 6. Skip link / Focus 가시화

키보드 사용자를 위한 focus ring.

이미 적용:
```css
.skip-link {
  position: absolute;
  top: -100px; left: 0;
  background: var(--accent);
  color: var(--text-inverse);
  padding: 8px 16px;
  transition: top var(--t-fast);
}
.skip-link:focus { top: 0; }
```

`:focus-visible` 도 글로벌:
```css
:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}
```

손대지 않는 게 좋음 (접근성).

---

## 7. 모션 절제 가이드

**좋은 모션**:
- 짧음 (≤ 300ms)
- 시작·종료가 명확
- 이유가 있음 (정보 위계 / 인과 관계 표현)

**나쁜 모션**:
- 1초 넘어가는 슬로우
- 반복 무한 (집중 방해)
- 의미 없는 회전 / 흔들기

`prefers-reduced-motion` 존중 — 시스템 설정에서 모션 줄임을 켠 사용자는 애니메이션을 안 봐도 사이트 사용 가능해야.

---

## 8. 변경 후 확인

```bash
npm run dev
```
- 라우터 이동 (Home → Projects → Skills) 마다 stagger 확인
- 마우스 hover 시 부드러움 확인
- macOS 시스템 설정 → 손쉬운 사용 → 동작 줄이기 켜고 다시 확인

---

연결: [[../hompage]] · [[tokens]] · [[cards]]
