---
title: "Next.js 다국어 (i18n)"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, i18n, localization]
---

# 다국어 (i18n)

**[[../nextjs|↑ Next.js hub]]**

> ko / en / ja 같이 여러 언어 지원.

---

## 1. App Router 의 공식 가이드

App Router 는 i18n 내장 없음. middleware + dynamic segment 직접 구현.

```
app/
├─ [lang]/
│  ├─ layout.tsx
│  ├─ page.tsx              → /ko, /en
│  └─ blog/
│     └─ page.tsx           → /ko/blog, /en/blog
└─ middleware.ts
```

---

## 2. middleware — 자동 redirect

```typescript
// middleware.ts
import { NextResponse, type NextRequest } from "next/server";

const LOCALES = ["ko", "en", "ja"];
const DEFAULT = "ko";

function getLocale(request: NextRequest) {
  const accept = request.headers.get("accept-language") ?? "";
  for (const locale of LOCALES) {
    if (accept.startsWith(locale)) return locale;
  }
  return DEFAULT;
}

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 이미 locale 있음
  const hasLocale = LOCALES.some(
    (l) => pathname.startsWith(`/${l}/`) || pathname === `/${l}`
  );
  if (hasLocale) return;

  // 없으면 redirect
  const locale = getLocale(request);
  return NextResponse.redirect(
    new URL(`/${locale}${pathname}`, request.url)
  );
}

export const config = {
  matcher: ["/((?!_next|api|.*\\.[\\w]+$).*)"],
};
```

---

## 3. dynamic segment

`app/[lang]/page.tsx`:
```tsx
export default async function Home({
  params,
}: {
  params: Promise<{ lang: string }>;
}) {
  const { lang } = await params;
  const t = await loadTranslations(lang);

  return <h1>{t.welcome}</h1>;
}

export async function generateStaticParams() {
  return [{ lang: "ko" }, { lang: "en" }, { lang: "ja" }];
}
```

→ 빌드 시 각 언어 페이지 SSG.

---

## 4. 번역 파일

```
src/i18n/
├─ ko.json
├─ en.json
└─ ja.json
```

```json
// ko.json
{
  "welcome": "환영합니다",
  "login": "로그인",
  "logout": "로그아웃",
  "blog": {
    "title": "블로그",
    "readMore": "더 보기"
  }
}
```

### loader
```typescript
// src/i18n/loader.ts
import "server-only";

const loaders = {
  ko: () => import("./ko.json").then((m) => m.default),
  en: () => import("./en.json").then((m) => m.default),
  ja: () => import("./ja.json").then((m) => m.default),
};

export async function loadTranslations(lang: string) {
  const load = loaders[lang as keyof typeof loaders] ?? loaders.ko;
  return load();
}
```

---

## 5. 컴포넌트에서 사용

### Server Component
```tsx
export default async function Page({ params }) {
  const { lang } = await params;
  const t = await loadTranslations(lang);

  return (
    <>
      <h1>{t.welcome}</h1>
      <a href={`/${lang}/login`}>{t.login}</a>
    </>
  );
}
```

### Client Component — context
```tsx
"use client";
import { createContext, useContext } from "react";

const I18nContext = createContext<Translations | null>(null);

export function I18nProvider({ children, t }) {
  return <I18nContext.Provider value={t}>{children}</I18nContext.Provider>;
}

export function useT() {
  const t = useContext(I18nContext);
  if (!t) throw new Error("useT outside provider");
  return t;
}
```

```tsx
// layout
const t = await loadTranslations(lang);
<I18nProvider t={t}>{children}</I18nProvider>

// child Client
function MyButton() {
  const t = useT();
  return <button>{t.login}</button>;
}
```

---

## 6. 링크 — locale 유지

```tsx
import Link from "next/link";

<Link href={`/${lang}/blog`}>{t.blog.title}</Link>
```

또는 helper:
```typescript
// src/i18n/Link.tsx
"use client";
import NextLink, { LinkProps } from "next/link";

export function Link({ href, lang, ...rest }: LinkProps & { lang: string }) {
  const localized = `/${lang}${href.toString()}`;
  return <NextLink href={localized} {...rest} />;
}
```

---

## 7. switching language

```tsx
"use client";
import { useRouter, usePathname } from "next/navigation";

export function LanguageSwitcher() {
  const router = useRouter();
  const pathname = usePathname();

  function switchTo(lang: string) {
    const newPath = pathname.replace(/^\/[a-z]{2}/, `/${lang}`);
    router.push(newPath);
  }

  return (
    <select onChange={(e) => switchTo(e.target.value)}>
      <option value="ko">한국어</option>
      <option value="en">English</option>
      <option value="ja">日本語</option>
    </select>
  );
}
```

---

## 8. 번역 라이브러리

직접 구현 외에 라이브러리:
- **next-intl** — Next 친화, 가장 인기
- **next-i18next** — Pages Router 위주
- **react-i18next** — 범용
- **formatjs / react-intl** — 복수형 / 날짜 등 완전 ICU

### next-intl 예
```bash
npm install next-intl
```

```typescript
// src/i18n.ts
import { getRequestConfig } from "next-intl/server";

export default getRequestConfig(async ({ locale }) => ({
  messages: (await import(`./messages/${locale}.json`)).default,
}));
```

```tsx
// page.tsx
import { useTranslations } from "next-intl";

export default function Page() {
  const t = useTranslations("blog");
  return <h1>{t("title")}</h1>;
}
```

---

## 9. 복수형 / 변수

### 단순 (직접)
```typescript
t.greet.replace("{name}", "유찬");
```

### ICU (라이브러리)
```json
{
  "items": "{count, plural, =0 {없음} one {1개} other {#개}}"
}
```

```tsx
t("items", { count: 3 });   // "3개"
t("items", { count: 0 });   // "없음"
```

---

## 10. 날짜 / 숫자 / 통화

### Intl API (브라우저 내장)
```typescript
new Intl.DateTimeFormat("ko-KR", {
  year: "numeric", month: "long", day: "numeric",
}).format(new Date());
// "2026년 5월 20일"

new Intl.NumberFormat("ko-KR", {
  style: "currency", currency: "KRW",
}).format(1000);
// "₩1,000"

new Intl.RelativeTimeFormat("ko").format(-3, "day");
// "3일 전"
```

라이브러리 없이도 강력. 권장.

---

## 11. SEO — hreflang

```tsx
export async function generateMetadata({ params }) {
  const { lang } = await params;
  return {
    alternates: {
      canonical: `/${lang}`,
      languages: {
        ko: "/ko",
        en: "/en",
        ja: "/ja",
      },
    },
  };
}
```

→ Google 이 언어 별 페이지 인식.

---

## 12. 자주 만나는 함정

| 증상 | 원인 |
| --- | --- |
| middleware 무한 redirect | static 파일 제외 안 함 (config matcher) |
| Client component 가 번역 안 됨 | context 또는 next-intl |
| 번역 파일 크기 폭증 | 페이지 별 번역 분리 + dynamic import |
| URL 일관성 깨짐 | Link 컴포넌트 wrapper 사용 |

---

## 13. 다음 단계

1. **[[../advanced/middleware]]** — 다국어 middleware 깊이
2. **[[../routing/app-router]]** — dynamic segment

---

연결: [[../nextjs|↑ hub]] · [[../advanced/middleware]]
