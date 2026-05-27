---
title: "troubleshooting · Service Worker 가 옛 bundle 캐싱"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, pwa, service-worker, cache, deploy]
home_hub: "60-troubleshooting"
related: ["[[frontend]]", "[[webpack-chunk-hash.md|webpack-chunk-hash]]"]
---

# Service Worker — 새 deploy 후에도 옛 버전 표시

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- deploy 했는데 사용자가 옛 UI 보고 있음
- "강제 새로고침 (Cmd+Shift+R)" 하면 됨
- DevTools Application → Service Workers 에 stale SW 활성

## 원인

cache-first 전략 SW 가 새 deploy 후에도 cache 우선 → 옛 chunk / index.html 반환. 새 SW 는 background install 만 — 다음 페이지 로드 / 모든 tab 닫고 다시 열어야 활성.

## 해결

### `skipWaiting` + `clients.claim`
```js
// service-worker.js
self.addEventListener('install', e => {
  self.skipWaiting();   // 새 SW 즉시 activate
});

self.addEventListener('activate', e => {
  e.waitUntil(clients.claim());   // 모든 tab 즉시 controlled
});
```

→ 즉시 적용. 단 사용자가 polling 중인 화면이 reload 됨.

### Versioned cache
```js
const CACHE_NAME = 'app-v3';   // deploy 마다 bump

self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(names =>
      Promise.all(names.filter(n => n !== CACHE_NAME).map(n => caches.delete(n)))
    )
  );
});
```

### Network-first for HTML
```js
self.addEventListener('fetch', e => {
  const url = new URL(e.request.url);

  // HTML — 항상 network 우선 (index.html 가 최신 chunk hash)
  if (e.request.mode === 'navigate' || url.pathname.endsWith('.html')) {
    e.respondWith(
      fetch(e.request).catch(() => caches.match(e.request))
    );
    return;
  }

  // JS / CSS / image — cache-first (filename 에 hash 있어서 immutable)
  e.respondWith(
    caches.match(e.request).then(cached => cached || fetch(e.request).then(r => {
      const clone = r.clone();
      caches.open(CACHE_NAME).then(c => c.put(e.request, clone));
      return r;
    }))
  );
});
```

### Workbox 사용 (권장)
```js
import { registerRoute } from 'workbox-routing';
import { NetworkFirst, CacheFirst } from 'workbox-strategies';

registerRoute(
  ({ request }) => request.mode === 'navigate',
  new NetworkFirst({ cacheName: 'pages' })
);

registerRoute(
  ({ request }) => ['script', 'style', 'image'].includes(request.destination),
  new CacheFirst({ cacheName: 'assets' })
);
```

### 사용자에 새 버전 알림
```js
// Client side
navigator.serviceWorker.addEventListener('controllerchange', () => {
  window.location.reload();   // 새 SW activate 시 reload
});

// 또는 toast 안내
if (registration.waiting) {
  showToast('새 버전 준비됨', { action: () => registration.waiting.postMessage({ type: 'SKIP_WAITING' }) });
}
```

## SW 자체 제거 (긴급)

```js
// 모든 SW unregister + cache 삭제
navigator.serviceWorker.getRegistrations()
  .then(regs => Promise.all(regs.map(r => r.unregister())));

caches.keys().then(keys => Promise.all(keys.map(k => caches.delete(k))));
```

배포에 broken SW 가 푸시되면 큰 사고 (모든 사용자 영구 stale). 해결책: emergency fallback SW (위 코드 — 자기 자신 unregister).

## 진단

```
# DevTools Application → Service Workers
- 현재 SW 의 install / activate time
- Skip waiting / Update on reload 옵션 (dev)

# Network tab
- Response → from ServiceWorker / from disk cache / from memory cache
- 옛 chunk → SW cache 가 stale
```

## 재발 방지

- HTML 는 network-first / asset 은 cache-first 패턴 표준
- `skipWaiting` + `clients.claim` default
- deploy time 에 emergency unregister SW 배포 흐름 준비
- 모니터링: SW version + client version mismatch 알람

## 관련

- [[frontend]] / [[webpack-chunk-hash]] / [[localstorage-quota-exceeded]]
