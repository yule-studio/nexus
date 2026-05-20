---
title: "Code Splitting — bundle 분할"
kind: knowledge
project: frontend
status: current
tags: [frontend, performance, code-splitting, bundle]
---

# Code Splitting

**[[../performance-optimization|↑ Performance hub]]**

> 큰 번들을 작게 쪼개 → 첫 로드 빠름.

---

## 1. 문제 — 한 덩어리 번들

```
index.js (3MB)
  ↓
브라우저 다운로드 (3G: 30초)
  ↓
파싱 / 실행
  ↓
화면
```

→ 첫 페이지에 안 쓰는 코드까지 전부 다운로드 = 낭비.

---

## 2. 해결 — split 분할

```
main.js (100KB) — 첫 페이지에 꼭 필요
chart.js (500KB) — Chart 페이지 만
admin.js (300KB) — Admin 만
heavy-lib.js (1MB) — 실제 사용 시 only
```

→ 첫 로드 100KB 만. 나머지는 필요할 때.

---

## 3. dynamic import — 기본

```typescript
// 보통
import { heavyFn } from "./heavy";

// 동적
const { heavyFn } = await import("./heavy");
```

→ 그 시점에 fetch + parse.

### Vite / Next 자동
`import()` 보면 자동으로 별도 chunk 생성.

---

## 4. React lazy + Suspense

```tsx
import { lazy, Suspense } from "react";

const HeavyChart = lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<p>차트 로딩...</p>}>
        <HeavyChart />
      </Suspense>
    </div>
  );
}
```

→ `<HeavyChart>` 가 화면에 나타날 때 비로소 fetch.

---

## 5. Route 기반 split

가장 자주 — 페이지 별 별도 chunk.

### React Router
```tsx
import { lazy } from "react";

const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));

<Suspense fallback={<Loading />}>
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/settings" element={<Settings />} />
  </Routes>
</Suspense>
```

### Next.js (자동)
페이지 단위 자동 split. 추가 작업 X.

### `next/dynamic`
```tsx
import dynamic from "next/dynamic";

const Chart = dynamic(() => import("./Chart"), {
  loading: () => <p>로딩...</p>,
  ssr: false,    // SSR 끔 (브라우저에서만)
});
```

---

## 6. 조건부 import

```tsx
async function exportPdf() {
  const { generatePdf } = await import("./pdf-generator");
  generatePdf(data);
}

<button onClick={exportPdf}>PDF 내보내기</button>
```

→ pdf-generator (큰 라이브러리) 가 실제 클릭 시 fetch.

---

## 7. 큰 라이브러리 다루기

### 전체 import (안 좋음)
```typescript
import _ from "lodash";    // 700KB
_.chunk(arr, 2);
```

### 부분 import (좋음)
```typescript
import { chunk } from "lodash";   // tree-shake 가능 (esm 일 때)
chunk(arr, 2);
```

### path import (확실)
```typescript
import chunk from "lodash/chunk";   // 정확히 chunk 만
```

### es-toolkit 같은 modern 라이브러리
```bash
npm install es-toolkit
```
ES module + tree-shake 친화.

---

## 8. 대안 라이브러리 (가벼움)

| 무거움 | 대안 |
| --- | --- |
| moment (300KB) | date-fns / dayjs (10KB) |
| lodash | es-toolkit / radash |
| jQuery | 그냥 native DOM |
| Material UI | Mantine / Radix UI |
| Chart.js (full) | recharts / visx |

---

## 9. Bundle 분석

### Vite
```bash
npm install -D rollup-plugin-visualizer
```

```typescript
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [react(), visualizer({ open: true })],
});
```

`npm run build` → 자동으로 시각 분석 열림.

### Next.js
```bash
npm install -D @next/bundle-analyzer
```

`next.config.ts`:
```typescript
import withBundleAnalyzer from "@next/bundle-analyzer";

const config = { ... };
export default withBundleAnalyzer({ enabled: process.env.ANALYZE === "true" })(config);
```

```bash
ANALYZE=true npm run build
```

### Webpack
`webpack-bundle-analyzer`.

---

## 10. manual chunks (Vite)

```typescript
// vite.config.ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        "react-vendor": ["react", "react-dom", "react-router-dom"],
        "ui-vendor": ["@mui/material", "@emotion/react"],
        "data-vendor": ["@tanstack/react-query", "axios"],
      },
    },
  },
}
```

이러면 큰 라이브러리들이 별도 vendor chunk 로 → 캐싱 효율.

---

## 11. 이미지 / 폰트는 별도

JS code splitting 아니지만 같은 원리:

### Lazy image
```html
<img src="..." loading="lazy">    <!-- native -->
```

### 폰트 부분만
```css
@font-face {
  font-family: "MyFont";
  src: url("/font-subset-ko.woff2") format("woff2");
  unicode-range: U+AC00-D7A3;    /* 한글 음절 만 */
}
```

---

## 12. preload / prefetch / preconnect

```html
<!-- preload — 즉시 다운로드 (LCP 자원) -->
<link rel="preload" as="image" href="/hero.webp">
<link rel="preload" as="font" href="/font.woff2" crossorigin>

<!-- prefetch — 한가할 때 다운로드 (다음 페이지) -->
<link rel="prefetch" href="/next-page.js">

<!-- preconnect — TCP / TLS 미리 (외부 도메인) -->
<link rel="preconnect" href="https://cdn.example.com">

<!-- dns-prefetch — DNS 미리 -->
<link rel="dns-prefetch" href="https://cdn.example.com">
```

Next.js `<Link>` 는 prefetch 자동.

---

## 13. tree shaking 안 되는 경우

- CommonJS (require) 사용
- Side effect 있는 라이브러리 (`import "lib"` 만으로 전역 추가)
- `package.json` 에 `sideEffects: false` 없음

```json
// package.json
{
  "sideEffects": false       // 또는 ["*.css"]
}
```

---

## 14. 체크리스트

- [ ] 첫 페이지 bundle < 200KB (gzip)
- [ ] Route 별 lazy load
- [ ] 큰 라이브러리는 dynamic import
- [ ] vendor / route chunk 분리
- [ ] 사용 안 하는 import 제거
- [ ] tree-shake 가능한 라이브러리 사용
- [ ] preload critical fonts / images

---

## 15. 함정

| 증상 | 원인 |
| --- | --- |
| chunk 가 너무 많음 | 과도한 split → HTTP 요청 ↑. 적당한 크기 |
| lazy 가 안 됨 | named export — `lazy(() => import().then(m => ({ default: m.X })))` |
| Suspense 가 fallback 안 보임 | 너무 빨라서. throttle 로 테스트 |
| bundle 분석 결과 한 chunk 가 거대 | barrel import / side effect |

---

## 16. 다음 단계

1. **[[../images/image-optimization]]** — 이미지
2. **[[../web-vitals/core-web-vitals]]** — 측정

---

연결: [[../performance-optimization|↑ hub]]
