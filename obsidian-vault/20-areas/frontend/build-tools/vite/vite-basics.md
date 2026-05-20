---
title: "Vite — 기초 + dev / build"
kind: knowledge
project: frontend
status: current
tags: [frontend, build-tools, vite, beginner]
---

# Vite

**[[../build-tools|↑ Build tools hub]]**

> 빠른 dev 서버 + Rollup 기반 production 빌드. React/Vue/Svelte 다 지원.

---

## 1. 왜 빠른가

### Webpack (옛)
1. 모든 모듈 번들링 → dev 서버 시작 (느림)
2. 코드 변경 → 다시 번들 (느림)

### Vite
1. 브라우저의 native ESM 활용 → 번들 안 함 (dev 즉시 시작)
2. 코드 변경 → 그 모듈만 변환 (즉시 HMR)
3. Production 만 Rollup 으로 번들

→ 100 모듈 → Vite 200ms vs Webpack 10s.

---

## 2. 프로젝트 시작

```bash
npm create vite@latest my-app
# Framework? React / Vue / Svelte / Vanilla
# Variant? TypeScript

cd my-app
npm install
npm run dev      # http://localhost:5173
```

---

## 3. 기본 구조

```
my-app/
├─ index.html              ← 진입점 (root 에 있음 — 특이)
├─ src/
│  ├─ main.tsx
│  ├─ App.tsx
│  └─ ...
├─ public/                  ← 정적 파일 (가공 X)
├─ vite.config.ts
├─ tsconfig.json
└─ package.json
```

### index.html
```html
<!DOCTYPE html>
<html>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

다른 도구와 달리 **index.html 이 root** 에 있고 그 안의 `<script>` 가 진입점.

---

## 4. npm scripts

```json
{
  "scripts": {
    "dev": "vite",                    // dev 서버
    "build": "tsc -b && vite build",  // production 빌드
    "preview": "vite preview"          // build 결과 미리보기
  }
}
```

### dev 서버
```bash
npm run dev
# http://localhost:5173
```

옵션:
```bash
npm run dev -- --port 3000
npm run dev -- --host    # LAN 노출 (다른 디바이스에서 접근)
```

### Build
```bash
npm run build
# dist/ 생성
```

### Preview (build 결과 테스트)
```bash
npm run preview
# http://localhost:4173
```

---

## 5. import 옵션

### 정적 import
```typescript
import App from "./App";
import "./style.css";
```

### dynamic import (코드 분할)
```typescript
const Chart = await import("./Chart");
// 사용 시점에만 로드
```

### React lazy
```typescript
import { lazy, Suspense } from "react";

const Heavy = lazy(() => import("./Heavy"));

<Suspense fallback={<div>...</div>}>
  <Heavy />
</Suspense>
```

### 이미지 / 정적 파일
```typescript
import logoUrl from "./logo.svg";     // URL 문자열
import logoRaw from "./logo.svg?raw"; // 텍스트 내용

<img src={logoUrl} />
```

### CSS
```typescript
import "./style.css";          // 자동 주입
import classes from "./Button.module.css";   // CSS Modules
```

### JSON
```typescript
import data from "./data.json";
console.log(data.name);
```

---

## 6. CSS

### 단순 CSS
```css
/* style.css */
body { background: black; }
```

```typescript
import "./style.css";
```

### CSS Modules
```css
/* Button.module.css */
.button { background: blue; }
```

```typescript
import styles from "./Button.module.css";
<button className={styles.button} />
```

자동 클래스명 고유화 (`Button_button_x29Y` 같이).

### PostCSS
`postcss.config.cjs`:
```javascript
module.exports = {
  plugins: { autoprefixer: {}, "postcss-preset-env": {} },
};
```

자동 prefix (`-webkit-`, `-ms-` 등).

### Sass / Less
```bash
npm install -D sass    # 또는 less
```

자동 작동:
```css
@import "./vars";
.btn { color: $primary; }
```

### Tailwind
```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

`tailwind.config.js`:
```javascript
module.exports = {
  content: ["./index.html", "./src/**/*.{ts,tsx}"],
  theme: { extend: {} },
  plugins: [],
};
```

`src/index.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

## 7. 환경변수

`.env`:
```
VITE_API_URL=https://api.example.com
VITE_PUBLIC_KEY=abc
```

⚠️ **`VITE_` 시작 만 브라우저에 노출**. 그 외는 무시.

사용:
```typescript
import.meta.env.VITE_API_URL    // "https://api.example.com"
import.meta.env.MODE             // "development" | "production"
import.meta.env.DEV              // boolean
import.meta.env.PROD             // boolean
```

### .env 파일 우선순위
1. `.env.local` (모든 모드, gitignore)
2. `.env.development` / `.env.production`
3. `.env`

---

## 8. import alias

`vite.config.ts`:
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

사용:
```typescript
import Layout from "@/components/Layout";
import { api } from "@/lib/api";
```

---

## 9. plugin

### React
```bash
npm install -D @vitejs/plugin-react
```

```typescript
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
});
```

### 자주 쓰는 plugin
```bash
npm install -D vite-plugin-svgr     # SVG → React component
npm install -D vite-tsconfig-paths   # tsconfig paths 자동
npm install -D vite-plugin-pwa       # PWA
```

---

## 10. proxy (CORS 우회)

dev 서버에서 다른 backend 로:
```typescript
export default defineConfig({
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:8080",
        changeOrigin: true,
      },
    },
  },
});
```

→ `/api/*` 요청이 `http://localhost:8080/api/*` 로.

---

## 11. build output 분석

```typescript
export default defineConfig({
  build: {
    sourcemap: true,           // .map 파일 생성
    minify: "terser",          // 또는 "esbuild" (default)

    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          ui: ["@mui/material", "antd"],
        },
      },
    },
  },
});
```

### Bundle 크기 확인
```bash
npm install -D rollup-plugin-visualizer
```

```typescript
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [react(), visualizer({ open: true })],
});
```

`npm run build` → 자동으로 stats.html 열림 → 시각적 분석.

---

## 12. SSR / SSG (Vite 자체)

Vite 는 SSR 지원하지만 Next 같이 통합 X. 직접 셋업 필요.

대안: **vite-plugin-ssr (Vike)** 또는 **Next.js** / **Astro**.

---

## 13. library mode (npm 패키지 만들기)

```typescript
export default defineConfig({
  build: {
    lib: {
      entry: "src/index.ts",
      name: "MyLib",
      fileName: "my-lib",
      formats: ["es", "cjs"],
    },
    rollupOptions: {
      external: ["react", "react-dom"],
    },
  },
});
```

→ `dist/my-lib.js` + `dist/my-lib.cjs.js` 생성.

---

## 14. 자주 만나는 함정

| 증상 | 원인 |
| --- | --- |
| 환경변수 undefined | `VITE_` 접두사 + 재시작 |
| import alias 안 됨 | vite.config + tsconfig 둘 다 |
| HMR 안 됨 | export default 한 컴포넌트만 (named X) |
| build 후 화면 빈 | `<script type="module">` 누락 / base path |
| CORS 에러 (dev) | proxy 설정 |
| 큰 번들 | `manualChunks` / dynamic import |

---

## 15. 다음 단계

1. **[[vite-config]]** — vite.config.ts 자세히
2. **[[../bundlers/code-splitting]]** — 분할 패턴

---

연결: [[../build-tools|↑ hub]] · [[../../react/react|React]]
