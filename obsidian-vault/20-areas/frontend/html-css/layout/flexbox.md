---
title: "Flexbox — 1차원 레이아웃"
kind: knowledge
project: frontend
status: current
tags: [frontend, css, beginner, flexbox, layout]
---

# Flexbox

**[[../html-css|↑ HTML/CSS hub]]**

> 가장 자주 쓰는 레이아웃 도구. 가로 또는 세로 1 방향으로 요소 정렬.

---

## 1. 기본 — flex 만 켜기

```html
<div class="container">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
```

```css
.container {
  display: flex;
}
```

→ A B C 가 가로로 나란히.

### display: flex 안 했을 때 (기본)
A, B, C 가 각각 한 줄씩 (div 는 block) — 세로 쌓임.

### display: flex 했을 때
A B C 가 한 줄에 — flex item 으로 변신.

---

## 2. 방향 — flex-direction

```css
.container {
  display: flex;
  flex-direction: row;            /* 기본 — 가로 (왼→오) */
  flex-direction: row-reverse;    /* 가로 반대 */
  flex-direction: column;         /* 세로 (위→아래) */
  flex-direction: column-reverse; /* 세로 반대 */
}
```

---

## 3. 가로 정렬 — justify-content

`flex-direction: row` 일 때 좌우 정렬.

```css
.container {
  display: flex;
  justify-content: flex-start;     /* 왼쪽 (기본) */
  justify-content: flex-end;       /* 오른쪽 */
  justify-content: center;         /* 가운데 */
  justify-content: space-between;  /* 양 끝 + 사이 균등 */
  justify-content: space-around;   /* 사이 + 양 옆 균등 */
  justify-content: space-evenly;   /* 모든 간격 같음 */
}
```

### 시각화
```
flex-start:     [A][B][C]------
flex-end:       ------[A][B][C]
center:         ---[A][B][C]---
space-between:  [A]----[B]----[C]
space-around:   -[A]--[B]--[C]-
space-evenly:   --[A]--[B]--[C]--
```

---

## 4. 세로 정렬 — align-items

`flex-direction: row` 일 때 위아래 정렬.

```css
.container {
  display: flex;
  align-items: stretch;       /* 기본 — 세로 채움 */
  align-items: flex-start;    /* 위쪽 */
  align-items: flex-end;      /* 아래쪽 */
  align-items: center;        /* 가운데 */
  align-items: baseline;      /* 글자 baseline 맞춤 */
}
```

---

## 5. 가장 자주 — 가운데 정렬

```css
.center-everything {
  display: flex;
  justify-content: center;    /* 가로 가운데 */
  align-items: center;        /* 세로 가운데 */
  height: 100vh;              /* 화면 전체 높이 */
}
```

→ 안의 요소가 화면 정중앙.

---

## 6. 사이 간격 — gap

```css
.container {
  display: flex;
  gap: 12px;            /* 모든 item 사이 12px */
  gap: 12px 24px;       /* row gap / column gap */
  row-gap: 12px;
  column-gap: 24px;
}
```

`margin` 대신 `gap` 권장 — 더 깔끔.

---

## 7. 줄바꿈 — flex-wrap

```css
.container {
  display: flex;
  flex-wrap: nowrap;       /* 기본 — 한 줄 강제, 넘치면 줄어듦 */
  flex-wrap: wrap;         /* 넘치면 다음 줄로 */
  flex-wrap: wrap-reverse;
}
```

### 칩 / 태그 그룹 패턴
```css
.chip-row {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
```

---

## 8. Item 측 속성

위는 다 **container** 측. 각 item 은 다른 속성:

### flex-grow — 남는 공간 분배
```css
.item {
  flex-grow: 0;     /* 기본 — 안 늘어남 */
  flex-grow: 1;     /* 남는 공간 다 차지 */
}
```

### 예 — 가운데가 자동 늘어남
```css
.left   { flex-grow: 0; }
.center { flex-grow: 1; }   /* 늘어남 */
.right  { flex-grow: 0; }
```

### flex-shrink — 공간 부족 시 줄어듦
```css
.item {
  flex-shrink: 1;    /* 기본 — 줄어듦 */
  flex-shrink: 0;    /* 줄어들지 않음 (원본 크기 유지) */
}
```

### flex-basis — 기본 크기
```css
.item {
  flex-basis: auto;   /* 기본 */
  flex-basis: 100px;
}
```

### 단축
```css
.item {
  flex: 1;               /* = flex-grow: 1; flex-shrink: 1; flex-basis: 0 */
  flex: 0 0 200px;       /* = grow: 0, shrink: 0, basis: 200px (고정 200) */
  flex: 1 0 auto;
}
```

### align-self — 개별 item 의 세로 정렬
```css
.item {
  align-self: center;    /* 이 item 만 가운데 */
}
```

---

## 9. 자주 만드는 레이아웃

### A. Header — 로고 좌, 메뉴 우
```css
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
}
```

```html
<header class="header">
  <h1>로고</h1>
  <nav>
    <a href="/">Home</a>
    <a href="/about">About</a>
  </nav>
</header>
```

### B. 카드 안 — 좌측 아이콘 + 우측 텍스트
```css
.row {
  display: flex;
  gap: 12px;
  align-items: center;
}
.icon { flex-shrink: 0; width: 32px; }
.text { flex: 1; }
```

### C. 사이드바 + 본문
```css
.layout {
  display: flex;
}
.sidebar { flex: 0 0 240px; }   /* 240px 고정 */
.main    { flex: 1; }            /* 나머지 */
```

### D. 가운데 정렬 (가로 + 세로)
```css
.page {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}
```

### E. Sticky footer (footer 가 항상 맨 아래)
```css
body {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}
main { flex: 1; }   /* 본문이 남는 공간 다 차지 */
footer { /* 자동으로 맨 아래 */ }
```

### F. 같은 너비 칸 3 개
```css
.row {
  display: flex;
  gap: 16px;
}
.col {
  flex: 1;          /* 균등 분배 */
}
```

---

## 10. flex 안 flex (중첩)

```css
.outer {
  display: flex;          /* 가로 */
}
.inner {
  display: flex;
  flex-direction: column; /* 안은 세로 */
}
```

자주 씀. 카드 안 컨텐츠 세로 정렬할 때 등.

---

## 11. 함정

| 증상 | 원인 |
| --- | --- |
| 한 줄에 다 안 들어감 | `flex-wrap: wrap` 안 함 |
| item 가 너무 줄어듦 | `flex-shrink: 0` 으로 줄어듦 막기 |
| 세로 정렬 안 됨 | container 에 height 가 없음 (item 만 정렬할 공간 없음) |
| 가운데 정렬 안 됨 | `align-items: center` 빠뜨림 |
| gap 이 안 먹음 | 옛 브라우저 (Safari < 14.1). gap 대신 margin 으로 |

---

## 12. 연습 — Flexbox Froggy

https://flexboxfroggy.com — 24 단계 게임으로 Flexbox 마스터.

---

## 13. 다음 단계

1. **[[grid]]** — Grid 2차원 레이아웃
2. **[[positioning]]** — position / z-index

---

연결: [[../html-css|↑ hub]] · [[grid]] · [[positioning]]
