---
title: "Next.js App Router — 라우팅 깊이"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, routing, app-router]
---

# App Router 라우팅

**[[../nextjs|↑ Next.js hub]]**

> 폴더가 URL. page.tsx / layout.tsx 가 화면. 모든 기능을 자세히.

---

## 1. 폴더 → URL 매핑

```
app/
├─ page.tsx                  → /
├─ about/page.tsx            → /about
├─ blog/
│  ├─ page.tsx               → /blog
│  └─ [slug]/page.tsx        → /blog/anything
├─ shop/
│  ├─ [...path]/page.tsx     → /shop/a/b/c
│  └─ [[...optional]]/page.tsx → /shop/a (또는 그냥 /shop)
└─ docs/
   ├─ (group)/
   │  └─ intro/page.tsx      → /docs/intro
   └─ layout.tsx             → /docs/* 의 wrapper
```

---

## 2. page.tsx — 페이지 정의

```tsx
// app/about/page.tsx
export default function About() {
  return (
    <main>
      <h1>About</h1>
    </main>
  );
}
```

- default export 필수
- 함수 이름 자유

### async page (Server Component)
```tsx
export default async function Posts() {
  const posts = await fetch("/api/posts").then(r => r.json());
  return <ul>{posts.map(p => <li>{p.title}</li>)}</ul>;
}
```

---

## 3. layout.tsx — 공유 wrapper

### Root layout (필수)
```tsx
// app/layout.tsx
export const metadata = {
  title: "내 사이트",
};

export default function RootLayout({ children }) {
  return (
    <html lang="ko">
      <body>
        <header>...</header>
        {children}
        <footer>...</footer>
      </body>
    </html>
  );
}
```

### 중첩 layout
```tsx
// app/blog/layout.tsx
export default function BlogLayout({ children }) {
  return (
    <div className="blog-shell">
      <aside>Sidebar</aside>
      <main>{children}</main>
    </div>
  );
}
```

→ `/blog`, `/blog/post1`, `/blog/post2` 모두 BlogLayout 안.

### 핵심 동작
페이지 이동 시 **layout 은 다시 렌더 안 함** — children 만 바뀜. 성능 좋음.

---

## 4. Link — 페이지 이동

```tsx
import Link from "next/link";

<Link href="/about">About</Link>
<Link href="/blog/post-1">Post 1</Link>
<Link href={`/posts/${id}`}>{title}</Link>

{/* 외부 링크 */}
<a href="https://example.com" target="_blank">외부</a>
```

### prefetch
`<Link>` 가 화면에 보이면 백그라운드에서 자동 prefetch → 클릭 시 즉시 표시.

### 명시적 비활성
```tsx
<Link href="/heavy-page" prefetch={false}>Heavy</Link>
```

---

## 5. 페이지 이동 (코드)

`useRouter` hook (Client Component):
```tsx
"use client";
import { useRouter } from "next/navigation";

export default function LoginButton() {
  const router = useRouter();

  function handleClick() {
    router.push("/dashboard");
    // router.replace("/login");  // 뒤로가기에 안 남김
    // router.back();             // 뒤로
    // router.refresh();          // 현재 페이지 다시 fetch
  }

  return <button onClick={handleClick}>로그인</button>;
}
```

---

## 6. 동적 라우트 — `[param]`

### 한 개
```
app/posts/[id]/page.tsx
```

`/posts/1` 매칭:
```tsx
export default async function Post({ params }) {
  const { id } = await params;   // Next 15+ params 는 Promise
  return <h1>Post {id}</h1>;
}
```

Next 14 이전:
```tsx
export default function Post({ params }: { params: { id: string } }) {
  return <h1>Post {params.id}</h1>;
}
```

### catch-all — `[...name]`
```
app/docs/[...slug]/page.tsx
```

`/docs/a/b/c` 매칭 → `slug = ["a", "b", "c"]`.

### 선택적 — `[[...name]]`
```
app/shop/[[...path]]/page.tsx
```

`/shop` (path 없음), `/shop/a`, `/shop/a/b` 모두 매칭.

자세히 [[dynamic-routes]].

---

## 7. group — `(name)`

URL 영향 X. 폴더 정리 + 별도 layout 부여 용.

```
app/
├─ (marketing)/
│  ├─ layout.tsx          ← 마케팅 페이지용
│  ├─ about/page.tsx      → /about
│  └─ pricing/page.tsx    → /pricing
└─ (app)/
   ├─ layout.tsx          ← 앱 페이지용
   ├─ dashboard/page.tsx  → /dashboard
   └─ settings/page.tsx   → /settings
```

URL 은 `/about`, `/dashboard` 등. 그룹 이름은 안 보임.

---

## 8. loading.tsx — 자동 로딩

```tsx
// app/posts/loading.tsx
export default function Loading() {
  return <p>로딩 중...</p>;
}
```

`/posts` 의 Server Component 가 데이터 fetching 중일 때 자동 표시. React Suspense 활용.

---

## 9. error.tsx — 에러 처리

```tsx
// app/posts/error.tsx
"use client";   // 필수

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>에러: {error.message}</h2>
      <button onClick={() => reset()}>다시 시도</button>
    </div>
  );
}
```

해당 라우트 (`/posts/*`) 에서 에러 나면 자동 표시.

---

## 10. not-found.tsx — 404

```tsx
// app/not-found.tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <div>
      <h1>404 - 페이지 없음</h1>
      <Link href="/">홈으로</Link>
    </div>
  );
}
```

`notFound()` 함수로 트리거:
```tsx
import { notFound } from "next/navigation";

export default async function Post({ params }) {
  const { id } = await params;
  const post = await fetchPost(id);
  if (!post) notFound();        // 404 page 로 점프
  return <h1>{post.title}</h1>;
}
```

---

## 11. template.tsx vs layout.tsx

| | layout | template |
| --- | --- | --- |
| 페이지 이동 시 | 유지 (재마운트 X) | 매번 재마운트 |
| state 유지 | ✅ | ❌ (초기화) |
| 사용 시점 | 사이드바 / nav 등 공유 | 페이지마다 시작 애니메이션 등 |

대부분 `layout` 사용. `template` 는 거의 안 씀.

---

## 12. 평행 라우트 — `@slot`

같은 layout 안 여러 page 동시 렌더.

```
app/
├─ layout.tsx
├─ @nav/page.tsx           ← 사이드 nav slot
├─ @main/page.tsx          ← 본문 slot
└─ page.tsx
```

```tsx
// layout.tsx
export default function Layout({ nav, main, children }) {
  return (
    <div>
      <aside>{nav}</aside>
      <main>{main || children}</main>
    </div>
  );
}
```

고급 기능. 처음엔 안 써도 됨.

---

## 13. intercepting routes — `(...)folder`

같은 url 인데 다른 layout (modal 처리).

```
app/
├─ photos/[id]/page.tsx       ← /photos/1 정상 페이지
└─ @modal/(.)photos/[id]/page.tsx  ← /photos/1 이지만 모달로
```

고급. 인스타그램 / Pinterest 사진 클릭 시 모달 UX.

---

## 14. metadata — SEO

### 정적
```tsx
export const metadata = {
  title: "My Page",
  description: "...",
  openGraph: {
    title: "My Page",
    images: ["/og.png"],
  },
  twitter: { card: "summary" },
};
```

### 동적
```tsx
export async function generateMetadata({ params }) {
  const { id } = await params;
  const post = await fetchPost(id);
  return {
    title: post.title,
    description: post.summary,
  };
}
```

---

## 15. searchParams — 쿼리스트링

`/search?q=hello&page=2`:
```tsx
export default async function SearchPage({ searchParams }) {
  const { q, page } = await searchParams;
  return <h1>검색: {q} (page {page})</h1>;
}
```

`useSearchParams` hook (Client):
```tsx
"use client";
import { useSearchParams } from "next/navigation";

const params = useSearchParams();
const q = params.get("q");
```

---

## 16. 함정

| 증상 | 원인 |
| --- | --- |
| 페이지가 404 | `page.tsx` 가 default export 안 함, 폴더 위치 |
| layout 이 적용 안 됨 | layout 만 있고 page 가 없으면 layout 활성 안 됨 |
| URL 의 `[id]` 가 undefined | params 가 Promise (Next 15+) — `await params` 필요 |
| useState / onClick 에러 | Server Component. `"use client"` 추가 |
| Link 가 새로 고침 | `<a>` 썼음. `<Link>` 로 |

---

## 17. 다음 단계

1. **[[dynamic-routes]]** — 동적 라우트 깊이
2. **[[loading-and-error]]** — loading / error 패턴
3. **[[../data/server-components]]** — Server vs Client

---

연결: [[../nextjs|↑ hub]] · [[../getting-started/project-structure]]
