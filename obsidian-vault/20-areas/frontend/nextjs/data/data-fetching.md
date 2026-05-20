---
title: "Next.js 데이터 Fetching — fetch / 캐싱 / revalidation"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, data, fetch, caching, ssg, ssr, isr]
---

# 데이터 Fetching

**[[../nextjs|↑ Next.js hub]]**

> Server Component 에서 fetch 하는 법. 캐싱 / SSG / SSR / ISR 한 번에 정리.

---

## 1. 기본 — Server Component fetch

```tsx
export default async function Posts() {
  const res = await fetch("https://api.example.com/posts");
  const posts = await res.json();
  return <ul>{posts.map(p => <li>{p.title}</li>)}</ul>;
}
```

### 자동 캐싱
Next 의 `fetch` 는 React 의 fetch 를 확장 → 자동 캐싱 / dedup / revalidate.

기본:
- 같은 URL 같은 옵션 → 메모리 캐시 재사용
- 캐시 만료 없음 (개발자가 명시)

---

## 2. 캐싱 옵션

### `cache: "force-cache"` — 영구 캐시 (default)
```tsx
fetch(url, { cache: "force-cache" });
```

→ 한 번 fetch 면 빌드 끝까지 그 결과 사용 (SSG 동작).

### `cache: "no-store"` — 캐시 안 함
```tsx
fetch(url, { cache: "no-store" });
```

→ 매 요청마다 새로 fetch (SSR 동작).

### `next: { revalidate: 60 }` — 60초마다 재검증
```tsx
fetch(url, { next: { revalidate: 60 } });
```

→ ISR. 60 초 캐시 후 다음 요청 시 재 fetch (백그라운드).

---

## 3. SSG / SSR / ISR — 매 페이지 별 선택

### SSG (빌드 시 1번, 영구)
```tsx
export default async function Page() {
  const data = await fetch(url, { cache: "force-cache" });
  return <div>...</div>;
}
```

→ `npm run build` 시 1번 fetch → HTML 생성 → CDN.

### SSR (매 요청)
```tsx
export default async function Page() {
  const data = await fetch(url, { cache: "no-store" });
  return <div>...</div>;
}
```

→ 매 사용자 요청마다 fetch.

### ISR (시간 기반 재생성)
```tsx
export default async function Page() {
  const data = await fetch(url, { next: { revalidate: 3600 } });   // 1시간
  return <div>...</div>;
}
```

→ 1시간 캐시 후 백그라운드 갱신.

---

## 4. tag 기반 revalidation

```tsx
// 데이터에 tag 붙임
const posts = await fetch(url, { next: { tags: ["posts"] } });
```

```tsx
// 다른 곳에서 tag 만료
import { revalidateTag } from "next/cache";

await createPost(...);
revalidateTag("posts");      // posts tag 캐시 무효화
```

→ 새 글 등록 시 즉시 갱신.

---

## 5. path 기반 revalidation

```typescript
import { revalidatePath } from "next/cache";

revalidatePath("/blog");        // /blog 캐시 무효화
revalidatePath("/blog/[slug]"); // 특정 경로 패턴
revalidatePath("/", "layout");  // root layout 까지 (sub-routes 다 갱신)
```

---

## 6. 여러 fetch — 자동 병렬

```tsx
export default async function Page() {
  const [user, posts, comments] = await Promise.all([
    fetch(`/api/user`).then(r => r.json()),
    fetch(`/api/posts`).then(r => r.json()),
    fetch(`/api/comments`).then(r => r.json()),
  ]);
  return <div>...</div>;
}
```

세 요청 동시 → 가장 느린 것 기준 대기.

---

## 7. dependent fetch (의존 관계)

```tsx
export default async function Page({ params }) {
  const { id } = await params;
  const user = await fetch(`/api/users/${id}`).then(r => r.json());
  const friends = await fetch(`/api/users/${user.id}/friends`).then(r => r.json());
  return <div>...</div>;
}
```

user fetch 후 friends. 순차.

### 더 좋은 패턴 — Suspense + 분리
```tsx
export default function Page({ params }) {
  return (
    <>
      <Suspense fallback={<Skeleton />}>
        <UserCard userId={params.id} />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <UserPosts userId={params.id} />
      </Suspense>
    </>
  );
}

async function UserCard({ userId }) {
  const user = await fetch(...).then(r => r.json());
  return <div>{user.name}</div>;
}
```

각 컴포넌트가 독립 fetch → 빠른 것부터 보임 (streaming).

---

## 8. 서버 환경변수 사용

```tsx
const res = await fetch(`${process.env.API_BASE_URL}/posts`, {
  headers: { Authorization: `Bearer ${process.env.API_TOKEN}` },
});
```

`.env.local`:
```
API_BASE_URL=https://api.example.com
API_TOKEN=secret-token
```

`NEXT_PUBLIC_` 없는 변수는 서버에서만 — 안전.

---

## 9. DB 직접 접근 (Server Component)

```tsx
// app/users/page.tsx
import { db } from "@/lib/db";

export default async function Users() {
  const users = await db.query("SELECT * FROM users");
  return <ul>{users.map(u => <li>{u.name}</li>)}</ul>;
}
```

```typescript
// lib/db.ts
import { Pool } from "pg";

export const db = new Pool({
  connectionString: process.env.DATABASE_URL,
});
```

브라우저 코드에 DB 노출 X (Server Component 라 안전).

---

## 10. error 처리

```tsx
export default async function Page() {
  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();
    return <div>...</div>;
  } catch (e) {
    return <p>에러: {e.message}</p>;
  }
}
```

또는 `throw` 해서 `error.tsx` 로:
```tsx
const res = await fetch(url);
if (!res.ok) throw new Error("Failed to fetch");
```

자세히 [[../routing/loading-and-error]].

---

## 11. force-dynamic / force-static

페이지 단위로 강제:
```tsx
export const dynamic = "force-dynamic";   // 매 요청 (SSR)
export const dynamic = "force-static";    // 빌드 시 1번 (SSG)
export const dynamic = "error";           // 동적 강제 시 에러 (검증용)
```

페이지 안에 fetch 없어도 위 설정으로 모드 결정.

---

## 12. revalidate (페이지 단위)

```tsx
export const revalidate = 60;   // 페이지 60초 마다

export default async function Page() {
  ...
}
```

→ fetch 의 revalidate 와 같은 효과지만 페이지 전체 단위.

---

## 13. 캐싱 정리표

| 패턴 | 사용 시점 | 명령 |
| --- | --- | --- |
| 영구 캐시 (SSG) | 빌드 후 안 바뀜 | `cache: "force-cache"` (default) |
| 캐시 안 함 (SSR) | 매번 fresh | `cache: "no-store"` |
| 시간 기반 (ISR) | 주기적 갱신 | `next: { revalidate: 60 }` |
| 이벤트 기반 | mutation 후 즉시 | `revalidateTag()` / `revalidatePath()` |

---

## 14. 클라이언트에서 fetch (Client Component)

```tsx
"use client";
import { useEffect, useState } from "react";

export default function ClientList() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch("/api/items")
      .then(r => r.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <p>로딩...</p>;
  return <ul>{data.map(i => <li>{i.name}</li>)}</ul>;
}
```

캐싱 / dedup 안 자동. 권장:
- **TanStack Query** (`useQuery`) — 캐싱 자동
- **SWR** — Vercel 의 작은 라이브러리

---

## 15. 외부 이미지 도메인 허용

```typescript
// next.config.ts
const nextConfig = {
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "github.com" },
      { protocol: "https", hostname: "**.example.com" },
    ],
  },
};
```

`<Image>` 사용 시 허용된 도메인만.

---

## 16. 함정

| 증상 | 원인 |
| --- | --- |
| 같은 데이터 두 번 fetch | dedup 안 됨 (다른 URL / 다른 options) |
| 캐시가 안 풀림 | revalidate 안 트리거 / tag/path 매칭 안 됨 |
| dev 에선 항상 동적 | 정상. production 에서 캐시 정책 적용 |
| 새 데이터 안 보임 | `revalidateTag` 호출 안 함 |
| build 가 timeout | 정적 페이지 너무 많음 — `generateStaticParams` 줄이기 |

---

## 17. 다음 단계

1. **[[server-actions]]** — form action 으로 서버 함수 호출
2. **[[../api-routes]]** — API endpoint 만들기

---

연결: [[../nextjs|↑ hub]] · [[server-components]]
