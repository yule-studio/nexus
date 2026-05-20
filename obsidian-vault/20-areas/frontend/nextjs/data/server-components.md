---
title: "Next.js Server vs Client Component"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, server-components, react, rsc]
---

# Server vs Client Component

**[[../nextjs|↑ Next.js hub]]**

> Next 13+ 의 핵심. 모든 컴포넌트가 둘 중 하나.

---

## 1. 한 줄 요약

| | Server Component (기본) | Client Component |
| --- | --- | --- |
| 실행 | 서버에서만 | 브라우저에서 |
| `useState` / `onClick` | ❌ | ✅ |
| `fetch` / DB 직접 | ✅ | ⭕ (외부 API 만) |
| 번들 크기 | 0 (안 보냄) | 포함 |
| 표시 | 기본 | `"use client"` 첫 줄 |

---

## 2. Server Component (기본)

```tsx
// app/posts/page.tsx
export default async function Posts() {
  const posts = await fetch("/api/posts").then(r => r.json());
  return (
    <ul>
      {posts.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}
```

### 가능
- `async` 함수
- `await fetch()`
- DB / file system 직접 접근
- 환경변수 (`process.env.SECRET`) — 클라이언트에 노출 X

### 불가능
- `useState`, `useEffect`, `useRef` 등 React hook
- `onClick`, `onChange` 같은 이벤트
- `window`, `document` 같은 브라우저 API

---

## 3. Client Component

파일 최상단에 `"use client"`:

```tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

### 가능
- 모든 React hook
- 이벤트 핸들러
- 브라우저 API

### 제한
- `async` 컴포넌트 X (`useEffect` 안에서 fetch)
- 번들에 포함됨 → 무거운 라이브러리 import 시 폭발

---

## 4. 언제 어느 걸?

### Server Component (기본)
- 정적 / 자주 안 바뀜
- 데이터 fetch 만
- 큰 라이브러리 (markdown 파서, syntax highlighter) — 서버에서만 돌리면 클라이언트 번들 0
- DB / 환경변수 사용

### Client Component
- `useState`, `useReducer` 필요
- 클릭 / 입력 이벤트
- 브라우저 API (`localStorage`, `geolocation`)
- 3rd party library 가 hook 사용 (예: framer-motion)

### 원칙
**최대한 Server. 인터랙티브 한 잎 (leaf) 만 Client.**

```
[Page (Server)]
  ├─ [Header (Server)]
  ├─ [PostList (Server)]
  │   └─ [LikeButton (Client)]   ← 작은 leaf 만 Client
  └─ [Footer (Server)]
```

---

## 5. Server → Client 데이터 전달

Server 에서 fetch → Client 컴포넌트에 props 로:

```tsx
// app/page.tsx (Server)
import LikeButton from "./LikeButton";

export default async function Page() {
  const post = await fetchPost();
  return (
    <article>
      <h1>{post.title}</h1>
      <LikeButton postId={post.id} initialLikes={post.likes} />
    </article>
  );
}
```

```tsx
// LikeButton.tsx (Client)
"use client";
import { useState } from "react";

export default function LikeButton({ postId, initialLikes }) {
  const [likes, setLikes] = useState(initialLikes);
  return (
    <button onClick={() => setLikes(likes + 1)}>
      ♥ {likes}
    </button>
  );
}
```

→ Client component 가 props 받음. JSON-serializable 만 (함수 / Symbol / Date 등 제한).

---

## 6. Client 안 Server X — 단, children 으로 패턴

```tsx
// ServerInside.tsx (Server)
export default function ServerInside() {
  return <h1>Server!</h1>;
}
```

```tsx
// ClientWrapper.tsx (Client)
"use client";
import { useState } from "react";

export default function ClientWrapper({ children }) {
  const [open, setOpen] = useState(true);
  if (!open) return null;
  return (
    <div>
      <button onClick={() => setOpen(false)}>닫기</button>
      {children}    {/* 여기에 Server Component 들어가도 됨 */}
    </div>
  );
}
```

```tsx
// page.tsx (Server)
import ClientWrapper from "./ClientWrapper";
import ServerInside from "./ServerInside";

export default function Page() {
  return (
    <ClientWrapper>
      <ServerInside />
    </ClientWrapper>
  );
}
```

→ Client 가 children 으로 Server 받음. 직접 import 는 안 됨, props 는 OK.

---

## 7. context — Server 에서는 못 씀

```tsx
// ❌ Server 에서
import { useTheme } from "./context";  // context hook 못 씀
```

context 가 필요한 영역은 다 Client 로:

```tsx
"use client";
import { ThemeContext } from "./context";

export default function Provider({ children }) {
  return <ThemeContext.Provider value={...}>{children}</ThemeContext.Provider>;
}
```

```tsx
// app/layout.tsx (Server)
import ThemeProvider from "./ThemeProvider";

export default function Layout({ children }) {
  return (
    <html>
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  );
}
```

---

## 8. fetch 차이

### Server Component 의 fetch
```tsx
const res = await fetch("https://api.example.com/data");
const data = await res.json();
```

- Next 가 자동 캐싱
- 한 번 호출하면 빌드 / 요청마다 결과 메모리에
- `cache: "force-cache" / "no-store"` 옵션

### Client Component 의 fetch
```tsx
"use client";
import { useEffect, useState } from "react";

export default function ClientList() {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch("/api/data").then(r => r.json()).then(setData);
  }, []);
  return <ul>{data?.map(...)}</ul>;
}
```

- 브라우저에서 호출
- 캐싱은 브라우저 / Service Worker
- TanStack Query 같은 라이브러리로 자주 관리

---

## 9. 패턴 — 데이터 + 인터랙션 조합

```
app/posts/[id]/
├─ page.tsx          ← Server (데이터 fetch)
├─ LikeButton.tsx    ← Client (좋아요)
└─ CommentForm.tsx   ← Client (댓글 입력)
```

```tsx
// page.tsx
import LikeButton from "./LikeButton";
import CommentForm from "./CommentForm";

export default async function Post({ params }) {
  const { id } = await params;
  const post = await fetchPost(id);
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <LikeButton postId={id} initialLikes={post.likes} />
      <CommentForm postId={id} />
    </article>
  );
}
```

---

## 10. 흔한 실수

### ❌ Server 에서 useState
```tsx
import { useState } from "react";

export default function Page() {
  const [n, setN] = useState(0);   // 에러
  ...
}
```

→ `"use client"` 추가.

### ❌ Client 에서 async 컴포넌트
```tsx
"use client";

export default async function Page() {   // 안 됨
  const data = await fetch(...);
  ...
}
```

→ `useEffect` 안 또는 TanStack Query.

### ❌ Server 에서 onClick
```tsx
export default function Btn() {
  return <button onClick={...}>눌러</button>;  // 에러
}
```

→ `"use client"` 추가.

### ❌ Client → Server import
```tsx
"use client";
import ServerComp from "./Server";   // 안 됨

export default function Client() {
  return <ServerComp />;
}
```

→ children prop 패턴 (§ 6) 사용.

### ❌ 비-serializable props
```tsx
<ClientComp onLoad={() => ...} />   // 함수 props 도 OK
<ClientComp date={new Date()} />     // Date 도 OK 사실은 (next 14+)
<ClientComp symbol={Symbol()} />     // ❌
```

함수, Date 는 사실 OK. Symbol, class instance 등은 X.

---

## 11. 디버깅

### 어떤 게 Server / Client 인지 확인
- 파일 첫 줄 `"use client"` 있나?
- 콘솔 로그 — Server 는 터미널, Client 는 브라우저 콘솔

### 빌드 결과 사이즈
```
Route (app)             Size  First Load JS
/                      150 B          90 kB
/posts                  2 kB         100 kB
/about                  1 kB          90 kB
```

`First Load JS` 가 너무 크면 Client Component / heavy library 의심.

---

## 12. 함정

| 증상 | 원인 |
| --- | --- |
| `useState 가 undefined` | `"use client"` 누락 |
| `window 가 정의되지 않음` | Server 에서 브라우저 API 호출 |
| context value 가 undefined | Provider 가 Server 라 context 작동 X |
| 환경변수가 undefined | `NEXT_PUBLIC_` 없는 변수는 client 에 X |
| `async function` 컴포넌트가 안 됨 | Client component 라서. `useEffect` |

---

## 13. 다음 단계

1. **[[data-fetching]]** — fetch 캐싱 / revalidation
2. **[[server-actions]]** — form 으로 서버 호출

---

연결: [[../nextjs|↑ hub]] · [[../getting-started/what-is-nextjs]]
