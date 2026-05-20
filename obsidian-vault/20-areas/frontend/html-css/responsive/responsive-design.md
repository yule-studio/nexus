---
title: "반응형 디자인 — 모바일 / 태블릿 / 데스크탑"
kind: knowledge
project: frontend
status: current
tags: [frontend, css, beginner, responsive, media-query]
---

# 반응형 디자인

**[[../html-css|↑ HTML/CSS hub]]**

> 같은 페이지가 폰 / 태블릿 / 데스크탑 에서 다 잘 보이게.

---

## 1. viewport meta (필수)

```html
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
```

이거 빠지면 모바일에서 사이트가 데스크탑 크기로 줄여 보임. **반드시 첫 줄**.

---

## 2. Media Query — 화면 크기 별 다른 스타일

```css
/* 기본 (모든 화면) */
.card { font-size: 16px; }

/* 화면 폭 ≤ 768px (태블릿 이하) */
@media (max-width: 768px) {
  .card { font-size: 14px; }
}

/* 화면 폭 ≤ 480px (모바일) */
@media (max-width: 480px) {
  .card { font-size: 12px; }
}
```

해석: 화면 크기 따라 다른 규칙 덮어쓰기.

### min-width 도 가능
```css
/* 데스크탑 (≥ 1024px) */
@media (min-width: 1024px) {
  .container { max-width: 1200px; }
}
```

### 결합
```css
@media (min-width: 641px) and (max-width: 960px) {
  /* 태블릿 범위 */
}
```

---

## 3. Breakpoint (분기점) 추천

### 단순 (mobile-first)
```css
/* 기본 = 모바일 */
.card { ... }

@media (min-width: 768px) {
  /* 태블릿 이상 */
}
@media (min-width: 1024px) {
  /* 데스크탑 */
}
```

### 단순 (desktop-first)
```css
/* 기본 = 데스크탑 */
.card { ... }

@media (max-width: 1024px) {
  /* 태블릿 이하 */
}
@media (max-width: 640px) {
  /* 모바일 */
}
```

### 어느 쪽을 쓸지
- **Mobile-first** (min-width) — 모던 추세. 모바일이 트래픽 다수.
- **Desktop-first** (max-width) — 기존 사이트 / 본인 hompage 패턴.

본인 hompage 는 desktop-first — `max-width: 960px`, `max-width: 640px`.

### 표준 break-point
| 폭 | 디바이스 |
| --- | --- |
| 0 ~ 480 | 작은 폰 |
| 481 ~ 640 | 큰 폰 |
| 641 ~ 960 | 태블릿 |
| 961 ~ 1280 | 노트북 |
| 1281+ | 큰 모니터 |

---

## 4. 반응형 패턴

### A. 칼럼 수 변경
```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);   /* 데스크탑: 3칸 */
}
@media (max-width: 960px) {
  .grid { grid-template-columns: repeat(2, 1fr); }   /* 태블릿: 2칸 */
}
@media (max-width: 640px) {
  .grid { grid-template-columns: 1fr; }              /* 모바일: 1칸 */
}
```

### B. 자동 wrap (media query 없이도)
```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
}
```

화면 폭 따라 자동으로 1~N 칸. media query 불필요. **권장**.

### C. 폰트 크기 줄이기
```css
h1 { font-size: 32px; }
@media (max-width: 640px) {
  h1 { font-size: 24px; }
}
```

또는 `clamp` 한 줄:
```css
h1 { font-size: clamp(24px, 5vw, 48px); }
```

### D. 모바일에서 요소 숨김
```css
@media (max-width: 640px) {
  .desktop-only { display: none; }
}
```

데스크탑에서만 보이는 요소.

### E. 모바일 전용 햄버거 메뉴
```css
.nav-desktop { display: flex; }
.nav-mobile  { display: none; }

@media (max-width: 768px) {
  .nav-desktop { display: none; }
  .nav-mobile  { display: block; }
}
```

### F. 직선 → 세로 변경
```css
.row {
  display: flex;
  gap: 16px;
}
@media (max-width: 640px) {
  .row { flex-direction: column; }
}
```

---

## 5. clamp() — 반응형 값 한 줄로

```css
font-size: clamp(14px, 2vw, 18px);
```

- 최소 14px
- 이상적으론 viewport 폭의 2%
- 최대 18px

### 적용 예
```css
h1 { font-size: clamp(28px, 5vw, 48px); }    /* 모바일 작게 ~ 데스크탑 크게 */
padding: clamp(16px, 4vw, 32px);              /* padding 반응형 */
gap: clamp(8px, 1.5vw, 24px);
```

`@media` 없이 부드럽게 반응. 모던.

---

## 6. vw / vh — viewport 단위

- `1vw` = 화면 폭의 1% (1920px 화면이면 19.2px)
- `1vh` = 화면 높이의 1%
- `100vh` = 화면 전체 높이

### 화면 가득 채우는 페이지
```css
.fullscreen {
  width: 100vw;
  height: 100vh;
}
```

### Hero 화면 절반
```css
.hero {
  height: 50vh;
}
```

---

## 7. container query (최신 — 일부만 지원)

부모 컨테이너 폭 기준 (viewport 가 아니라):
```css
.card-container {
  container-type: inline-size;
}

@container (max-width: 400px) {
  .card { font-size: 12px; }
}
```

복잡한 컴포넌트 라이브러리에서 유용. 처음엔 안 써도 됨.

---

## 8. 이미지 반응형

### 가장 단순
```css
img {
  max-width: 100%;
  height: auto;
}
```

→ 부모보다 작아질 수 있고 비율 유지.

### srcset (해상도 별 다른 사진)
```html
<img
  src="cat-400.jpg"
  srcset="cat-400.jpg 400w, cat-800.jpg 800w, cat-1600.jpg 1600w"
  sizes="(max-width: 640px) 100vw, 50vw"
  alt="고양이"
>
```

브라우저가 화면/네트워크 보고 최적 사진 자동 선택.

---

## 9. 본인 hompage 의 반응형

```css
/* layout.css */
.grid {
  grid-template-columns: repeat(12, minmax(0, 1fr));
}
@media (max-width: 960px) {
  .grid { grid-template-columns: repeat(8, minmax(0, 1fr)); }
  .card[data-span="sm"]   { grid-column: span 4; }
  .card[data-span="md"]   { grid-column: span 4; }
}
@media (max-width: 640px) {
  .grid { grid-template-columns: repeat(4, minmax(0, 1fr)); }
  .card[data-span] { grid-column: span 4; }   /* 모든 카드 full width */

  .topbar-status { display: none; }     /* 모바일에서 status 숨김 */
  .theme-toggle span { display: none; }  /* 글자 숨기고 아이콘만 */
}
```

---

## 10. 모바일 우선 사고

데스크탑 부터 만들고 모바일 적응 (큰 → 작은) vs 모바일 부터 만들고 데스크탑 확장 (작은 → 큰).

### Mobile-first 장점
- 모바일이 주 사용자 (현실)
- 모바일에서 잘 동작하면 데스크탑은 쉽게 확장
- CSS 더 깔끔 (max-width 보다 min-width 점진 증가가 자연스러움)

### 본인 hompage 는 desktop-first
이미 만들어진 사이트라 적용 어려움. 새로 시작하면 mobile-first 고려.

---

## 11. 디버깅 — 어떤 폭에서 깨지는지

### Chrome DevTools
1. F12 또는 Cmd+Option+I
2. Cmd+Shift+M (Toggle device toolbar)
3. 상단에서 iPhone / iPad / 커스텀 폭 선택
4. 폭 슬라이더 드래그하며 깨지는 지점 찾기

### 자주 깨지는 곳
- 카드 안 긴 영문 → `overflow-wrap: anywhere`
- 한국어 단어 어색하게 끊김 → `word-break: keep-all`
- 사진이 폭 넘침 → `max-width: 100%`
- table 이 가로 스크롤 → `overflow-x: auto`
- 가로 스크롤 발생 → 어딘가 `width: 100vw` 같은 게 padding 무시. `box-sizing: border-box` 확인

---

## 12. touch 디바이스 고려

```css
/* hover 효과는 마우스에만 */
@media (hover: hover) {
  .button:hover { background: #4ade80; }
}

/* 터치 대상은 최소 44×44 px 권장 */
button {
  min-width: 44px;
  min-height: 44px;
}
```

---

## 13. 다음 단계

1. **[[../forms/forms-basics]]** — 폼 / 입력
2. **[[../semantic-and-accessibility]]** — 의미 / 접근성

---

연결: [[../html-css|↑ hub]] · [[../layout/flexbox]] · [[../layout/grid]]
