---
title: "troubleshooting · Event listener / setInterval 누수"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, react, hooks, memory-leak, cleanup]
home_hub: "60-troubleshooting"
related: ["[[frontend]]", "[[stale-closure]]"]
---

# Event listener / setInterval 누수

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- SPA 페이지 이동 후 메모리 / CPU 사용량 증가
- 같은 event 가 여러 번 처리 (handler 가 누적)
- Chrome DevTools Memory profiler 에서 detached DOM tree 다수

## 원인

```tsx
function Bad() {
  useEffect(() => {
    window.addEventListener('resize', handleResize);
    // cleanup 누락 → unmount 후에도 listener 살아있음
  }, []);
}

function Bad2() {
  useEffect(() => {
    setInterval(tick, 1000);
    // clearInterval 누락 → mount 마다 새 interval, unmount 후에도 실행
  }, []);
}
```

`useEffect` cleanup 함수 누락 — unmount 시 정리 안 됨. mount/unmount 반복 (SPA 라우팅) 시 누적.

## 해결

### Cleanup function return
```tsx
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize);
}, []);

useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);
```

### AbortController 패턴 (modern)
```tsx
useEffect(() => {
  const controller = new AbortController();

  window.addEventListener('resize', onResize, { signal: controller.signal });
  document.addEventListener('keydown', onKey, { signal: controller.signal });
  fetch('/api/data', { signal: controller.signal }).then(...);

  return () => controller.abort();   // 모든 등록 한 번에 정리
}, []);
```

### Subscription pattern (RxJS / EventEmitter)
```tsx
useEffect(() => {
  const sub = observable.subscribe(setData);
  return () => sub.unsubscribe();
}, []);
```

### WebSocket
```tsx
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = handleMessage;
  return () => ws.close();
}, [url]);
```

### IntersectionObserver / MutationObserver
```tsx
useEffect(() => {
  const observer = new IntersectionObserver(callback);
  observer.observe(ref.current);
  return () => observer.disconnect();
}, []);
```

## 진단

```
# Chrome DevTools Memory
1. Snapshot 찍기 (heap snapshot)
2. 페이지 이동 + 복귀 10회
3. 다시 snapshot
4. Comparison view → Detached HTMLDivElement / 누적된 Listeners

# Performance tab
- Memory checkbox 켜기
- 페이지 이동 시뮬레이션 → JS Heap 우상향 = leak
```

```tsx
// 개발 중 listener 등록 추적
const listeners = new Map();
const origAdd = EventTarget.prototype.addEventListener;
EventTarget.prototype.addEventListener = function(type, handler, ...rest) {
  if (!listeners.has(type)) listeners.set(type, 0);
  listeners.set(type, listeners.get(type) + 1);
  console.log('listener added:', type, listeners.get(type));
  return origAdd.call(this, type, handler, ...rest);
};
```

## 재발 방지

- ESLint `react-hooks/exhaustive-deps` + cleanup 패턴 review
- 모든 `addEventListener` / `setInterval` / `subscribe` 는 cleanup 의무 (PR 룰)
- Memory leak 회귀: Cypress / Playwright 가 페이지 이동 후 heap 증가 검증

## 관련

- [[frontend]] / [[stale-closure]] / [[service-worker-stale-cache]]
