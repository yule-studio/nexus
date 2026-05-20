---
title: "Core Web Vitals — LCP / INP / CLS"
kind: knowledge
project: frontend
status: current
tags: [frontend, performance, web-vitals, lcp, inp, cls]
---

# Core Web Vitals

**[[../performance-optimization|↑ Performance hub]]**

> Google 이 정한 사용자 경험 핵심 3 지표. SEO 에도 영향.

---

## 1. LCP — Largest Contentful Paint

화면에서 가장 큰 요소 (이미지 / 큰 텍스트 블록) 가 보일 때까지의 시간.

### 기준
- 좋음: **< 2.5s**
- 개선 필요: 2.5 ~ 4s
- 나쁨: > 4s

### 자주 LCP 가 되는 요소
- 메인 hero 이미지
- H1 + 본문 첫 단락
- 큰 비디오

### 개선 — 핵심 5
1. **이미지 최적화** (WebP / AVIF / 적절한 크기)
2. **preload hero 이미지**
   ```html
   <link rel="preload" as="image" href="/hero.webp">
   ```
3. **preconnect 외부 도메인**
   ```html
   <link rel="preconnect" href="https://cdn.example.com">
   ```
4. **Server-side rendering (SSR / SSG)** — 첫 HTML 이 즉시 컨텐츠 포함
5. **Critical CSS inline** — 첫 화면 CSS 만 inline, 나머지는 async

### 빠른 검사
1. Lighthouse → LCP element 표시
2. Chrome DevTools → Performance → LCP marker

---

## 2. INP — Interaction to Next Paint

사용자 인터랙션 (클릭 / 키 입력) → 화면 응답 시간.

### 기준
- 좋음: **< 200ms**
- 개선 필요: 200 ~ 500ms
- 나쁨: > 500ms

### 이전 (FID) 와 차이
- FID — 첫 인터랙션 만
- INP — 모든 인터랙션의 최악값. 더 엄격.

### 자주 INP 가 안 좋은 경우
- 큰 동기 JS 실행 (정렬 / 계산)
- 너무 자주 setState
- React 의 controlled input 이 무거움
- 글로벌 state 의 큰 객체 업데이트

### 개선
1. **무거운 작업 분할** (`requestIdleCallback`, Web Worker)
2. **debounce / throttle** (검색 입력 등)
3. **React.startTransition** — non-urgent 작업 표시
   ```tsx
   import { startTransition } from "react";
   startTransition(() => setItems(bigList));
   ```
4. **useDeferredValue** — 무거운 list 의 입력
5. **virtualization** — 긴 list 는 보이는 것만 렌더 (react-virtual)
6. **React Compiler** (React 19+) — 자동 memoization

---

## 3. CLS — Cumulative Layout Shift

페이지 로딩 / 인터랙션 중 갑자기 element 가 움직이는 정도.

### 기준
- 좋음: **< 0.1**
- 개선 필요: 0.1 ~ 0.25
- 나쁨: > 0.25

### 자주 CLS 가 일어나는 경우
- 이미지 / 비디오에 width/height 없음 → 늦게 로드되면서 텍스트 밀어냄
- 폰트 늦게 로드 → text reflow
- 광고 / iframe 늦게 삽입
- 동적으로 추가되는 배너 / 알림

### 개선
1. **이미지 / 비디오에 size 명시**
   ```html
   <img src="..." width="800" height="600">
   ```
   또는:
   ```css
   img { aspect-ratio: 4/3; width: 100%; height: auto; }
   ```

2. **font-display: swap + preload**
   ```html
   <link rel="preload" as="font" href="/font.woff2" crossorigin>
   ```
   ```css
   @font-face {
     font-family: "MyFont";
     font-display: swap;
   }
   ```

3. **광고 자리 미리 예약**
   ```css
   .ad-slot { min-height: 100px; }
   ```

4. **dynamic content 는 기존 컨텐츠 위로**
   ```css
   /* ❌ 콘텐츠 사이에 삽입 */
   /* ✅ 페이지 맨 위 sticky 알림 */
   ```

---

## 4. 측정 — Lab vs Field

### Lab data (개발 시점)
- Lighthouse
- DevTools Performance
- WebPageTest
- 한 번의 측정. 통제된 환경

### Field data (실 사용자)
- web-vitals 라이브러리 + analytics
- Chrome User Experience Report (CrUX) — 공개 데이터
- Vercel Analytics

**둘 다 봐야.** Lab 은 빠른 피드백, Field 는 진짜 사용자.

---

## 5. 실 사용자 측정 셋업

```typescript
// src/lib/web-vitals.ts
import { onCLS, onINP, onLCP, onFCP, onTTFB } from "web-vitals";

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    id: metric.id,
    page: location.pathname,
  });

  // sendBeacon 권장 (페이지 닫혀도 전송)
  navigator.sendBeacon?.("/analytics", body) ||
    fetch("/analytics", { body, method: "POST", keepalive: true });
}

onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

```typescript
// App 진입점
import "./lib/web-vitals";
```

### Vercel / Cloudflare 자동
이미 web-vitals 자동 수집 + 대시보드.

---

## 6. Next.js 자동 최적화

Next 는 많은 걸 자동:

### Image
```tsx
import Image from "next/image";

<Image src="/hero.jpg" alt="" width={800} height={600} priority />
```

- WebP/AVIF 자동 변환
- 적응형 srcset
- lazy load
- `priority` = LCP 후보면 preload

### Font
```typescript
// app/layout.tsx
import { Inter } from "next/font/google";

const inter = Inter({ subsets: ["latin"], display: "swap" });

<body className={inter.className}>...
```

- Self-hosting (CSS만 외부 X)
- preload 자동
- display: swap 기본

---

## 7. 자주 만나는 패턴

### A. LCP 가 hero 이미지
```html
<!-- ✅ -->
<img src="hero.webp" alt="" width="1200" height="600"
     loading="eager" fetchpriority="high">

<!-- 또는 preload -->
<link rel="preload" as="image" href="hero.webp" fetchpriority="high">
```

### B. CLS — 사진 reflow
```css
img {
  width: 100%;
  height: auto;
  aspect-ratio: 16/9;
  background: #eee;   /* placeholder */
}
```

### C. INP — 큰 list 검색
```tsx
import { useDeferredValue } from "react";

function Search({ items }) {
  const [query, setQuery] = useState("");
  const deferred = useDeferredValue(query);
  const filtered = items.filter(i => i.includes(deferred));

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <List items={filtered} />
    </>
  );
}
```

input 은 즉시 반응, list 는 deferred 로 부드럽게.

---

## 8. 점수 향상 체크리스트

### LCP
- [ ] hero 이미지에 `loading="eager"` + `fetchpriority="high"`
- [ ] preconnect to image CDN
- [ ] SSR / SSG (Next 라면 자동)
- [ ] WebP/AVIF
- [ ] Critical CSS inline

### INP
- [ ] React.startTransition / useDeferredValue
- [ ] virtualization (긴 list)
- [ ] debounce 검색 / scroll
- [ ] Web Worker 로 무거운 계산

### CLS
- [ ] 모든 `<img>` 에 width/height
- [ ] font-display: swap
- [ ] 광고 / 동적 콘텐츠 자리 예약
- [ ] transform 사용 (top/left 보다)

---

## 9. 함정

| 증상 | 원인 |
| --- | --- |
| Lab 은 좋은데 Field 는 나쁨 | 실 모바일 / 느린 네트워크는 다름 |
| LCP 가 텍스트인데 안 좋음 | 큰 hero 이미지로 옮기거나 font preload |
| CLS 가 한 페이지만 높음 | 그 페이지의 동적 삽입 / 광고 확인 |
| INP 가 모바일만 높음 | 큰 JS bundle / unmemoized list |

---

## 10. 다음 단계

1. **[[../bundle/code-splitting]]** — JS bundle 줄이기
2. **[[../images/image-optimization]]** — 이미지

---

연결: [[../performance-optimization|↑ hub]]
