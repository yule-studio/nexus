---
title: "Next.js 프로젝트 구조"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, structure, app-router, beginner]
---

# Next.js 프로젝트 구조

**[[../nextjs|↑ Next.js hub]]**

> App Router (Next 13+) 기준. 각 폴더 / 파일 역할.

---

## 1. 전체 구조

```
my-next-app/
├─ src/                           ← 모든 소스 (선택 — src 없이도 가능)
│  └─ app/                        ← App Router 의 핵심
│     ├─ layout.tsx               ← 전체 레이아웃 (필수)
│     ├─ page.tsx                 ← / (홈)
│     ├─ globals.css              ← 전역 CSS
│     ├─ favicon.ico
│     │
│     ├─ about/                   ← /about 폴더
│     │  └─ page.tsx
│     │
│     ├─ blog/
│     │  ├─ page.tsx              ← /blog
│     │  └─ [slug]/               ← 동적 (dynamic)
│     │     └─ page.tsx           ← /blog/anything
│     │
│     ├─ (group)/                 ← URL 영향 없는 그룹
│     │  └─ marketing/
│     │     └─ page.tsx
│     │
│     ├─ api/                     ← API endpoint
│     │  └─ users/
│     │     └─ route.ts           ← GET /api/users
│     │
│     ├─ loading.tsx              ← 모든 페이지 로딩 시
│     ├─ error.tsx                ← 모든 페이지 에러 시
│     └─ not-found.tsx            ← 404
│
├─ public/                         ← 정적 파일 (이미지, 폰트 등)
│  ├─ images/
│  └─ favicon.ico
│
├─ next.config.ts                  ← Next 설정
├─ tsconfig.json                   ← TypeScript 설정
├─ eslint.config.mjs               ← ESLint 설정
├─ package.json                    ← 의존성 + 스크립트
├─ .env.local                      ← 환경변수 (git ignore)
└─ .gitignore
```

---

## 2. app/ — 핵심

### 핵심 규칙
- **폴더 이름** = URL 경로
- **page.tsx** = 그 URL 의 화면
- **layout.tsx** = wrapper (선택, 공유)
- 다른 파일은 페이지 X (utils / components 가능)

### 예
```
app/
├─ page.tsx               → /
├─ about/page.tsx         → /about
└─ blog/
   ├─ page.tsx            → /blog
   └─ [slug]/page.tsx     → /blog/anything
```

---

## 3. 특수 파일 이름

App Router 가 인식하는 예약 파일:

| 파일 | 역할 |
| --- | --- |
| `page.tsx` | 그 경로의 화면 |
| `layout.tsx` | 자식 모두 감싸는 wrapper |
| `loading.tsx` | 데이터 로딩 중 보일 화면 |
| `error.tsx` | 에러 시 화면 (`"use client"` 필요) |
| `not-found.tsx` | 404 화면 |
| `route.ts` | API endpoint |
| `template.tsx` | layout 비슷하지만 매번 재마운트 |
| `global-error.tsx` | 최상위 에러 |
| `default.tsx` | parallel routes 의 기본 |

---

## 4. layout.tsx — 공유 wrapper

```tsx
// app/layout.tsx (root)
import "./globals.css";

export default function RootLayout({ children }) {
  return (
    <html lang="ko">
      <body>
        <header>Header</header>
        {children}
        <footer>Footer</footer>
      </body>
    </html>
  );
}
```

root layout 은 **반드시 `<html>` + `<body>` 포함**.

### 중첩 layout
```tsx
// app/blog/layout.tsx
export default function BlogLayout({ children }) {
  return (
    <section>
      <h2>Blog</h2>
      {children}
    </section>
  );
}
```

→ `/blog/*` 페이지들은 RootLayout > BlogLayout > Page 순서로 감쌈.

---

## 5. page.tsx — 화면

```tsx
// app/about/page.tsx
export default function About() {
  return <h1>About</h1>;
}
```

- default export 필수
- async 가능 (Server Component)

### 메타데이터
```tsx
export const metadata = {
  title: "About | My Site",
  description: "사이트 소개",
};

export default function About() {
  return <h1>About</h1>;
}
```

→ HTML `<head>` 자동 설정.

---

## 6. dynamic route — `[param]`

### 한 개
```
app/posts/[id]/page.tsx        → /posts/1, /posts/abc
```

```tsx
export default async function Post({ params }) {
  const { id } = await params;     // Next 15+ 는 await
  return <h1>Post {id}</h1>;
}
```

### 여러 개 (catch-all)
```
app/docs/[...slug]/page.tsx    → /docs/a/b/c
```

```tsx
const { slug } = await params;   // slug = ["a", "b", "c"]
```

### 선택적
```
app/shop/[[...slug]]/page.tsx  → /shop, /shop/a, /shop/a/b
```

---

## 7. group — `(name)` 폴더

URL 에 영향 안 주는 그룹화.

```
app/
├─ (marketing)/
│  ├─ about/page.tsx       → /about
│  └─ pricing/page.tsx     → /pricing
└─ (app)/
   ├─ dashboard/page.tsx   → /dashboard
   └─ settings/page.tsx    → /settings
```

각 그룹 안에 별도 `layout.tsx` 가능 → marketing 페이지엔 큰 헤더, app 페이지엔 사이드바.

---

## 8. api/ — API 만들기

```
app/api/users/route.ts     → GET / POST /api/users
```

```typescript
// route.ts
export async function GET() {
  const users = await db.query("SELECT * FROM users");
  return Response.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  // ...
  return Response.json({ created: true });
}
```

자세히 [[../api-routes]].

---

## 9. private folder — `_name`

언더스코어 시작 폴더는 라우팅 무시. 안에 utils / components 보관.

```
app/
├─ _components/
│  ├─ Card.tsx
│  └─ Navigation.tsx
├─ _lib/
│  └─ db.ts
└─ page.tsx
```

라우터가 무시하므로 `/_components` 같은 URL 안 만들어짐.

---

## 10. public/

정적 파일. URL 에서 `/` 로 접근.

```
public/
├─ images/
│  └─ hero.jpg            → /images/hero.jpg
└─ favicon.ico
```

```tsx
<img src="/images/hero.jpg" alt="hero">
```

### Image 컴포넌트 (최적화)
```tsx
import Image from "next/image";

<Image src="/images/hero.jpg" alt="hero" width={800} height={400} />
```

자동 lazy load, WebP 변환, srcset.

---

## 11. next.config.ts

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // 이미지 외부 URL 허용
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "github.com" },
    ],
  },

  // 환경에 따른 baseUrl
  basePath: process.env.NODE_ENV === "production" ? "/app" : "",

  // 리다이렉트
  async redirects() {
    return [
      { source: "/old", destination: "/new", permanent: true },
    ];
  },
};

export default nextConfig;
```

---

## 12. 환경변수

### `.env.local` (로컬, git ignore)
```
DATABASE_URL=postgres://...
API_KEY=secret
NEXT_PUBLIC_API_URL=https://api.example.com
```

### 사용
```typescript
// 서버에서만
const dbUrl = process.env.DATABASE_URL;

// 클라이언트에서도 — NEXT_PUBLIC_ prefix 필수
const apiUrl = process.env.NEXT_PUBLIC_API_URL;
```

⚠️ `NEXT_PUBLIC_` 없는 변수는 브라우저에 노출 안 됨 (보안).

---

## 13. 추천 폴더 컨벤션

```
src/
├─ app/                  ← 라우팅 (page / layout / route.ts 만)
├─ components/           ← 공유 컴포넌트
│  ├─ ui/                ← Button / Input 등 atomic
│  └─ Header.tsx
├─ lib/                  ← 유틸 (db / auth / api)
│  ├─ db.ts
│  └─ auth.ts
├─ hooks/                ← Custom hooks
├─ types/                ← TypeScript 타입
└─ styles/               ← (선택) tokens.css 등
```

import alias:
```typescript
import Header from "@/components/Header";
import { db } from "@/lib/db";
```

`@/` 는 `src/` 의 alias. `tsconfig.json` 에 설정:
```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## 14. 흔한 실수

| 증상 | 원인 |
| --- | --- |
| 페이지가 안 보임 | `page.tsx` 가 default export 안 함 |
| layout 이 안 적용 | 폴더 안에 `page.tsx` 있어야 (그 폴더의 layout 활성) |
| useState 가 안 됨 | Server Component 임. `"use client"` 추가 |
| 환경변수 undefined | dev 서버 재시작 / `NEXT_PUBLIC_` 누락 |
| 이미지 안 보임 | `next.config.ts` 의 `images.remotePatterns` 안 설정 |

---

## 15. 다음 단계

1. **[[../routing/app-router]]** — 라우팅 자세히
2. **[[../data/server-components]]** — Server vs Client Component

---

연결: [[../nextjs|↑ hub]] · [[first-app]] · [[../routing/app-router]]
