---
title: "Next.js Loading / Error / Not-Found UI"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, loading, error, ui]
---

# Loading / Error / Not-Found

**[[../nextjs|↑ Next.js hub]]**

> 페이지 로딩 / 에러 / 404 화면을 자동으로 보여주는 특수 파일.

---

## 1. loading.tsx — 자동 로딩 UI

페이지의 데이터 fetching 중에 자동으로 보임.

```tsx
// app/posts/loading.tsx
export default function Loading() {
  return (
    <div className="spinner">
      <p>로딩 중...</p>
    </div>
  );
}
```

→ `/posts` 진입 시 데이터 가져오는 동안 이 화면, 끝나면 page.tsx.

### 어떻게 작동?
React Suspense + Server Component 의 데이터 fetch 가 자동 트리거.

### 중첩 가능
```
app/
├─ loading.tsx                 ← 모든 페이지 fallback
└─ blog/
   ├─ loading.tsx              ← /blog/* 만의 loading
   └─ [slug]/
      ├─ loading.tsx           ← /blog/{slug} 만의 loading
      └─ page.tsx
```

가장 가까운 loading 이 우선.

### 스켈레톤
```tsx
export default function Loading() {
  return (
    <div>
      <div className="skeleton-title" />
      <div className="skeleton-line" />
      <div className="skeleton-line" />
      <div className="skeleton-line short" />
    </div>
  );
}
```

```css
.skeleton-title, .skeleton-line {
  background: linear-gradient(90deg, #eee 0%, #f5f5f5 50%, #eee 100%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 4px;
  margin-bottom: 8px;
}
.skeleton-title { height: 32px; width: 60%; }
.skeleton-line  { height: 14px; width: 100%; }
.skeleton-line.short { width: 70%; }

@keyframes shimmer {
  0%   { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
```

---

## 2. Suspense — 부분 로딩

페이지 일부만 로딩 처리.

```tsx
import { Suspense } from "react";

export default function Page() {
  return (
    <>
      <h1>Dashboard</h1>

      <Suspense fallback={<p>차트 로딩...</p>}>
        <Chart />
      </Suspense>

      <Suspense fallback={<p>목록 로딩...</p>}>
        <RecentList />
      </Suspense>
    </>
  );
}

async function Chart() {
  const data = await fetchChartData();   // 느림
  return <div>{/* 차트 */}</div>;
}

async function RecentList() {
  const items = await fetchRecent();      // 빠름
  return <ul>{items.map(i => <li>{i.title}</li>)}</ul>;
}
```

→ Chart 와 RecentList 가 각자 로딩 / 완료. 빨리 완료된 것부터 보임.

---

## 3. error.tsx — 에러 처리

```tsx
// app/posts/error.tsx
"use client";   // 필수

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>에러 발생</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>다시 시도</button>
    </div>
  );
}
```

### 핵심
- **`"use client"` 필수** — 에러 UI 는 인터랙티브
- `error` — Error 객체
- `reset` — 에러 boundary 리셋 (재렌더 시도)

### 자동 호출 시점
해당 라우트 안에서 발생한 에러 모두.

```tsx
// app/posts/page.tsx
export default async function Posts() {
  const posts = await fetchPosts();   // 실패하면 error.tsx 로
  return <ul>...</ul>;
}
```

### 중첩
```
app/
├─ error.tsx                  ← 글로벌
└─ posts/
   ├─ error.tsx               ← /posts/* 만
   └─ [id]/
      └─ error.tsx            ← /posts/{id} 만
```

---

## 4. global-error.tsx — 최상위 에러

`error.tsx` 는 root layout 안에서만. root layout 자체가 에러 나면 `global-error.tsx`:

```tsx
// app/global-error.tsx
"use client";

export default function GlobalError({ error, reset }) {
  return (
    <html>
      <body>
        <h2>치명적 에러</h2>
        <button onClick={() => reset()}>다시</button>
      </body>
    </html>
  );
}
```

`<html>` + `<body>` 포함 필수 (root 가 에러라 layout 없음).

---

## 5. not-found.tsx — 404

### 자동 (URL 매칭 안 됨)
존재하지 않는 URL → `app/not-found.tsx` 자동.

```tsx
// app/not-found.tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <div>
      <h1>404</h1>
      <p>페이지를 찾을 수 없습니다.</p>
      <Link href="/">홈으로</Link>
    </div>
  );
}
```

### 수동 트리거
```tsx
import { notFound } from "next/navigation";

export default async function Post({ params }) {
  const { id } = await params;
  const post = await fetchPost(id);
  if (!post) notFound();   // not-found.tsx 표시
  return <article>{post.title}</article>;
}
```

### 중첩
```
app/
├─ not-found.tsx              ← 글로벌 404
└─ blog/
   └─ not-found.tsx           ← /blog/* 만의 404
```

---

## 6. 흔한 패턴

### A. 로딩 + 에러 같이
```
app/posts/
├─ page.tsx
├─ loading.tsx
└─ error.tsx
```

자동으로 모두 처리. 추가 코드 X.

### B. 데이터 fetch + early return
```tsx
export default async function User({ params }) {
  const { id } = await params;

  let user;
  try {
    user = await fetchUser(id);
  } catch (e) {
    // error.tsx 로 보내려면 throw 다시
    throw e;
  }

  if (!user) notFound();

  return <main>{user.name}</main>;
}
```

### C. Try with fallback
```tsx
export default async function Page() {
  try {
    const data = await fetchData();
    return <Display data={data} />;
  } catch (e) {
    return <p>임시 메시지 (서비스 점검 중)</p>;
  }
}
```

→ error boundary 안 거치고 자체 처리.

---

## 7. Streaming

Suspense + Server Component 가 자동으로 streaming.

```tsx
import { Suspense } from "react";

export default function Page() {
  return (
    <>
      <Header />     {/* 즉시 */}

      <Suspense fallback={<Skeleton />}>
        <SlowList />  {/* 데이터 오면 추가 */}
      </Suspense>

      <Suspense fallback={<Skeleton />}>
        <Comments />  {/* 데이터 오면 추가 */}
      </Suspense>
    </>
  );
}
```

→ 사용자는 Header 즉시, 데이터 오는 순서대로 점진 표시.

---

## 8. 함정

| 증상 | 원인 |
| --- | --- |
| error.tsx 가 안 보임 | `"use client"` 누락 |
| loading.tsx 가 안 보임 | 페이지가 너무 빨리 끝남 (정상 동작) |
| not-found 자동 안 됨 | `notFound()` 안 호출 + URL 매칭됐음 |
| reset 후 같은 에러 | 에러 원인 그대로면 그대로 발생. 재시도 전 원인 해결 |
| Suspense 가 안 트리거 | Server Component 의 fetch 만 (Client Component 는 별도) |

---

## 9. 다음 단계

1. **[[../data/server-components]]** — Server vs Client Component
2. **[[../data/data-fetching]]** — fetch 캐싱 / revalidation

---

연결: [[../nextjs|↑ hub]] · [[app-router]] · [[dynamic-routes]]
