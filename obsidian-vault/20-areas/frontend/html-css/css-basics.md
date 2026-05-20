---
title: "CSS 기초 — selector / 속성 / 박스 모델"
kind: knowledge
project: frontend
status: current
tags: [frontend, css, beginner, box-model, selector]
---

# CSS 기초

**[[html-css|↑ HTML/CSS hub]]**

---

## 1. CSS 구조

```css
selector {
  property: value;
  property: value;
}
```

예:
```css
h1 {
  color: blue;
  font-size: 32px;
}
```

해석: `h1` 요소를 → 색 파랑, 폰트 크기 32px 로.

---

## 2. CSS 어디에 쓰나

### 인라인 (한 줄에 직접) — 비추천
```html
<h1 style="color: red;">제목</h1>
```

### `<head>` 안에
```html
<head>
  <style>
    h1 { color: red; }
  </style>
</head>
```

### 외부 파일 (가장 권장)
```html
<head>
  <link rel="stylesheet" href="style.css">
</head>
```

`style.css`:
```css
h1 { color: red; }
```

---

## 3. Selector (어떤 요소 골라낼지)

### 태그 selector
```css
h1 { color: red; }      /* 모든 h1 */
p  { color: gray; }     /* 모든 p */
a  { text-decoration: none; }
```

### class selector — `.`
```html
<p class="warning">주의</p>
```
```css
.warning { color: orange; }
```
**가장 자주 씀**. 같은 클래스 여러 요소에 가능.

### id selector — `#`
```html
<div id="header">...</div>
```
```css
#header { background: black; }
```
페이지에 ID 는 1번만. id 도 가능하지만 보통 class 가 권장.

### 자손 selector (한 칸 띄움)
```css
.card h2 { color: blue; }   /* .card 안의 모든 h2 */
```

### 직접 자식 selector (`>`)
```css
.card > h2 { ... }          /* .card 의 바로 자식 h2 만 */
```

### 여러 selector 같은 스타일 (`,`)
```css
h1, h2, h3 { font-family: serif; }
```

### 가상 클래스 (`:`)
```css
a:hover     { color: red; }       /* 마우스 올림 */
a:visited   { color: purple; }    /* 방문한 적 있는 링크 */
a:focus     { outline: 2px solid blue; }   /* 포커스 (Tab 누름) */
li:first-child { font-weight: bold; }      /* 첫 자식 */
li:last-child  { border-bottom: 0; }       /* 마지막 자식 */
li:nth-child(2n) { background: #f0f0f0; }  /* 짝수번째 */
```

### 가상 요소 (`::`)
```css
p::first-letter { font-size: 32px; }   /* 첫 글자만 */
p::before { content: "→ "; }            /* 앞에 내용 추가 */
p::after  { content: "."; }             /* 뒤에 내용 추가 */
```

---

## 4. 속성 (자주 쓰는 것)

### 글자
```css
color: #4ade80;             /* 글자 색 */
font-size: 16px;            /* 글자 크기 */
font-weight: bold;          /* 굵기 — normal/bold/100~900 */
font-family: "Inter", sans-serif;  /* 폰트 */
font-style: italic;         /* 기울임 */
text-align: center;         /* 정렬 — left/center/right */
text-decoration: underline; /* 밑줄 — none/underline */
line-height: 1.5;           /* 줄 간격 */
letter-spacing: 0.05em;     /* 자간 */
text-transform: uppercase;  /* 대문자 — uppercase/lowercase/capitalize */
```

### 배경
```css
background-color: #f0f0f0;
background-image: url(bg.jpg);
background-size: cover;     /* 영역 채움 */
background-position: center;
background-repeat: no-repeat;
/* 짧게 한 줄: */
background: #f0f0f0 url(bg.jpg) center / cover no-repeat;
```

### 박스 (다음 §6 박스 모델)
```css
width: 200px;
height: 100px;
padding: 16px;       /* 내부 여백 */
margin: 24px;        /* 외부 여백 */
border: 1px solid black;
border-radius: 8px;  /* 모서리 둥글게 */
```

### 위치
```css
display: block;       /* 줄 차지 */
display: inline;      /* 한 줄 안 */
display: inline-block;
display: flex;        /* Flexbox */
display: grid;        /* Grid */
display: none;        /* 숨김 */

position: relative;   /* 본래 위치 기준 */
position: absolute;   /* 부모 기준 */
position: fixed;      /* 화면 기준 (스크롤 영향 X) */
position: sticky;     /* 스크롤 시 고정 */
top: 0;
left: 0;
z-index: 10;          /* 위/아래 쌓임 순서 */
```

### 효과
```css
opacity: 0.5;                          /* 투명도 0~1 */
box-shadow: 0 4px 12px rgba(0,0,0,0.2);
transition: all 0.3s ease;             /* 부드러운 변화 */
transform: translateY(-2px);           /* 이동/회전/크기 */
cursor: pointer;                       /* 마우스 모양 */
overflow: hidden;                      /* 넘침 처리 */
```

---

## 5. 값 단위

### 절대 단위
- `px` — 픽셀 (가장 자주 — `font-size: 16px`)
- `pt` — 포인트 (인쇄용)

### 상대 단위
- `em` — 부모 글자 크기 기준 (`padding: 1em` = 부모 font-size 와 같은 값)
- `rem` — 루트 (`<html>`) font-size 기준 (대부분 16px)
- `%` — 부모 크기 비율
- `vw` / `vh` — 뷰포트 너비 / 높이 (`100vw` = 화면 전체 너비)
- `vmin` / `vmax`

### 색상
```css
color: red;                       /* 이름 */
color: #ff0000;                   /* HEX */
color: #f00;                      /* HEX 단축 */
color: rgb(255, 0, 0);            /* RGB */
color: rgba(255, 0, 0, 0.5);      /* + alpha (투명도) */
color: hsl(0, 100%, 50%);         /* HSL */
color: hsla(0, 100%, 50%, 0.5);
```

### 함수
```css
width: calc(100% - 40px);         /* 계산 */
font-size: clamp(14px, 2vw, 18px); /* 최소/이상/최대 */
gap: min(20px, 5%);
```

---

## 6. 박스 모델 (핵심!)

모든 HTML 요소는 박스. 박스는 4 겹:

```
┌─ margin (외부 여백) ──────────────┐
│  ┌─ border (테두리) ───────────┐  │
│  │  ┌─ padding (내부 여백) ─┐  │  │
│  │  │  [ content (내용) ]   │  │  │
│  │  └────────────────────────┘  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### CSS 예
```css
.card {
  width: 200px;          /* content 너비 */
  padding: 16px;          /* 안쪽 여백 */
  border: 1px solid #ccc; /* 테두리 */
  margin: 24px;           /* 바깥 여백 */
}
```

**기본 동작**: 박스 전체 너비 = content + padding × 2 + border × 2 = 200 + 32 + 2 = 234px. 직관 X.

### box-sizing: border-box (반드시 추천)
```css
* {
  box-sizing: border-box;
}
```

→ `width: 200px` 이 padding 과 border 포함한 전체 = 직관적.

대부분 첫 줄에 `* { box-sizing: border-box; margin: 0; padding: 0; }` 같은 reset 작성.

### padding / margin 단축
```css
padding: 16px;                  /* 4 방향 같음 */
padding: 16px 24px;             /* top/bottom 16, left/right 24 */
padding: 16px 24px 8px;         /* top 16, l/r 24, bottom 8 */
padding: 16px 24px 8px 12px;    /* top, right, bottom, left (시계방향) */

padding-top: 16px;
padding-right: 24px;
padding-bottom: 8px;
padding-left: 12px;
```

`margin` 도 동일.

---

## 7. 우선순위 (Cascade)

같은 요소에 여러 규칙 → 우선순위로 결정.

### 점수
1. `style="..."` 인라인 — 1000점
2. `#id` — 100점
3. `.class` / `[attr]` / `:pseudo` — 10점
4. `tag` — 1점
5. 더 많은 점수가 이김

### 예
```css
p { color: red; }              /* 1점 */
.warning { color: orange; }    /* 10점 — 이김 */
#alert { color: black; }       /* 100점 — 이김 */
```

### `!important` — 최후의 무기
```css
p { color: red !important; }   /* 다 무시하고 빨강 */
```

남용 금지. 디버깅 시 강제로 우선 적용할 때만.

---

## 8. 상속 (Inheritance)

일부 속성은 부모 → 자식 자동 상속.

```html
<div style="color: blue;">
  <p>이 글자도 자동으로 파랑</p>
</div>
```

### 상속되는 것
- `color`, `font-family`, `font-size`, `line-height`, `text-align`

### 상속 안 되는 것
- `margin`, `padding`, `border`, `background`, `width`, `height`

### 강제 상속
```css
border: inherit;
```

---

## 9. 자주 만나는 패턴

### 카드
```html
<div class="card">
  <h3 class="card-title">제목</h3>
  <p class="card-body">내용</p>
</div>
```
```css
.card {
  background: white;
  border: 1px solid #e5e5e5;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}
.card-title {
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 8px;
}
.card-body {
  color: #666;
  line-height: 1.5;
}
```

### 버튼
```css
button {
  background: #4ade80;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 8px 16px;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.2s;
}
button:hover {
  background: #22c55e;
}
```

### 가운데 정렬 (가로)
```css
.center {
  margin: 0 auto;   /* 좌우 마진 자동 */
  width: 600px;     /* 너비가 정해져 있어야 */
}
```

### 가운데 정렬 (가로 + 세로) — Flexbox
```css
.center-both {
  display: flex;
  justify-content: center;   /* 가로 */
  align-items: center;       /* 세로 */
  height: 100vh;             /* 화면 전체 높이 */
}
```

자세히 [[layout/flexbox]].

---

## 10. 다음 단계

1. **[[colors-and-typography]]** — 색 / 폰트
2. **[[layout/flexbox]]** — Flexbox 1차원 레이아웃
3. **[[layout/grid]]** — Grid 2차원 레이아웃

---

연결: [[html-css|↑ hub]] · [[html-tags-essentials]] · [[colors-and-typography]]
