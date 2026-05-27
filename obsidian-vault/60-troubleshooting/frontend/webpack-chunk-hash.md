---
title: "troubleshooting · Webpack/Vite chunk hash mismatch — ChunkLoadError"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, webpack, vite, deploy, chunk]
home_hub: "60-troubleshooting"
related: ["[[frontend]]", "[[service-worker-stale-cache]]"]
---

# ChunkLoadError — `Loading chunk N failed`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

```
ChunkLoadError: Loading chunk 42 failed.
(error: https://app.example.com/_next/static/chunks/42-abc123.js)
```

- 사용자가 페이지 보고 있는 동안 새 deploy 발생
- lazy chunk (`React.lazy` / `dynamic import`) 가 옛 hash 로 요청 → 404
- 또는 옛 SW 가 옛 index.html 반환 → 옛 hash 의 chunk 요청

## 원인

1. **Deploy 중 race**: 새 index.html 가 새 chunk hash 참조하는데, 옛 index.html cache 가 옛 hash 의 chunk 요청 → 옛 chunk 는 삭제됨 → 404
2. **CDN cache miss**: 새 chunk 가 CDN 에 아직 propagate 안 됨
3. **Service worker stale** — [[service-worker-stale-cache]]

## 해결

### Atomic deploy — 옛 chunk 유지 (grace period)
```
deploy 시:
1. 새 chunk 업로드 (기존 chunk 덮어쓰기 X)
2. 새 index.html 업로드 — 새 chunk hash 참조
3. 옛 chunk 는 7일 유지 (점진적 새 deploy 후 정리)
```

S3 / CDN policy 로 immutable + long TTL.

### Client-side ChunkLoadError handler
```tsx
// React Router / Next.js
useEffect(() => {
  const handler = (e: PromiseRejectionEvent) => {
    if (e.reason?.name === 'ChunkLoadError') {
      console.warn('Chunk load failed — reloading');
      window.location.reload();
    }
  };
  window.addEventListener('unhandledrejection', handler);
  return () => window.removeEventListener('unhandledrejection', handler);
}, []);
```

또는 webpack 의 `output.publicPath` 가 retry 지원하는 plugin (`retry-chunk-load-plugin`).

```js
// webpack.config.js
const RetryChunkLoadPlugin = require('webpack-retry-chunk-load-plugin');

plugins: [
  new RetryChunkLoadPlugin({
    retryDelay: 3000,
    maxRetries: 3,
    lastResortScript: 'window.location.reload()',
  }),
];
```

### Next.js — `swcMinify` + `output: 'standalone'`
```js
// next.config.js
module.exports = {
  generateBuildId: async () => process.env.GIT_SHA,   // commit hash 기반
  experimental: { swcMinify: true },
};
```

### Cache busting + immutable
```
Cache-Control: public, max-age=31536000, immutable
# chunk file (hash 포함) 은 영구 cache
# index.html 만 cache 짧게
```

### Long polling 화면 — 새 version 알림
```tsx
const { data: serverVersion } = useSWR('/api/version', fetcher, { refreshInterval: 60_000 });

useEffect(() => {
  if (serverVersion && serverVersion !== window.__APP_VERSION__) {
    toast('새 버전 — 새로고침 권장', { onClick: () => window.location.reload() });
  }
}, [serverVersion]);
```

## 진단

```
# DevTools Network
- chunk URL 의 hash + 응답 (404 = atomic deploy 실패)

# CDN log
- 404 응답 chunk 확인
- 새 chunk 가 origin 에 있는지

# Build artifact
- dist/chunks/ 의 hash 가 index.html 의 reference 와 일치
```

## 재발 방지

- atomic deploy + 옛 chunk grace period (S3 lifecycle 7-30일)
- ChunkLoadError handler 표준 적용
- 모니터링: 404 응답 chunk file 알람 (deploy 직후 잠시 spike 정상, 지속 시 사고)
- e2e: 옛 buildId 시뮬레이션 → reload 회복 검증

## 관련

- [[frontend]] / [[service-worker-stale-cache]] / [[../devops/devops]]
