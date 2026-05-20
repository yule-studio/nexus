---
title: "Next.js Dynamic Routes — [param] / [...slug] / [[...optional]]"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, routing, dynamic]
---

# Dynamic Routes

**[[../nextjs|↑ Next.js hub]]**

> URL 의 일부를 변수로 받기. 블로그 글 / 사용자 페이지 / 상품 상세 등.

---

## 1. 세 가지 종류

| 패턴 | 매칭 | params |
| --- | --- | --- |
| `[id]` | 한 단계 | `id: "abc"` |
| `[...slug]` | 여러 단계 | `slug: ["a", "b", "c"]` |
| `[[...slug]]` | 여러 + 선택 | `slug: ["a"]` 또는 undefined |

---

## 2. 한 개 — `[id]`

```
app/posts/[id]/page.tsx
```

매칭:
- `/posts/1` → `id: "1"`
- `/posts/hello-world` → `id: "hello-world"`

```tsx
export default async function Post({ params }) {
  const { id } = await params;
  const post = await fetchPost(id);
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}
```

### TypeScript 타입
```typescript
type Params = Promise<{ id: string }>;

export default async function Post({ params }: { params: Params }) {
  const { id } = await params;
  // ...
}
```

---

## 3. 여러 개 — `[...slug]` (catch-all)

```
app/docs/[...slug]/page.tsx
```

매칭:
- `/docs/a` → `slug: ["a"]`
- `/docs/a/b` → `slug: ["a", "b"]`
- `/docs/a/b/c` → `slug: ["a", "b", "c"]`

```tsx
export default async function Docs({ params }) {
  const { slug } = await params;
  const path = slug.join("/");   // "a/b/c"
  const doc = await fetchDoc(path);
  return <main>{doc.content}</main>;
}
```

---

## 4. 선택적 — `[[...slug]]`

```
app/shop/[[...path]]/page.tsx
```

매칭:
- `/shop` → `path: undefined`
- `/shop/electronics` → `path: ["electronics"]`
- `/shop/electronics/phones` → `path: ["electronics", "phones"]`

```tsx
export default async function Shop({ params }) {
  const { path } = await params;
  if (!path) return <AllProducts />;
  return <CategoryProducts path={path} />;
}
```

---

## 5. 여러 동적 segment

```
app/blog/[year]/[slug]/page.tsx
```

`/blog/2026/hello`:
```tsx
const { year, slug } = await params;
```

---

## 6. 정적 생성 — generateStaticParams

빌드 시점에 미리 만들 페이지 목록.

```tsx
// app/posts/[id]/page.tsx
export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map((p) => ({ id: p.id.toString() }));
}

export default async function Post({ params }) {
  const { id } = await params;
  const post = await fetchPost(id);
  return <article>...</article>;
}
```

→ 빌드 시 `/posts/1`, `/posts/2`, ... 모두 정적 HTML 생성 (SSG).

### 자동 fallback
미리 만들지 않은 ID 가 요청 오면:
- 동적 렌더 (default)
- 또는 `notFound()`

```tsx
export const dynamicParams = false;  // 미리 만든 것만, 나머지 404
```

---

## 7. searchParams (쿼리스트링)

`/posts/1?from=home`:
```tsx
export default async function Post({ params, searchParams }) {
  const { id } = await params;
  const { from } = await searchParams;
  return <article data-from={from}>...</article>;
}
```

---

## 8. notFound — 404 트리거

```tsx
import { notFound } from "next/navigation";

export default async function Post({ params }) {
  const { id } = await params;
  const post = await fetchPost(id);
  if (!post) notFound();   // not-found.tsx 로 점프

  return <article>{post.title}</article>;
}
```

`app/not-found.tsx`:
```tsx
export default function NotFound() {
  return <h1>404 - 페이지 없음</h1>;
}
```

또는 라우트별 not-found 가능:
```
app/posts/[id]/not-found.tsx     ← /posts/* 의 404
```

---

## 9. redirect — 다른 페이지로

```tsx
import { redirect } from "next/navigation";

export default async function OldUrl() {
  redirect("/new-url");
}
```

또는 조건부:
```tsx
const user = await getUser();
if (!user) redirect("/login");
```

---

## 10. 패턴 — 블로그

### 구조
```
app/
└─ blog/
   ├─ page.tsx                  ← /blog (전체 목록)
   ├─ [slug]/
   │  ├─ page.tsx               ← /blog/{slug}
   │  └─ not-found.tsx          ← /blog/{없는 slug} 의 404
   └─ tag/[tag]/page.tsx        ← /blog/tag/python
```

### 목록 페이지
```tsx
// app/blog/page.tsx
import Link from "next/link";

export default async function Blog() {
  const posts = await fetchAllPosts();
  return (
    <main>
      <h1>Blog</h1>
      <ul>
        {posts.map(p => (
          <li key={p.slug}>
            <Link href={`/blog/${p.slug}`}>{p.title}</Link>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

### 상세
```tsx
// app/blog/[slug]/page.tsx
import { notFound } from "next/navigation";

export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map(p => ({ slug: p.slug }));
}

export async function generateMetadata({ params }) {
  const { slug } = await params;
  const post = await fetchPost(slug);
  return {
    title: post.title,
    description: post.summary,
  };
}

export default async function Post({ params }) {
  const { slug } = await params;
  const post = await fetchPost(slug);
  if (!post) notFound();

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.date}</p>
      <div dangerouslySetInnerHTML={{ __html: post.html }} />
    </article>
  );
}
```

---

## 11. 패턴 — 사용자 페이지

```
app/users/[username]/page.tsx
```

`/users/codwithyc`:
```tsx
import { notFound } from "next/navigation";

export default async function User({ params }) {
  const { username } = await params;
  const user = await fetchUser(username);
  if (!user) notFound();

  return (
    <main>
      <img src={user.avatar} />
      <h1>{user.name}</h1>
      <p>@{user.username}</p>
    </main>
  );
}
```

---

## 12. 패턴 — 다국어 (i18n)

```
app/[lang]/page.tsx       → /ko, /en, /ja
app/[lang]/blog/page.tsx  → /ko/blog, /en/blog
```

```tsx
export async function generateStaticParams() {
  return [{ lang: "ko" }, { lang: "en" }, { lang: "ja" }];
}

export default async function Page({ params }) {
  const { lang } = await params;
  const t = await loadTranslations(lang);
  return <h1>{t.welcome}</h1>;
}
```

---

## 13. dynamic 가 우선 (정적 vs 동적 충돌)

```
app/posts/featured/page.tsx     ← /posts/featured (정적)
app/posts/[id]/page.tsx         ← /posts/* (동적)
```

`/posts/featured` 요청 시 정적 페이지 우선 (구체적 매칭이 이김).

---

## 14. 함정

| 증상 | 원인 |
| --- | --- |
| params 가 undefined | Next 15+ 는 `await params` |
| 모든 페이지가 같은 데이터 | 동적 segment 이름 잘못 (`[id]` 폴더에서 `params.userId` 같이) |
| catch-all 이 안 매칭 | `[...slug]` 의 `slug` 가 항상 배열 (단일 매칭도) |
| 정적 + 동적 혼합 안 됨 | 같은 깊이에서 정적이 우선. 의도된 동작 |
| generateStaticParams 가 안 호출 | dev 에선 항상 동적. build 시점만 호출 |

---

## 15. 다음 단계

1. **[[loading-and-error]]** — loading / error 패턴
2. **[[../data/server-components]]** — Server vs Client

---

연결: [[../nextjs|↑ hub]] · [[app-router]]
