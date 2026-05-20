---
title: "Design Tokens — 디자인 변수 체계"
kind: knowledge
project: frontend
status: current
tags: [frontend, design-systems, tokens]
---

# Design Tokens

**[[../design-systems|↑ Design Systems hub]]**

> "Design Decision 의 단일 source of truth". 색·간격·폰트를 변수로.

---

## 1. 왜?

### 토큰 없으면 (안 좋음)
```css
.button { background: #3b82f6; padding: 8px 16px; }
.alert  { background: #3b82f6; padding: 8px 16px; }
.tag    { background: #3b82f6; padding: 8px 16px; }
```

→ 색 바꾸려면 grep / 치환. 누락 위험.

### 토큰 사용
```css
:root {
  --color-primary: #3b82f6;
  --spacing-md: 16px;
  --spacing-sm: 8px;
}

.button { background: var(--color-primary); padding: var(--spacing-sm) var(--spacing-md); }
.alert  { background: var(--color-primary); padding: var(--spacing-sm) var(--spacing-md); }
.tag    { background: var(--color-primary); padding: var(--spacing-sm) var(--spacing-md); }
```

→ 한 곳 (`--color-primary`) 변경 → 전체 적용.

---

## 2. 3 계층 (현업 표준)

### Primitive (원시)
브랜드 무관한 raw 값.
```css
--blue-50:  #eff6ff;
--blue-500: #3b82f6;
--blue-900: #1e3a8a;

--gray-50:  #f9fafb;
--gray-500: #6b7280;

--space-1: 4px;
--space-2: 8px;
```

### Semantic (의미)
브랜드 / 의도 부여.
```css
--color-primary: var(--blue-500);
--color-bg: var(--gray-50);
--color-text: var(--gray-900);
--color-danger: var(--red-500);

--space-card-padding: var(--space-4);
```

### Component (특정 컴포넌트)
```css
--button-bg: var(--color-primary);
--button-text: white;
--card-bg: white;
--card-border: var(--gray-200);
```

→ 컴포넌트가 변경되면 component 토큰만 수정.

---

## 3. 본인 hompage 의 토큰

```css
/* src/styles/tokens.css */
:root {
  /* surface */
  --bg: #0b0d0c;
  --bg-card: #0f1311;

  /* accent */
  --accent: #4ade80;
  --accent-strong: #22c55e;

  /* text */
  --text: #e6ece8;
  --text-muted: #8a948f;

  /* spacing */
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;

  /* radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-pill: 999px;

  /* type */
  --text-xs: 11px;
  --text-sm: 12.5px;
  --text-md: 15px;
  --text-lg: 17px;
  --text-2xl: 26px;
}
```

자세히 [[../../10-projects/hompage/styling/tokens|↑ hompage tokens 사전]].

---

## 4. 다크 / 라이트 모드

### `data-theme` 속성으로 분기
```css
:root,
:root[data-theme="dark"] {
  --bg: #0b0d0c;
  --text: white;
}

:root[data-theme="light"] {
  --bg: white;
  --text: #161a18;
}
```

```html
<html data-theme="dark">    <!-- 또는 "light" -->
```

```javascript
document.documentElement.dataset.theme = "light";
```

### media query 자동
```css
@media (prefers-color-scheme: dark) {
  :root {
    --bg: black;
    --text: white;
  }
}
```

OS 시스템 설정 따라감.

---

## 5. 플랫폼 / 환경 다층

### Tailwind 시스템
`tailwind.config.js`:
```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: "#3b82f6",
        secondary: "#8b5cf6",
      },
      spacing: {
        "card": "16px",
      },
    },
  },
};
```

```html
<button class="bg-primary p-card">Click</button>
```

### Figma 와 동기화
- **Tokens Studio** (Figma plugin) — Figma 에서 정의 → JSON export
- **Style Dictionary** (Amazon) — JSON 토큰 → CSS / iOS / Android / 웹 변환

```json
{
  "color": {
    "primary": { "value": "#3b82f6" }
  }
}
```
↓ Style Dictionary
↓
```css
:root { --color-primary: #3b82f6; }
```
```swift
let colorPrimary = UIColor(rgb: 0x3b82f6)
```

→ Design 팀 + 개발 팀 동기화.

---

## 6. 토큰 명명 규칙

### 좋은 이름
```css
--color-primary           /* 의미 명확 */
--space-card-padding      /* 의도 명확 */
--text-button-label
```

### 나쁜 이름
```css
--blue          /* 무슨 파랑? primary? */
--padding-3     /* 어디 쓰는? */
--my-color
```

### Tailwind 식 (T-shirt size)
```css
--text-xs / sm / md / lg / xl / 2xl / 3xl
--space-1 / 2 / 3 / 4 / 5
--radius-sm / md / lg
```

직관적. 본인 hompage 도 이 패턴.

---

## 7. 다중 브랜드 (multi-brand)

```css
:root.brand-a {
  --color-primary: blue;
}

:root.brand-b {
  --color-primary: purple;
}
```

같은 컴포넌트, 다른 브랜드.

---

## 8. 토큰 사용 빈도 (현업 통계)

자주:
- 색 (color)
- 간격 (spacing) — padding / margin / gap
- 반지름 (radius)
- 폰트 크기 / line-height
- 그림자 (shadow)
- 트랜지션 시간 / easing

가끔:
- z-index 단계 (modal / tooltip 등)
- breakpoint (반응형)
- container 최대 폭

---

## 9. 안티패턴

### ❌ raw 색을 컴포넌트에
```css
.button { background: #3b82f6; }
```

### ✅ semantic 거쳐서
```css
.button { background: var(--color-primary); }
```

### ❌ 토큰 너무 세분화
```css
--space-very-very-small: 2px;
--space-very-small: 4px;
--space-quite-small: 6px;
```

### ✅ 적절한 단계
```css
--space-1: 4px;
--space-2: 8px;
--space-3: 12px;
```

---

## 10. 도구

- **Style Dictionary** (Amazon) — JSON → 다 플랫폼 변환
- **Tokens Studio** — Figma plugin
- **Theo** (Salesforce) — Style Dictionary 이전 도구
- **Specify** — SaaS 토큰 관리

---

## 11. 새 컴포넌트 만들 때 — 체크리스트

- [ ] 색은 semantic token 사용 (`--color-primary` 같은)
- [ ] padding / margin 은 spacing token (`--space-3`)
- [ ] 반지름은 radius token (`--radius-md`)
- [ ] 폰트 크기는 text token (`--text-md`)
- [ ] hard-coded 값 없음 (`#3b82f6` 같은 직접 X)

---

## 12. 다음 단계

1. **[[../storybook/storybook-basics]]** — Storybook 으로 토큰 / 컴포넌트 문서화

---

연결: [[../design-systems|↑ hub]]
