---
title: "Next.js API Routes — route.ts"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, api, backend, route]
---

# API Routes

**[[nextjs|↑ Next.js hub]]**

> Next 안에서 직접 API endpoint 만들기. 별도 backend 서버 불필요.

---

## 1. 가장 단순한 API

`app/api/hello/route.ts`:
```typescript
export async function GET() {
  return Response.json({ message: "Hello" });
}
```

→ `GET /api/hello` 호출 시 `{ "message": "Hello" }`.

테스트:
```bash
curl http://localhost:3000/api/hello
```

---

## 2. 폴더 구조

```
app/api/
├─ hello/route.ts            → /api/hello
├─ users/
│  ├─ route.ts               → /api/users
│  └─ [id]/route.ts          → /api/users/123
└─ posts/
   ├─ route.ts               → /api/posts
   └─ [id]/
      ├─ route.ts            → /api/posts/123
      └─ comments/route.ts   → /api/posts/123/comments
```

폴더 = URL. `route.ts` 가 endpoint.

---

## 3. HTTP method

각 method 별 함수 export:

```typescript
// app/api/users/route.ts
export async function GET() { ... }
export async function POST(request: Request) { ... }
export async function PUT(request: Request) { ... }
export async function PATCH(request: Request) { ... }
export async function DELETE(request: Request) { ... }
```

method 만 export → 다른 method 요청 시 405 자동.

---

## 4. POST — body 받기

```typescript
export async function POST(request: Request) {
  const body = await request.json();    // JSON
  // const text = await request.text();  // 텍스트
  // const form = await request.formData(); // FormData

  console.log(body.name);

  return Response.json({ created: true });
}
```

테스트:
```bash
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"유찬"}'
```

---

## 5. dynamic route — params

`app/api/users/[id]/route.ts`:
```typescript
export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const user = await db.users.find(id);
  return Response.json(user);
}
```

테스트: `curl http://localhost:3000/api/users/42`

---

## 6. searchParams (쿼리 ?)

```typescript
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const q = searchParams.get("q");
  const limit = Number(searchParams.get("limit") ?? "10");

  const results = await search(q, limit);
  return Response.json(results);
}
```

테스트: `curl 'http://localhost:3000/api/search?q=react&limit=5'`

---

## 7. 응답 종류

### JSON
```typescript
return Response.json({ data: [...] });
```

### 텍스트
```typescript
return new Response("Hello", {
  headers: { "Content-Type": "text/plain" },
});
```

### HTML
```typescript
return new Response("<h1>Hi</h1>", {
  headers: { "Content-Type": "text/html" },
});
```

### Status code
```typescript
return Response.json({ error: "Not found" }, { status: 404 });
return Response.json({ created: true }, { status: 201 });
```

### Redirect
```typescript
return Response.redirect(new URL("/login", request.url), 302);
```

---

## 8. 인증 헤더

```typescript
export async function GET(request: Request) {
  const auth = request.headers.get("authorization");
  if (!auth?.startsWith("Bearer ")) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }
  const token = auth.slice(7);
  const user = await verifyToken(token);
  if (!user) return Response.json({ error: "Invalid" }, { status: 401 });

  return Response.json({ user });
}
```

테스트:
```bash
curl -H "Authorization: Bearer xxx" http://localhost:3000/api/me
```

---

## 9. cookies

```typescript
import { cookies } from "next/headers";

export async function GET() {
  const cookieStore = await cookies();
  const sessionId = cookieStore.get("session")?.value;

  return Response.json({ sessionId });
}

export async function POST() {
  const cookieStore = await cookies();
  cookieStore.set("session", "abc123", {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 60 * 60 * 24 * 7,    // 7 days
  });
  return Response.json({ ok: true });
}
```

---

## 10. DB 연결

```typescript
// app/api/users/route.ts
import { db } from "@/lib/db";

export async function GET() {
  const users = await db.query("SELECT * FROM users");
  return Response.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const result = await db.query(
    "INSERT INTO users (name) VALUES ($1) RETURNING *",
    [body.name]
  );
  return Response.json(result.rows[0], { status: 201 });
}
```

---

## 11. CORS (외부 호출)

기본은 같은 origin 만. 외부 호출 허용 시:

```typescript
export async function GET() {
  const data = await fetch("...").then(r => r.json());
  return Response.json(data, {
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, POST",
    },
  });
}

export async function OPTIONS() {
  return new Response(null, {
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    },
  });
}
```

---

## 12. 에러 처리

```typescript
export async function GET() {
  try {
    const data = await riskyOperation();
    return Response.json(data);
  } catch (error) {
    console.error(error);
    return Response.json(
      { error: error.message ?? "Server error" },
      { status: 500 }
    );
  }
}
```

---

## 13. middleware — 전역 처리

`middleware.ts` (프로젝트 root):
```typescript
import { NextResponse, type NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("session")?.value;

  if (!token && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/api/private/:path*"],
};
```

→ 매 요청 전 실행. 인증 / 리다이렉트 / 헤더 추가 / 로깅 등.

---

## 14. Route Handler 의 캐싱

기본은 dynamic (매 요청마다 호출). 정적으로 만들려면:

```typescript
export const dynamic = "force-static";

export async function GET() {
  const data = await fetchExpensive();
  return Response.json(data);
}
```

→ 빌드 시 한 번만 실행, 결과 정적 파일.

---

## 15. API vs Server Action — 언제?

| | API Route | Server Action |
| --- | --- | --- |
| 외부에서 호출 (third-party) | ✅ | ❌ |
| 모바일 앱 / 다른 frontend | ✅ | ❌ |
| 같은 앱 내 form 제출 | ⭕ | ✅ (권장) |
| 같은 앱 내 데이터 fetch | ⭕ | ⭕ |
| Webhook 수신 | ✅ | ❌ |

자체 앱 안 form / mutation → **Server Action** 권장.
외부 client (모바일, 다른 도메인 fetch) → **API Route**.

---

## 16. 자주 만드는 패턴

### A. CRUD users
```typescript
// app/api/users/route.ts
export async function GET() {
  return Response.json(await db.users.findAll());
}

export async function POST(req: Request) {
  const body = await req.json();
  const user = await db.users.create(body);
  return Response.json(user, { status: 201 });
}
```

```typescript
// app/api/users/[id]/route.ts
export async function GET(_, { params }) {
  const { id } = await params;
  const user = await db.users.find(id);
  if (!user) return Response.json({ error: "Not found" }, { status: 404 });
  return Response.json(user);
}

export async function PUT(req, { params }) {
  const { id } = await params;
  const body = await req.json();
  const updated = await db.users.update(id, body);
  return Response.json(updated);
}

export async function DELETE(_, { params }) {
  const { id } = await params;
  await db.users.delete(id);
  return new Response(null, { status: 204 });
}
```

### B. 외부 API proxy
```typescript
export async function GET() {
  const res = await fetch("https://external.api.com/data", {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` },
  });
  const data = await res.json();
  return Response.json(data);
}
```

→ API key 가 서버에만 노출 (브라우저는 모름).

### C. Webhook 수신
```typescript
export async function POST(req: Request) {
  const sig = req.headers.get("x-webhook-signature");
  const body = await req.text();
  if (!verify(sig, body, process.env.WEBHOOK_SECRET)) {
    return Response.json({ error: "Invalid" }, { status: 401 });
  }
  // 처리
  return Response.json({ received: true });
}
```

---

## 17. 함정

| 증상 | 원인 |
| --- | --- |
| 405 Method Not Allowed | export 한 method 없음 |
| 404 | route.ts 위치 / 파일명 오타 |
| body 가 undefined | `await request.json()` 안 함 |
| `params` 가 Promise | Next 15+ — `await params` |
| middleware 가 안 돔 | matcher 패턴 안 맞음 |
| CORS 에러 | OPTIONS handler 누락 |

---

## 18. 다음 단계

1. **[[deploy/vercel]]** — 배포
2. **[[data/server-actions]]** — Server Actions 비교

---

연결: [[nextjs|↑ hub]] · [[data/server-components]]
