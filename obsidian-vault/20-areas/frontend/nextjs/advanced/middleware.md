---
title: "Next.js Middleware — 전 요청 처리"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, middleware, auth, edge]
---

# Middleware

**[[../nextjs|↑ Next.js hub]]**

> 모든 요청 / 일부 경로 매칭 시 자동 실행. 인증 / 리다이렉트 / 헤더 / A/B 테스트.

---

## 1. 기본 — `middleware.ts`

프로젝트 root 에 (`app/` 옆):
```typescript
// middleware.ts
import { NextResponse, type NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  console.log("요청 들어옴:", request.url);
  return NextResponse.next();
}
```

→ 모든 요청 (페이지 + API + 정적) 에서 자동 실행.

---

## 2. matcher — 어느 path 만

```typescript
export const config = {
  matcher: ["/dashboard/:path*", "/api/private/:path*"],
};
```

이 패턴 매칭 요청 만 middleware 동작.

### 패턴 종류
```typescript
matcher: [
  "/dashboard",                  // 정확
  "/dashboard/:path*",            // 그 아래 다
  "/((?!api|_next|static).*)",    // api / _next / static 제외
  "/api/((?!auth).*)",            // /api/* 단 /api/auth 제외
],
```

---

## 3. 리다이렉트 / rewrite / 다음 단계

### redirect (URL 변경, 브라우저에 보임)
```typescript
return NextResponse.redirect(new URL("/login", request.url));
```

### rewrite (URL 유지, 다른 페이지 렌더)
```typescript
return NextResponse.rewrite(new URL("/maintenance", request.url));
```

### next (다음 처리로)
```typescript
return NextResponse.next();
```

### 응답 직접 (API style)
```typescript
return new Response("OK", { status: 200 });
```

---

## 4. 인증 패턴

```typescript
import { NextResponse, type NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("session")?.value;
  const isAuthPage = request.nextUrl.pathname.startsWith("/login");

  // 로그인 안 했으면 → /login
  if (!token && !isAuthPage) {
    const url = new URL("/login", request.url);
    url.searchParams.set("from", request.nextUrl.pathname);
    return NextResponse.redirect(url);
  }

  // 이미 로그인 한 사람이 /login → /dashboard
  if (token && isAuthPage) {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/login"],
};
```

---

## 5. 헤더 추가

```typescript
export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  // 보안 헤더
  response.headers.set("X-Frame-Options", "DENY");
  response.headers.set("X-Content-Type-Options", "nosniff");
  response.headers.set("Referrer-Policy", "strict-origin-when-cross-origin");
  response.headers.set("Permissions-Policy", "camera=(), microphone=(), geolocation=()");

  // 사용자 식별
  response.headers.set("X-User-Country", request.geo?.country ?? "Unknown");

  return response;
}
```

---

## 6. Cookies

### 읽기
```typescript
const token = request.cookies.get("session")?.value;
const all = request.cookies.getAll();
```

### 쓰기 (응답에)
```typescript
const response = NextResponse.next();
response.cookies.set("visited", "true", {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  maxAge: 60 * 60 * 24 * 30,    // 30 days
});
```

---

## 7. 지역 (geolocation) — Vercel

```typescript
const { country, city } = request.geo ?? {};

if (country === "KR") {
  return NextResponse.redirect(new URL("/ko", request.url));
}
```

Vercel 에 배포 시 자동 제공. 다른 호스트 X.

---

## 8. A/B 테스트

```typescript
export function middleware(request: NextRequest) {
  let bucket = request.cookies.get("bucket")?.value;

  if (!bucket) {
    bucket = Math.random() < 0.5 ? "A" : "B";
    const response = NextResponse.rewrite(new URL(`/variant-${bucket}`, request.url));
    response.cookies.set("bucket", bucket);
    return response;
  }

  return NextResponse.rewrite(new URL(`/variant-${bucket}`, request.url));
}
```

→ 50/50 분할 → 다음 방문 시 같은 variant.

---

## 9. 다국어 (i18n)

```typescript
export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const hasLocale = ["ko", "en", "ja"].some(
    locale => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );

  if (!hasLocale) {
    const acceptLang = request.headers.get("accept-language") ?? "";
    const lang = acceptLang.startsWith("ko") ? "ko" :
                 acceptLang.startsWith("ja") ? "ja" : "en";

    return NextResponse.redirect(
      new URL(`/${lang}${pathname}`, request.url)
    );
  }
}
```

자세히 [[../i18n/]].

---

## 10. 제한 사항

### 사용 가능
- `URL`, `Headers`, `Cookies`, `Response`
- `fetch` (다른 서버 호출 가능)
- 기본 JS

### 사용 불가
- Node.js API (`fs`, `child_process`)
- 큰 라이브러리 (예: Mongoose) — Edge runtime 한계
- DB 직접 connection (가능은 하지만 권장 X)

→ 무거운 작업은 API Route 로.

---

## 11. Edge runtime

middleware 는 기본 Edge runtime — 가벼움 + 글로벌 빠름.

명시:
```typescript
export const config = {
  runtime: "edge",       // 기본
  matcher: [...],
};
```

`route.ts` 도 edge 가능:
```typescript
// app/api/hello/route.ts
export const runtime = "edge";

export async function GET() {
  return Response.json({ hi: "from edge" });
}
```

---

## 12. 자주 만드는 패턴

### A. 인증 + 다국어
```typescript
export function middleware(request: NextRequest) {
  // 1. 다국어
  const locale = getLocale(request);

  // 2. 인증
  if (!hasValidSession(request) && requiresAuth(request.nextUrl.pathname)) {
    return NextResponse.redirect(new URL(`/${locale}/login`, request.url));
  }

  // 3. 헤더 + cookie
  const response = NextResponse.next();
  response.headers.set("X-Locale", locale);
  return response;
}
```

### B. Rate limiting (간단)
```typescript
import { kv } from "@vercel/kv";

export async function middleware(request: NextRequest) {
  const ip = request.ip ?? "unknown";
  const count = await kv.incr(`rate:${ip}`);
  await kv.expire(`rate:${ip}`, 60);   // 1분

  if (count > 100) {
    return new Response("Too many requests", { status: 429 });
  }

  return NextResponse.next();
}
```

### C. URL 정규화
```typescript
export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // trailing slash 제거
  if (pathname.length > 1 && pathname.endsWith("/")) {
    return NextResponse.redirect(
      new URL(pathname.slice(0, -1), request.url),
      308
    );
  }

  // 소문자 강제
  if (pathname !== pathname.toLowerCase()) {
    return NextResponse.redirect(
      new URL(pathname.toLowerCase(), request.url),
      308
    );
  }
}
```

---

## 13. 디버깅

### 콘솔 로그
```typescript
console.log("middleware:", request.url);
```
→ 서버 터미널 (dev) / Vercel Functions 탭 (prod).

### 응답 헤더로
```typescript
response.headers.set("X-Debug", "abc");
// 브라우저 Network 탭에서 확인
```

---

## 14. 함정

| 증상 | 원인 |
| --- | --- |
| middleware 가 안 돔 | matcher 패턴 안 맞음 |
| 무한 redirect | 같은 path 로 redirect (조건 확인) |
| DB connection 실패 | Edge runtime — Node 만 가능한 lib |
| `request.geo` undefined | Vercel 외 호스트 — 다른 방법 (IP 조회) |
| cookie 가 안 set 됨 | response 가 아니라 request 에 set 시도 |

---

## 15. 다음 단계

1. **[[edge-runtime]]** (예정) — Edge runtime 깊이
2. **[[../routing/app-router]]** — 라우팅 자세히
3. **[[../api-routes]]** — API Route

---

연결: [[../nextjs|↑ hub]] · [[../routing/app-router]]
