---
title: "CSS Animations / Transitions"
kind: knowledge
project: frontend
status: current
tags: [frontend, css, animations, transitions]
---

# CSS Animations / Transitions

**[[../html-css|↑ HTML/CSS hub]]**

> 부드러운 모션. JS 없이도 풍부한 인터랙션.

---

## 1. transition — 상태 변화

```css
.btn {
  background: blue;
  transition: background 0.3s ease;
}
.btn:hover {
  background: darkblue;
}
```

→ hover 시 부드럽게 색 변화.

### syntax
```css
transition: property duration timing-function delay;
transition: background 0.3s ease 0.1s;
```

### 여러 속성
```css
transition:
  background 0.3s,
  transform 0.2s,
  opacity 0.15s;

/* 또는 all */
transition: all 0.3s ease;
```

---

## 2. timing-function (easing)

```css
transition-timing-function: linear;
transition-timing-function: ease;          /* 기본 */
transition-timing-function: ease-in;
transition-timing-function: ease-out;       /* 권장 (hover 등) */
transition-timing-function: ease-in-out;

/* custom — cubic-bezier */
transition-timing-function: cubic-bezier(0.22, 1, 0.36, 1);    /* ease-out-quart */
```

### cubic-bezier 시각화
https://cubic-bezier.com — 곡선 그리며 값 얻기.

### 자주 쓰는 값
```css
/* Material Design */
--ease-standard: cubic-bezier(0.4, 0, 0.2, 1);

/* Apple */
--ease-out: cubic-bezier(0.22, 1, 0.36, 1);
--ease-in-out: cubic-bezier(0.65, 0, 0.35, 1);
```

---

## 3. transition 가능한 속성

### Animatable
- color, background, border-color
- opacity
- transform (translate / scale / rotate / skew)
- width, height (성능 X)
- padding, margin (성능 X)
- box-shadow

### Animatable 아님
- display
- font-family
- 일부 layout 속성

→ display: none → block 변화는 부드럽지 않음. 대신 opacity + visibility 조합.

---

## 4. transform — 가장 자주

GPU 가속. 성능 좋음.

### translate (이동)
```css
.box:hover {
  transform: translateX(10px);
  transform: translateY(-5px);
  transform: translate(10px, -5px);
  transform: translate3d(10px, -5px, 0);    /* GPU layer 강제 */
}
```

### scale (크기)
```css
.box:hover {
  transform: scale(1.05);
  transform: scaleX(2);
}
```

### rotate (회전)
```css
.box:hover {
  transform: rotate(45deg);
  transform: rotate(0.25turn);    /* 90도 */
}
```

### 결합
```css
transform: translate(10px, 0) scale(1.1) rotate(5deg);
```

순서 중요 — 오른쪽 → 왼쪽 적용.

---

## 5. @keyframes — 복잡한 애니메이션

```css
@keyframes pulse {
  0%   { transform: scale(1); }
  50%  { transform: scale(1.1); }
  100% { transform: scale(1); }
}

.heart {
  animation: pulse 1s ease-in-out infinite;
}
```

### syntax
```css
animation: name duration timing-function delay iteration-count direction fill-mode play-state;
```

### 자주 쓰는 옵션
```css
.x {
  animation-name: slideIn;
  animation-duration: 0.5s;
  animation-timing-function: ease-out;
  animation-delay: 0.2s;
  animation-iteration-count: 1;            /* 1 / 3 / infinite */
  animation-direction: normal;             /* normal / reverse / alternate */
  animation-fill-mode: both;               /* none / forwards / backwards / both */
  animation-play-state: running;           /* paused */
}

/* 단축 */
animation: slideIn 0.5s ease-out 0.2s 1 both;
```

### fill-mode 의미
- `none` (기본) — 끝나면 원래대로
- `forwards` — 마지막 frame 유지
- `backwards` — 첫 frame 즉시 적용 (delay 중에도)
- `both` — backwards + forwards

대부분 `both` 권장.

---

## 6. 자주 만드는 keyframe

### fadeIn
```css
@keyframes fadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}
.fade-in { animation: fadeIn 0.3s ease-out; }
```

### slideUp
```css
@keyframes slideUp {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}
.slide-up { animation: slideUp 0.4s ease-out both; }
```

본인 hompage 의 page transition 패턴.

### shimmer (스켈레톤)
```css
@keyframes shimmer {
  0%   { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
.skeleton {
  background: linear-gradient(90deg, #eee 0%, #f5f5f5 50%, #eee 100%);
  background-size: 200% 100%;
  animation: shimmer 1.5s linear infinite;
}
```

### bounce
```css
@keyframes bounce {
  0%, 20%, 50%, 80%, 100% { transform: translateY(0); }
  40% { transform: translateY(-20px); }
  60% { transform: translateY(-10px); }
}
```

### spin (loader)
```css
@keyframes spin {
  to { transform: rotate(360deg); }
}
.spinner { animation: spin 1s linear infinite; }
```

---

## 7. 진입 / 진출 (enter / exit)

CSS 만으론 어려움 — display:none ↔ block 안 됨.

### opacity + pointer-events 트릭
```css
.modal {
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.2s;
}
.modal.open {
  opacity: 1;
  pointer-events: auto;
}
```

### `display: none` → 부드럽게 (실험적)
```css
.modal {
  transition: display 0.3s allow-discrete, opacity 0.3s;
  opacity: 0;
  display: none;
}
.modal.open {
  opacity: 1;
  display: block;
}

@starting-style {
  .modal.open { opacity: 0; }
}
```

브라우저 신문법. 안 되면 framer-motion 등 라이브러리.

---

## 8. View Transitions API (최신)

페이지 / element 간 부드러운 전환 (Chrome / Safari).

### 같은 페이지
```javascript
document.startViewTransition(() => {
  element.style.background = "red";
});
```

```css
::view-transition-old(root) {
  animation: fadeOut 0.3s;
}
::view-transition-new(root) {
  animation: fadeIn 0.3s;
}
```

### Next.js (실험적)
View Transitions 사용해서 페이지 전환 부드럽게.

---

## 9. prefers-reduced-motion

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation: none !important;
    transition: none !important;
  }
}
```

OS 설정으로 모션 줄임 켠 사용자 존중. 접근성 핵심.

---

## 10. 성능 — 어떤 속성이 빠른가

### Fast (GPU 가속)
- `transform`
- `opacity`
- `filter` (일부)

### Slow (layout 다시 계산)
- `width`, `height`
- `top`, `left`, `right`, `bottom`
- `padding`, `margin`
- `border-width`

### 좋은 vs 안 좋은
```css
/* ❌ 느림 (layout) */
.box { transition: left 0.3s; }
.box.move { left: 100px; }

/* ✅ 빠름 (transform) */
.box { transition: transform 0.3s; }
.box.move { transform: translateX(100px); }
```

---

## 11. will-change — 미리 GPU layer

```css
.modal {
  will-change: transform, opacity;
}
```

→ 브라우저에 "이게 곧 애니메이션 될 거다" 힌트. 미리 GPU layer 만듦.

주의: 남용 X. 정말 필요한 곳만.

---

## 12. JS 와 결합 — Web Animations API

```javascript
element.animate(
  [
    { opacity: 0, transform: "translateY(20px)" },
    { opacity: 1, transform: "translateY(0)" },
  ],
  {
    duration: 400,
    easing: "cubic-bezier(0.22, 1, 0.36, 1)",
    fill: "both",
  }
);
```

CSS keyframe 과 동일. JS 로 더 동적.

---

## 13. JS 라이브러리

복잡한 애니메이션:
- **Framer Motion** — React 가장 인기
- **React Spring** — spring physics
- **GSAP** — 가장 강력 (paid for some features)
- **Lottie** — Adobe After Effects 결과 재생

### Framer Motion 예
```tsx
import { motion } from "framer-motion";

<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  exit={{ opacity: 0 }}
  transition={{ duration: 0.3 }}
>
  내용
</motion.div>
```

CSS 로 어려운 enter/exit / drag / gestures 등 강력.

---

## 14. 자주 만드는 패턴

### A. 카드 hover
```css
.card {
  transition: transform 0.2s ease-out, box-shadow 0.2s;
}
.card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 20px rgba(0,0,0,0.15);
}
```

### B. button press
```css
.btn:active {
  transform: scale(0.96);
  transition: transform 0.05s;
}
```

### C. 화살표 따라옴
```css
.link::after {
  content: "→";
  transition: transform 0.2s;
}
.link:hover::after {
  transform: translateX(4px);
}
```

본인 hompage 의 card-link 패턴.

### D. 점진 등장 (stagger)
```css
.list > * {
  animation: fadeIn 0.3s ease-out both;
}
.list > *:nth-child(1) { animation-delay: 0ms; }
.list > *:nth-child(2) { animation-delay: 80ms; }
.list > *:nth-child(3) { animation-delay: 160ms; }
```

본인 hompage 의 page-anim 패턴.

---

## 15. 함정

| 증상 | 원인 |
| --- | --- |
| `display: none → block` transition 안 됨 | 안 됨. opacity + 트릭 |
| 부드럽지 않음 | width/height transition — transform 으로 |
| jank (끊김) | 작은 jank 는 will-change 또는 라이브러리 |
| Safari 만 이상 | -webkit-prefix 옛 코드 / vendor specific |

---

## 16. 다음 단계

1. **[[../layout/positioning]]** — modal / tooltip 위치
2. JS 라이브러리 — Framer Motion

---

연결: [[../html-css|↑ hub]]
