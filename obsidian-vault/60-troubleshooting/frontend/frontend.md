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

| #  | 증상 / 에러                                                                 | 한 줄 원인                          | 핵심 해결                                                                  | 깊이 노트                              |
| -- | ----------------------------------------------------------------------- | ------------------------------- | ---------------------------------------------------------------------- | ---------------------------------- |
| 1  | **Hydration mismatch** (Next.js / SSR)                                  | 서버 / 클라이언트 렌더 분기                  | useEffect client-only / 결정적 데이터 / suppressHydrationWarning (최후)         | [[hydration-mismatch]]             |
| 2  | **무한 re-render — `Too many re-renders`**                                | dep 누락 / setState in render        | dep array 정확히 / useMemo / useCallback / setState callback              | [[infinite-rerender]]              |
| 3  | **Stale closure — handler 가 옛 state**                                   | closure 가 render 시점 state 캡처    | useRef / setState((prev) => ...) / useInterval custom hook              | [[stale-closure]]                  |
| 4  | **Event listener / setInterval 누적**                                     | useEffect cleanup 누락            | cleanup return / AbortController 패턴                                    | [[event-listener-leak]]            |
| 5  | **CORS preflight 차단**                                                   | 서버가 OPTIONS 안 받음 / 헤더 누락        | server CORS middleware (Spring/Express) / same-origin proxy            | [[cors-preflight-blocked]]         |
| 6  | **z-index 가 안 먹음**                                                      | parent 의 stacking context        | Portal (createPortal to body) / isolation:isolate / design token       | [[z-index-stacking-context]]       |
| 7  | **iOS Safari 100vh — viewport 잘림**                                      | URL bar 포함 100vh 계산             | `100dvh` + fallback / `-webkit-fill-available` / JS measure            | [[ios-safari-100vh]]               |
| 8  | **localStorage QuotaExceededError**                                     | 5-10MB 초과 / iOS private mode    | IndexedDB 전환 / 압축 / private mode 감지 + memory fallback                | [[localstorage-quota-exceeded]]    |
| 9  | **Service Worker 가 옛 bundle 캐싱**                                        | cache-first + 새 deploy           | network-first for HTML / skipWaiting + claim / Workbox                | [[service-worker-stale-cache]]     |
| 10 | **ChunkLoadError — chunk hash mismatch**                                | deploy 중 옛 index.html 가 옛 chunk 참조 | atomic deploy + 옛 chunk grace period / ChunkLoadError handler        | [[webpack-chunk-hash]]             |
| 11 | **Tailwind class 가 prod 에서 사라짐**                                        | content path 미설정 → purge        | content 정확 / lookup table / safelist                                  | [[tailwind-class-purged]]          |
| 12 | **CSS Grid blow-out**                                                   | grid item default min-width:auto | min-width:0 / minmax(0, 1fr) / overflow:hidden                         | [[css-grid-blowout]]               |
| 13 | **Next.js Image — Invalid src prop**                                    | next.config.js images 미등록      | images.remotePatterns / custom loader / unoptimized (최후)             | [[nextjs-image-domain]]            |
| 14 | **Tab 전환 시 React state reset**                                          | 컴포넌트 unmount/remount             | state lift up / context / Zustand / URL search params                 | [[tab-state-reset]]                |
| 15 | **Fetch 중복 — Strict Mode dev**                                          | React 18 Strict Mode 가 effect 2회 | AbortController cleanup / TanStack Query dedupe / POST 는 event 에서   | [[fetch-strict-mode-duplicate]]    |

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
