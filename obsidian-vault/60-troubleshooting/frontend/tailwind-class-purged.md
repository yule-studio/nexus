---
title: "troubleshooting · Tailwind class 가 prod 에서 사라짐"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, tailwind, css, build]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# Tailwind class 가 prod 에서 사라짐

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- dev 에서는 정상, prod 에서만 스타일 깨짐
- 동적 className (`text-${color}-500`) 가 적용 안 됨
- 새 라이브러리 (HeadlessUI 등) class 가 적용 안 됨

## 원인

Tailwind 가 JIT mode 로 사용된 class 만 CSS 생성 (purge). `content` config 가 모든 파일 / dynamic 사용 capture 못하면 prod build 에서 누락.

## 해결

### `content` path 정확히
```js
// tailwind.config.js
module.exports = {
  content: [
    './src/**/*.{js,ts,jsx,tsx,html}',
    './node_modules/@my-org/ui/**/*.{js,ts,jsx,tsx}',   // 외부 라이브러리
    './public/**/*.html',
  ],
  // ...
};
```

Next.js 14 App Router:
```js
content: [
  './app/**/*.{js,ts,jsx,tsx,mdx}',
  './pages/**/*.{js,ts,jsx,tsx,mdx}',
  './components/**/*.{js,ts,jsx,tsx,mdx}',
],
```

### 동적 className — 전체 명시 또는 safelist

**Bad** (Tailwind 가 못 봄):
```tsx
<div className={`text-${color}-500`} />
<div className={`bg-${size > 100 ? 'red' : 'blue'}-500`} />
```

**Good — 전체 명시**:
```tsx
const colorClass = {
  red: 'text-red-500',
  blue: 'text-blue-500',
};
<div className={colorClass[color]} />
```

**또는 safelist** (config):
```js
safelist: [
  'text-red-500', 'text-blue-500', 'text-green-500',
  { pattern: /^(bg|text)-(red|blue|green)-(100|500|900)$/ },
  { pattern: /^col-span-/, variants: ['md', 'lg'] },
],
```

### `@apply` 사용
```css
/* globals.css */
.btn-danger {
  @apply text-red-500 bg-red-100 hover:bg-red-200;
}
```

→ `@apply` 가 build time 해석 — dynamic class 가 컴파일됨.

## 진단

```bash
# Build 후 CSS 에 class 가 있는지
grep "text-red-500" dist/css/*.css

# Tailwind 가 어떤 파일 scan 하는지
npx tailwindcss --content "./src/**/*.{js,ts,jsx,tsx}" -o test.css
ls -la test.css   # 크기 비교
```

## 재발 방지

- 동적 class 는 lookup table 패턴
- 새 npm package 추가 시 `content` 경로 검토
- prod build 후 visual regression (Chromatic / Percy) — 스타일 깨짐 즉시 발견

## 관련

- [[frontend]] / [[webpack-chunk-hash]]
