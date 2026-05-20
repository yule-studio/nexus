---
title: "Performance Optimization — Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, performance, hub]
---

# Performance — Hub

**[[../frontend|↑ frontend]]**

> 빠른 첫 화면 / 빠른 인터랙션 / 부드러운 스크롤.

---

## 1. 측정 먼저

> "측정 안 하고 최적화 X. 추측은 항상 틀림."

### 도구
- **Lighthouse** (Chrome DevTools)
- **Chrome DevTools Performance**
- **React DevTools Profiler**
- **Web Vitals 라이브러리** — 실 사용자
- **WebPageTest** — 외부 측정

---

## 2. 학습 로드맵

### Tier 1 — 측정 / 지표
- [[web-vitals/core-web-vitals]] — LCP / INP / CLS

### Tier 2 — 코드 최적화
- [[bundle/code-splitting]] — lazy / dynamic / chunk

### Tier 3 — 자산
- [[images/image-optimization]] — WebP / AVIF / lazy / responsive

---

## 3. 핵심 지표 (Core Web Vitals)

| 지표 | 의미 | 좋음 |
| --- | --- | --- |
| **LCP** | 가장 큰 요소 로드 시점 | <2.5s |
| **INP** | 인터랙션 응답 | <200ms |
| **CLS** | 레이아웃 흔들림 | <0.1 |

자세히 [[web-vitals/core-web-vitals]].

---

## 4. 가장 큰 효과 7 가지

| 순위 | 항목 | 효과 |
| --- | --- | --- |
| 1 | 이미지 최적화 (WebP/lazy/responsive) | LCP ↓↓↓ |
| 2 | Code splitting / lazy load | First Load JS ↓↓ |
| 3 | Font display: swap + subset | LCP / CLS ↓↓ |
| 4 | Tree shaking + minify | Bundle size ↓ |
| 5 | Critical CSS inline | FCP ↓ |
| 6 | preconnect / preload | LCP ↓ |
| 7 | CDN + compression | TTFB ↓ |

---

## 5. 자주 빠지는 함정

1. **측정 안 하고 추측** — 측정 → 사실 다른 곳이 느림
2. **너무 작은 최적화** — `memo` 남용 → 코드 복잡, 효과 X
3. **premature optimization** — 처음부터 → 가독성 ↓
4. **Bundle 큰 줄 모름** — 정기 분석 필요

---

## 6. 관련

- [[../react/performance/performance|↑ React 성능]] — memoization 등
- [[../build-tools/build-tools|↑ Build tools]]
- [[../nextjs/nextjs|↑ Next.js]] — 자동 최적화
