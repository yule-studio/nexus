---
title: "CSS Grid — 2차원 레이아웃"
kind: knowledge
project: frontend
status: current
tags: [frontend, css, beginner, grid, layout]
---

# CSS Grid

**[[../html-css|↑ HTML/CSS hub]]**

> 가로 + 세로 2 차원. Flexbox 는 1 줄, Grid 는 표 같은 격자.

---

## 1. 기본 — 칸 나누기

```html
<div class="grid">
  <div>A</div>
  <div>B</div>
  <div>C</div>
  <div>D</div>
</div>
```

```css
.grid {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;   /* 3 칸 균등 */
}
```

→ A B C / D 가 3 칼럼에 자동 배치.

### `fr` 단위
"남는 공간의 비율". `1fr 1fr 1fr` = 3 등분.

### `1fr 2fr 1fr`
```css
grid-template-columns: 1fr 2fr 1fr;
```
→ 왼쪽 1, 가운데 2, 오른쪽 1 비율.

### 픽셀 + fr 섞기
```css
grid-template-columns: 240px 1fr;   /* 사이드바 240, 본문 나머지 */
```

---

## 2. repeat — 반복

```css
grid-template-columns: repeat(3, 1fr);              /* 1fr 3 번 */
grid-template-columns: repeat(12, minmax(0, 1fr));  /* 12 칸 */
grid-template-columns: repeat(4, 200px);
```

### 자동 채우기
```css
grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
```

해석: 최소 280px 폭의 칸들로 자동 채움. 화면 폭에 따라 칸 수 변동.

- 화면 1200px → 4 칸 (280×4 = 1120 + gap)
- 화면 800px → 2 칸
- 화면 400px → 1 칸

본인 hompage 의 `.section-grid` 패턴.

---

## 3. 행 (row)

```css
grid-template-rows: 100px 200px 100px;     /* 3 행 */
grid-template-rows: repeat(3, 1fr);
grid-auto-rows: 100px;                      /* 자동으로 만들어지는 행 높이 */
```

### grid-auto-rows
명시하지 않은 행이 생기면 이 높이로:
```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: 100px;     /* 모든 행 100px */
  gap: 8px;
}
```

---

## 4. 간격 — gap

```css
gap: 12px;             /* 행 / 열 같음 */
gap: 12px 24px;        /* 행 12 / 열 24 */
row-gap: 12px;
column-gap: 24px;
```

---

## 5. 칸 차지 — grid-column / grid-row

각 item 에 적용.

### grid-column
```css
.item {
  grid-column: span 2;       /* 2 칸 차지 */
  grid-column: 1 / 3;        /* 1번째 line 부터 3번째 line 까지 (= 2칸) */
  grid-column: 1 / -1;       /* 1번 부터 끝까지 (full width) */
}
```

### grid-row
```css
.item {
  grid-row: span 2;          /* 2 행 차지 */
  grid-row: 1 / 3;
}
```

### 예 — 첫 카드만 2칸, 2행
```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: 100px;
  gap: 8px;
}
.card:first-child {
  grid-column: span 2;
  grid-row: span 2;
}
```

---

## 6. 본인 hompage 의 dashboard grid

```css
.grid {
  display: grid;
  grid-template-columns: repeat(12, minmax(0, 1fr));
  grid-auto-flow: dense;
  grid-auto-rows: 100px;
  gap: 2px;
}
.card[data-span="sm"]   { grid-column: span 4;  grid-row: span 3; }
.card[data-span="md"]   { grid-column: span 8;  grid-row: span 3; }
.card[data-span="xl"]   { grid-column: span 12; grid-row: span 3; }
```

- 12 칼럼 grid
- 100px 행 높이
- sm 카드 = 4 칼럼 × 3 행 = 1/3 폭, 300px 높이
- md = 8 칼럼 = 2/3 폭
- xl = 12 = full width

### grid-auto-flow: dense
빈 칸이 있으면 뒤 item 이 그 자리 채움 (대시보드 빽빽하게).

### minmax(0, 1fr)
"최소 0, 최대 1fr" — 칼럼이 컨텐츠 크기로 늘어나는 걸 막음. 글자 잘 잘리게 보장.

---

## 7. 명명 영역 (named areas)

복잡한 레이아웃 시각화.

```css
.layout {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-rows: 60px 1fr 40px;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  gap: 8px;
  height: 100vh;
}
.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.footer  { grid-area: footer; }
```

```html
<div class="layout">
  <header class="header">상단</header>
  <aside class="sidebar">사이드</aside>
  <main class="main">본문</main>
  <footer class="footer">하단</footer>
</div>
```

---

## 8. justify / align

### container 측 (item 들 전체 정렬)
```css
.grid {
  display: grid;
  justify-items: stretch;    /* 기본 — 가로 채움 */
  justify-items: start | center | end;

  align-items: stretch;      /* 기본 — 세로 채움 */
  align-items: start | center | end;

  justify-content: start;    /* grid 전체의 가로 위치 */
  align-content: start;      /* grid 전체의 세로 위치 */
}
```

### item 측 (개별 item)
```css
.item {
  justify-self: center;
  align-self: center;
}
```

---

## 9. 자주 만드는 레이아웃

### A. 카드 4 개 균등
```css
.row {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
}
```

### B. 갤러리 (반응형 자동)
```css
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 12px;
}
```

### C. 메인 + 사이드
```css
.page {
  display: grid;
  grid-template-columns: 1fr 280px;
  gap: 24px;
}
```

### D. 12-col 디자인 시스템
```css
.row {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: 16px;
}
.col-6 { grid-column: span 6; }   /* 절반 */
.col-4 { grid-column: span 4; }   /* 1/3 */
.col-12 { grid-column: span 12; } /* 전체 */
```

---

## 10. Grid vs Flexbox

| 케이스 | 추천 |
| --- | --- |
| 한 줄로 정렬 (header, nav) | Flexbox |
| 카드 균등 격자 | Grid |
| 동적 폭 (큰 + 작은 카드) | Grid |
| 알 수 없는 개수 자동 wrap | Flexbox (`flex-wrap`) 또는 Grid (`auto-fill`) |
| 복잡한 페이지 레이아웃 (사이드 + 본문 + 푸터) | Grid (named areas) |

---

## 11. 함정

| 증상 | 원인 |
| --- | --- |
| 칸이 컨텐츠 따라 늘어남 | `minmax(0, 1fr)` 사용 (1fr 만으론 부족) |
| 갤러리가 1 칼럼 됨 | `auto-fill, minmax(X, 1fr)` 에서 X 가 너무 큼 |
| 빈칸이 생김 | `grid-auto-flow: dense` 로 채우거나, item 순서 조정 |
| grid 안 item 이 너무 큼 | item 에 `min-width: 0` 추가 |

---

## 12. 연습

https://cssgridgarden.com — Grid Garden 게임으로 학습.

---

## 13. 다음 단계

1. **[[positioning]]** — position / z-index
2. **[[../responsive/responsive-design]]** — 반응형

---

연결: [[../html-css|↑ hub]] · [[flexbox]] · [[positioning]]
