---
title: "Next.js 첫 앱 만들기"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, beginner, tutorial]
---

# Next.js 첫 앱

**[[../nextjs|↑ Next.js hub]]**

> `create-next-app` 으로 5 분 만에 띄우기.

---

## 1. Node.js 설치 확인

```bash
node --version    # v18 이상 필요
npm --version
```

없으면 https://nodejs.org → LTS 다운로드.

---

## 2. 프로젝트 생성

```bash
cd ~/local-dev
npx create-next-app@latest my-next-app
```

**옵션 선택** (다 Yes / 기본 추천):
- TypeScript? → **Yes** (강추)
- ESLint? → **Yes**
- Tailwind CSS? → Yes (또는 No — 본인 hompage 처럼 순수 CSS 도 OK)
- `src/` 디렉토리? → Yes
- App Router? → **Yes** (반드시 — Pages Router 는 옛 방식)
- Turbopack? → Yes (빠른 dev 서버)
- import alias (`@/*`)? → Yes

→ 자동 npm install + 폴더 생성.

---

## 3. dev 서버 시작

```bash
cd my-next-app
npm run dev
```

```
▲ Next.js 15.x
- Local:        http://localhost:3000
```

브라우저에서 http://localhost:3000 → "Welcome to Next.js" 화면.

저장하면 자동 새로고침 (Hot Reload).

---

## 4. 폴더 구조

```
my-next-app/
├─ src/
│  └─ app/                ← App Router 의 핵심
│     ├─ favicon.ico
│     ├─ globals.css      ← 전역 CSS
│     ├─ layout.tsx       ← 전체 wrapper
│     └─ page.tsx         ← / 페이지
├─ public/                 ← 정적 파일 (이미지 등)
├─ next.config.ts          ← Next 설정
├─ tsconfig.json
├─ eslint.config.mjs
└─ package.json
```

자세히 [[project-structure]].

---

## 5. 첫 페이지 수정

`src/app/page.tsx` 열기:
```tsx
export default function Home() {
  return (
    <main>
      <h1>안녕, Next.js!</h1>
      <p>이건 내 첫 Next 페이지.</p>
    </main>
  );
}
```

저장 → 브라우저가 자동 새로고침.

---

## 6. 두 번째 페이지 — /about

`src/app/about/` 폴더 만들기 → 그 안에 `page.tsx`:

```tsx
export default function About() {
  return (
    <main>
      <h1>About</h1>
      <p>자기소개 페이지.</p>
    </main>
  );
}
```

→ http://localhost:3000/about 자동 활성. **라우팅 코드 0 줄**.

---

## 7. 링크 — Next 의 `<Link>`

`src/app/page.tsx`:
```tsx
import Link from "next/link";

export default function Home() {
  return (
    <main>
      <h1>안녕, Next.js!</h1>
      <nav>
        <Link href="/about">About 으로</Link>
      </nav>
    </main>
  );
}
```

- `<a>` 대신 `<Link>` — Next 가 페이지 prefetch + 빠른 navigation.
- 외부 링크는 그냥 `<a>` 사용.

---

## 8. 전역 layout

`src/app/layout.tsx`:
```tsx
import "./globals.css";
import Link from "next/link";

export default function RootLayout({ children }) {
  return (
    <html lang="ko">
      <body>
        <header>
          <Link href="/">Home</Link> · <Link href="/about">About</Link>
        </header>
        <main>{children}</main>
        <footer>© 2026</footer>
      </body>
    </html>
  );
}
```

→ 모든 페이지가 이 wrapper 안에 들어감.

자세히 [[../routing/app-router]].

---

## 9. 스타일 (CSS)

### globals.css — 전역
`src/app/globals.css`:
```css
body {
  font-family: -apple-system, sans-serif;
  background: #0b0d0c;
  color: #e6ece8;
  margin: 0;
}

header, main, footer {
  max-width: 720px;
  margin: 0 auto;
  padding: 16px;
}

a { color: #4ade80; text-decoration: none; }
a:hover { text-decoration: underline; }
```

### CSS Module — 특정 컴포넌트만
`src/app/Button.module.css`:
```css
.button {
  background: #4ade80;
  color: white;
  padding: 8px 16px;
  border-radius: 6px;
  border: none;
}
```

```tsx
import styles from "./Button.module.css";

export default function Button({ text }) {
  return <button className={styles.button}>{text}</button>;
}
```

자동으로 클래스명 고유화 → 충돌 X.

---

## 10. 데이터 가져오기 — 첫 fetch

`src/app/page.tsx`:
```tsx
export default async function Home() {
  const res = await fetch("https://jsonplaceholder.typicode.com/posts");
  const posts = await res.json();

  return (
    <main>
      <h1>최근 글</h1>
      <ul>
        {posts.slice(0, 5).map((p) => (
          <li key={p.id}>{p.title}</li>
        ))}
      </ul>
    </main>
  );
}
```

### 핵심
- `async function` — Server Component (기본)
- 서버에서 fetch 실행 → 완성된 HTML 보냄
- `useState` / `useEffect` 불필요

자세히 [[../data/data-fetching]].

---

## 11. dynamic route — `/posts/:id`

`src/app/posts/[id]/page.tsx`:
```tsx
export default async function Post({ params }) {
  const { id } = await params;
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
  const post = await res.json();

  return (
    <main>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </main>
  );
}
```

→ `/posts/1`, `/posts/42` 등 모두 매칭.

자세히 [[../routing/dynamic-routes]].

---

## 12. build / start

```bash
npm run build       # production 빌드
npm run start       # production 모드 실행
```

`npm run dev` 는 개발용 (Hot Reload), `npm run start` 는 실제 배포 모드.

---

## 13. 배포 (Vercel)

```bash
npm install -g vercel
vercel              # 최초 실행 시 로그인 + 프로젝트 설정
```

또는 GitHub push → Vercel 이 자동 감지. 자세히 [[../deploy/vercel]].

---

## 14. 다음 단계

1. **[[project-structure]]** — App Router 폴더 구조
2. **[[../routing/app-router]]** — 라우팅 자세히
3. **[[../data/server-components]]** — Server vs Client Component

---

연결: [[../nextjs|↑ hub]] · [[what-is-nextjs]] · [[project-structure]]
