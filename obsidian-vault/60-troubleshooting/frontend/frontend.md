---
title: "60-troubleshooting/frontend — Frontend 트러블 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-27
tags: [troubleshooting, frontend, react, nextjs, css, browser, hub]
home_hub: "60-troubleshooting"
---

# Frontend 트러블 카탈로그

> React / Next.js / Browser / CSS 에서 자주 만나는 대표 트러블.

**[[../troubleshooting|↑ 60-troubleshooting]]** · **[[../../20-areas/frontend/frontend|↑↑ 20-areas/frontend]]**

---

## 카탈로그

| #  | 증상 / 에러                                                                 | 한 줄 원인                          | 핵심 해결                                                                  |
| -- | ----------------------------------------------------------------------- | ------------------------------- | ---------------------------------------------------------------------- |
| 1  | **Hydration mismatch — `Text content did not match`** (Next.js / SSR)   | 서버 / 클라이언트 렌더 분기 (Date / random / `window`) | `useEffect` 로 client-only 분기 / `suppressHydrationWarning` (최후) / 결정적 데이터 |
| 2  | **무한 re-render — `Too many re-renders`**                                | useEffect dep 누락 / setState in render | dep array 정확히 / state 갱신 조건 / `useMemo` / `useCallback`               |
| 3  | **Stale closure — handler 가 옛 state 사용**                                 | closure 가 render 시점 state 캡처    | `useRef` 로 latest 참조 / `setState((prev) => ...)` callback              |
| 4  | **Event listener leak — `setInterval` / `addEventListener` 누적**         | useEffect cleanup 누락            | cleanup 함수 return / `AbortController` 패턴                               |
| 5  | **CORS preflight 차단 — `No 'Access-Control-Allow-Origin'`**              | 서버가 OPTIONS 응답 안 함 / 헤더 누락      | server: CORS middleware (origin/methods/credentials) / 같은 origin proxy |
| 6  | **z-index 가 안 먹음 — modal 가 다른 element 뒤로**                              | parent 의 stacking context 가 새로 만들어짐 | `position` + `z-index` parent 까지 / portal (`createPortal` to body)    |
| 7  | **iOS Safari 100vh — viewport 가 잘림**                                    | URL bar 포함 100vh 계산             | `100dvh` (dynamic viewport unit) / `-webkit-fill-available` / JS 측정    |
| 8  | **localStorage QuotaExceededError**                                     | 5-10MB 초과 / iOS private mode    | IndexedDB 전환 / 데이터 압축 / private mode detection                         |
| 9  | **Service Worker 가 옛 bundle 캐싱**                                        | cache-first 정책 + 새 deploy       | versioned cache key / `skipWaiting()` + `clients.claim()` / network-first |
| 10 | **Webpack/Vite chunk hash mismatch — `ChunkLoadError`**                 | deploy 중 새 chunk 가 옛 index.html 참조 | atomic deploy / fallback `window.location.reload()` on ChunkLoadError  |
| 11 | **Tailwind class 가 prod 에서 사라짐**                                        | content path 미설정 → purge 됨      | `tailwind.config.js content` 에 모든 경로 / `safelist` (동적 className)      |
| 12 | **CSS Grid blow-out — `min-width: auto` 가 grid item 을 늘림**               | grid item 의 default `min-width: auto` | `min-width: 0` 또는 `overflow: hidden` 명시                                |
| 13 | **Next.js Image — `Invalid src prop`**                                  | `next.config.js` images.domains 미등록 | `images.remotePatterns` 등록 / `unoptimized={true}` (개발용)               |
| 14 | **Tab 전환 시 React state reset**                                          | parent 가 unmount/mount (key 변경) | parent key stable / context / Zustand / lifted state                  |
| 15 | **Fetch 중복 발진 — Strict Mode dev 환경**                                    | React 18 Strict Mode 가 effect 2회 실행 | dev 만 의도 / `AbortController` 로 idempotent / data 캐시 (TanStack Query) |

## 자주 쓰는 진단

```text
# Browser DevTools
- Performance tab: re-render frame
- Network tab: preflight (OPTIONS) / waterfall
- Application tab: storage / cookies / service worker
- React DevTools: Profiler

# CLI
npx lighthouse https://...      # 성능 진단
npx next-bundle-analyzer         # 번들 분석
```

## 관련

- [[../troubleshooting]]
- [[../../20-areas/frontend/frontend|20-areas/frontend]]
- [[../network/network|network 트러블]] — CORS / TLS 는 frontend ↔ network 경계
