---
title: "troubleshooting · Hydration mismatch (Next.js / SSR)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, react, nextjs, ssr, hydration]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# Hydration mismatch — `Text content did not match`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

```
Warning: Text content did not match.
  Server: "2026-05-27 14:30"
  Client: "2026-05-27 14:31"
```

또는:
```
Hydration failed because the initial UI does not match what was rendered on the server.
```

- 페이지 로드 후 React tree 가 reset / flicker
- SEO / 첫 paint 손상

## 원인

서버 / 클라이언트 가 다른 결과 렌더:

- `Date.now()` / `Math.random()` — 시간차
- `window.innerWidth` / `navigator.userAgent` — server 없음
- `localStorage` / `sessionStorage` — server 없음
- `new Date().toLocaleString()` — locale / TZ 차이
- 조건부 `typeof window !== 'undefined'`
- 브라우저 확장 (Grammarly 등) 이 DOM 수정

## 해결

### `useEffect` 로 client-only 렌더
```tsx
const [now, setNow] = useState<string | null>(null);

useEffect(() => {
  setNow(new Date().toLocaleString('ko-KR'));
}, []);

return <span>{now ?? ' '}</span>;
```

### `dynamic` import (Next.js) — server 측 skip
```tsx
import dynamic from 'next/dynamic';

const ChartClientOnly = dynamic(() => import('./Chart'), { ssr: false });
```

### `suppressHydrationWarning` — 최후
```tsx
<time suppressHydrationWarning>{new Date().toLocaleString()}</time>
```

→ warning 만 끔. 실제 mismatch 는 그대로. 사용 자제.

### 결정적 데이터 — server 가 timestamp 전달
```tsx
// server component
export default async function Page() {
  const now = await getServerTime();   // server 기준 timestamp
  return <Client now={now} />;
}

// client component
'use client';
export function Client({ now }: { now: string }) {
  return <span>{new Date(now).toLocaleString()}</span>;
}
```

### TZ / locale 명시
```tsx
new Intl.DateTimeFormat('ko-KR', {
  timeZone: 'Asia/Seoul',
  dateStyle: 'short',
  timeStyle: 'short',
}).format(date);
```

server / client 같은 출력 보장.

## Next.js 14+ App Router 특수 케이스

- `cookies()` / `headers()` 는 server-only → client component 에서 직접 사용 X
- `useSearchParams` 는 hydration 후 동작 → fallback `<Suspense>`
- `usePathname` 도 마찬가지

## 진단

```tsx
// React DevTools → Components tab
// → 빨간색으로 mismatch 마킹

// Console 에 hydration warning 의 element / text 추적
```

## 재발 방지

- ESLint: `no-restricted-globals: ['window', 'document']` (server file)
- e2e 테스트 (Playwright) 에서 console error 검사
- CI: build log 에 "Hydration" 키워드 fail

## 관련

- [[frontend]] / [[fetch-strict-mode-duplicate]]
