---
title: "Next.js 가 뭐예요? — React 와의 차이"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, react, beginner]
---

# Next.js 가 뭐예요?

**[[../nextjs|↑ Next.js hub]]**

---

## 1. 한 줄 요약

> **React** = 컴포넌트 라이브러리.
> **Next.js** = React + 라우팅 + 서버 렌더링 + 데이터 fetching + 배포 도구 = **풀스택 프레임워크**.

---

## 2. React 만으로 부족한 점

### React 만 쓰면
- 화면 만들기는 좋음 (컴포넌트)
- 하지만 **여러 페이지** 만들려면 → React Router 따로 설치
- 서버 렌더링 (SSR) 안 됨 → SEO 약함
- 데이터 fetching 패턴 안 정해짐
- 배포 직접 셋업
- API 만들려면 별도 backend 서버

### Next.js 가 다 해줌
- **파일 시스템 라우팅** — 폴더 만들면 자동 경로
- **SSR / SSG / ISR** — 페이지 마다 선택
- **데이터 fetching 표준** — `fetch()` + 캐싱
- **API Routes** — backend 도 같은 프로젝트
- **Vercel 배포** — git push 만으로
- **이미지 / 폰트 / 메타 데이터** 최적화 내장

---

## 3. 같은 화면, 두 가지 방식

### React + Vite (본인 hompage 처럼)
```
src/
├─ App.tsx           ← 라우터 직접 등록
├─ pages/
│  ├─ Home.tsx
│  └─ About.tsx
└─ main.tsx
```

`App.tsx`:
```tsx
import { Routes, Route } from "react-router-dom";
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
</Routes>
```

### Next.js (App Router)
```
app/
├─ page.tsx          ← /  ("Home")
└─ about/
   └─ page.tsx       ← /about
```

라우팅 코드 0 줄. 폴더 = URL.

---

## 4. SSR / SSG / CSR

같은 React 컴포넌트라도 어디서 그리느냐 (rendering) 다름.

### CSR (Client-Side Rendering) — 기본 React
1. 브라우저가 빈 HTML 받음
2. JS 다운로드
3. JS 실행 → React 가 HTML 그림
4. 사용자에게 화면 보임

**문제**: 초기 빈 화면 → 검색엔진이 못 봄 (SEO 약함). 느림.

### SSR (Server-Side Rendering)
1. 서버가 React 컴포넌트 실행해서 완성된 HTML 보냄
2. 브라우저가 그 HTML 즉시 표시 (검색엔진도 OK)
3. JS 로드 후 interactivity 추가

**장점**: 빠른 첫 화면 + SEO.

### SSG (Static Site Generation)
1. **빌드** 시점에 모든 페이지의 HTML 미리 생성
2. CDN 에서 정적 파일로 서빙

**장점**: 매우 빠름. 서버 비용 0.
**단점**: 데이터가 빌드 시점 고정. 자주 바뀌면 안 맞음.

### ISR (Incremental Static Regeneration)
SSG + 일정 시간마다 백그라운드에서 다시 빌드.

```typescript
fetch(url, { next: { revalidate: 60 } });   // 60초마다 갱신
```

---

## 5. Next.js 가 그리는 시나리오 (App Router)

```
사용자 → URL 입력
  ↓
Vercel/서버
  ↓
app/blog/[slug]/page.tsx 매칭
  ↓
Server Component 실행
  ↓ fetch() → DB / API
HTML 생성
  ↓
브라우저로 전송 (이미 데이터 채워진 HTML)
  ↓
JS 로드 → Client Component 만 hydrate
```

→ 첫 화면 빠름 + 검색엔진 친화 + 인터랙티브.

---

## 6. App Router 핵심 — 폴더가 URL

```
app/
├─ page.tsx                    → /
├─ about/
│  └─ page.tsx                 → /about
├─ blog/
│  ├─ page.tsx                 → /blog
│  └─ [slug]/
│     └─ page.tsx              → /blog/anything
├─ projects/
│  ├─ page.tsx                 → /projects
│  └─ [id]/
│     ├─ page.tsx              → /projects/123
│     └─ edit/
│        └─ page.tsx           → /projects/123/edit
└─ layout.tsx                  → 전체 페이지 wrapper
```

자세히 [[../routing/app-router]].

---

## 7. Server vs Client Component (Next 13+)

### Server Component (기본)
- 서버에서만 실행
- `fetch`, DB 직접 호출 가능
- `useState` / `onClick` 같은 브라우저 hook **불가**
- 번들 사이즈 0 (브라우저에 안 보냄)

```tsx
// app/posts/page.tsx
export default async function Posts() {
  const posts = await fetch("/api/posts").then(r => r.json());
  return <ul>{posts.map(p => <li>{p.title}</li>)}</ul>;
}
```

### Client Component (인터랙티브 시)
- 파일 최상단에 `"use client"` 필요
- `useState`, `onClick`, `useEffect` 사용 가능

```tsx
"use client";
import { useState } from "react";

export default function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n+1)}>{n}</button>;
}
```

자세히 [[../data/server-components]].

---

## 8. Next.js 가 적합한 경우

| 상황 | Next.js 추천? |
| --- | --- |
| 블로그 / 마케팅 사이트 | ✅✅✅ (SSG / ISR 빛남) |
| e-commerce | ✅✅✅ (SSR + SEO) |
| Dashboard (인증 후) | ✅ (React + Vite 도 OK) |
| 작은 정적 사이트 (hompage 같은) | ⭕ (Vite 도 충분) |
| Mobile app | ❌ (React Native) |
| Game | ❌ (게임 엔진) |
| 작은 학습용 | ❌ (Vite 가 더 간단) |

---

## 9. Vercel 배포 — git push 하나로

1. https://vercel.com 가입 (GitHub 로)
2. "New Project" → GitHub repo 선택
3. 자동 감지 → "Deploy"
4. 끝. `https://your-app.vercel.app` 라이브
5. 이후 `git push` 마다 자동 재배포

자세히 [[../deploy/vercel]].

---

## 10. Next vs 다른 React 프레임워크

| 도구 | 특징 |
| --- | --- |
| **Next.js** | 가장 인기, Vercel, App Router, SSR/SSG/ISR/Server Actions |
| **Remix** | Form 중심 / RSC 도입 / Shopify 인수 |
| **Astro** | 마크다운 + 다양한 프레임워크 섞기 / 블로그 강함 |
| **Vite + React Router** | 가장 단순 / SPA 만 |
| **Gatsby** | 옛 SSG 강자 (요즘 Next 에 밀림) |

처음엔 Next 하나에 집중 추천.

---

## 11. 다음 단계

1. **[[first-app]]** — `create-next-app` 으로 첫 앱
2. **[[project-structure]]** — App Router 폴더 구조

---

연결: [[../nextjs|↑ hub]] · [[first-app]] · [[../../react/react|React]]
