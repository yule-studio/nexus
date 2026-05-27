---
title: "troubleshooting · Fetch 중복 발진 — React 18 Strict Mode"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, react, strict-mode, fetch, useeffect]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# Fetch 중복 — React 18 Strict Mode dev 환경

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- dev 환경에서 fetch 가 2회 발진 (Network tab)
- POST 가 2회 → 같은 데이터 2건 생성
- prod 에서는 정상

## 원인

React 18 `<StrictMode>` 가 dev 에서 effect 를 의도적으로 2회 실행 (mount → unmount → mount). 미래 React feature (concurrent rendering, fast refresh 중 state 유지) 를 위한 effect 가 idempotent 한지 검증 목적.

```tsx
useEffect(() => {
  fetch('/api/data');   // dev 에서 2회
}, []);
```

## 해결

### AbortController + cleanup
```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch('/api/data', { signal: controller.signal })
    .then(r => r.json())
    .then(setData)
    .catch(e => {
      if (e.name !== 'AbortError') throw e;
    });

  return () => controller.abort();   // 첫 번째 effect 의 fetch 취소
}, []);
```

→ Strict Mode 의 첫 mount 의 fetch 가 abort, 두 번째 mount 의 fetch 만 남음.

### Data fetching library — 자동 dedupe
```tsx
// TanStack Query
const { data } = useQuery({
  queryKey: ['data'],
  queryFn: () => fetch('/api/data').then(r => r.json()),
});

// SWR
const { data } = useSWR('/api/data', fetcher);
```

→ 같은 key 의 동시 fetch 는 자동 dedupe. cache + revalidation.

### POST 는 useEffect 가 아닌 event handler 에서
```tsx
// Bad
useEffect(() => {
  fetch('/api/orders', { method: 'POST', body: ... });
}, []);

// Good
<button onClick={async () => {
  await fetch('/api/orders', { method: 'POST', body: ... });
}}>Order</button>
```

→ side effect 가 user action 에 trigger. mount 시 자동 호출 X.

또는 idempotency-key:
```tsx
const idempotencyKey = useId();   // 안정적 unique
useEffect(() => {
  fetch('/api/orders', {
    method: 'POST',
    headers: { 'Idempotency-Key': idempotencyKey },
    body: ...
  });
}, []);
```

→ server 가 [[../backend/payment-double-charge]] 의 idempotency 패턴 처리.

### Strict Mode 끄기 — 안 권장
```tsx
// Bad
<App />   // StrictMode 제거

// Good — 유지
<StrictMode>
  <App />
</StrictMode>
```

→ Strict Mode 는 미래 호환성 신호. 끄지 말고 effect 를 idempotent 하게.

## prod 와 dev 의 차이

| | dev | prod |
| --- | --- | --- |
| Strict Mode 2회 mount | ✅ | ❌ |
| HMR / fast refresh 영향 | ✅ | ❌ |
| 의도 | bug 검출 | 정상 동작 |

→ dev 에서 2회 발진은 **버그가 아니라 검증 도구**. prod 에서 0회 / 1회로 동작.

## 진단

```tsx
// useEffect 실행 추적
useEffect(() => {
  console.log('effect run', Date.now());
  return () => console.log('cleanup', Date.now());
}, []);

// React DevTools Profiler — render count
```

## 재발 방지

- 모든 useEffect 에 cleanup return (AbortController 표준)
- Data fetching 은 TanStack Query / SWR — 자동 dedupe
- POST / side-effect 는 user event 에 binding
- 서버 측 idempotency 도 함께 (frontend 신뢰 X)

## 관련

- [[frontend]] / [[stale-closure]] / [[event-listener-leak]] / [[../backend/payment-double-charge]]
