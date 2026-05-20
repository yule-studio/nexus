---
title: "Next.js 배포 — Vercel / 환경변수 / 도메인"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, deploy, vercel]
---

# 배포 (Vercel 중심)

**[[../nextjs|↑ Next.js hub]]**

> Next 만든 회사 Vercel 이 가장 매끄러움. 5 분 setup.

---

## 1. 가장 빠른 배포 — GitHub 연동

### Step 1 — 코드를 GitHub 에
```bash
cd my-next-app
git init
git add .
git commit -m "first commit"
gh repo create my-next-app --public --source=. --push
```

(또는 GitHub 웹에서 repo 만들고 push)

### Step 2 — Vercel 가입
1. https://vercel.com → "Continue with GitHub"
2. 권한 승인

### Step 3 — 프로젝트 import
1. "Add New..." → "Project"
2. GitHub repo 목록에서 `my-next-app` 선택
3. "Import"
4. 자동 감지 (Next.js) → "Deploy"
5. 1~2 분 후 라이브: `https://my-next-app.vercel.app`

### Step 4 — 이후 자동
`git push origin main` → Vercel 이 자동 빌드 + 배포.
PR 만들면 preview URL 자동 생성.

---

## 2. CLI 로 배포

```bash
npm install -g vercel
cd my-next-app
vercel
```

대화형 설정:
- Set up and deploy? → Y
- Which scope? → 본인 계정
- Link to existing? → N
- Project name? → my-next-app
- Directory? → ./
- Override settings? → N

배포 후 URL 출력.

### production 배포
```bash
vercel --prod
```

---

## 3. 환경변수

### 로컬 — `.env.local`
```
DATABASE_URL=postgres://...
API_KEY=secret
NEXT_PUBLIC_API_URL=https://api.example.com
```

### Vercel 대시보드
1. 프로젝트 → Settings → Environment Variables
2. KEY / VALUE 입력
3. Environment 선택:
   - Production
   - Preview (PR 배포)
   - Development (`vercel dev`)
4. Save

### 보안
- `NEXT_PUBLIC_` 시작 = 브라우저에 노출 (안전한 값만)
- 그 외 = 서버에서만 (API key, DB URL 등)

### 갱신 후
환경변수 바꾸면 → "Redeploy" 클릭 (또는 다음 push).

---

## 4. 도메인 연결

### Vercel 대시보드
1. 프로젝트 → Settings → Domains
2. "Add" → 도메인 입력 (예: `myapp.com`)
3. DNS 설정 안내 따름:
   - A record → `76.76.21.21`
   - 또는 CNAME → `cname.vercel-dns.com`
4. SSL 자동 발급 (Let's Encrypt)

### 서브도메인
- `www.myapp.com` → CNAME `cname.vercel-dns.com`
- `app.myapp.com` → 같음

### 자동 redirect
- `www` → 비-`www` 또는 반대로 자동 처리

---

## 5. preview deploy (PR 마다)

### 자동
GitHub PR 만들면 → Vercel 이 자동으로 preview URL 만듦
- `https://my-next-app-git-feat-xxx-username.vercel.app`

### 활용
- PR 에 자동 코멘트로 preview URL
- 디자이너 / PM 이 직접 확인
- merge 전 검토

---

## 6. build 시간 / 비용

Vercel 무료 (Hobby):
- 100GB 데이터 (월)
- 100GB-hours 빌드 시간
- 무제한 사이트
- 개인용 한정

Pro 플랜 ($20/month):
- 1TB 데이터
- 협업
- 비밀번호 보호 등

대부분 개인 프로젝트엔 무료 충분.

---

## 7. 다른 배포 옵션

### Cloudflare Pages
- 무료, 글로벌 CDN, 빠름
- Next 지원 (Pages Functions 로)

### Netlify
- Vercel 비슷, Next 지원
- 무료 plan 넉넉

### AWS Amplify
- AWS 통합 필요할 때
- 좀 복잡

### Docker / VPS (Linode, DigitalOcean)
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["npm", "start"]
```

```bash
docker build -t my-next-app .
docker run -p 3000:3000 my-next-app
```

### Self-host
```bash
npm run build
npm run start    # production 모드 (포트 3000)
```

nginx / Caddy 로 reverse proxy + SSL.

---

## 8. 빌드 / 환경 별 다른 설정

`next.config.ts`:
```typescript
const nextConfig = {
  // production 만
  ...(process.env.NODE_ENV === "production" && {
    compress: true,
    poweredByHeader: false,
  }),

  // 환경별 baseUrl
  basePath: process.env.NEXT_PUBLIC_BASE_PATH || "",

  // 환경변수에서 URL 가져옴
  env: {
    API_BASE: process.env.API_BASE_URL,
  },
};
```

---

## 9. 빌드 시간 줄이기

### Static generation 활용
`fetch` 에 `cache: "force-cache"` (default) → SSG.

### 큰 페이지는 ISR
```typescript
fetch(url, { next: { revalidate: 3600 } });
```

### dynamic import (코드 분할)
```typescript
import dynamic from "next/dynamic";

const HeavyChart = dynamic(() => import("./HeavyChart"));
```

→ 첫 로드 시 안 가져오고, 화면에 보일 때만.

### 이미지 최적화 — `<Image>`
```tsx
import Image from "next/image";

<Image src="/hero.jpg" alt="" width={1200} height={600} priority />
```

자동 WebP / lazy load / srcset.

---

## 10. 모니터링

### Vercel 대시보드
- Analytics (페이지뷰)
- Speed Insights (Core Web Vitals)
- Logs (Server / Function)

### Vercel 외부
- Sentry (에러)
- LogRocket (사용자 세션)
- PostHog (분석)

---

## 11. 트러블슈팅

### Build 실패
- Vercel 대시보드 → Deployments → 실패한 빌드 → Logs
- 흔한 원인:
  - 환경변수 누락
  - TypeScript 에러
  - 패키지 버전 충돌

### 런타임 에러
- Functions 탭 → 로그
- `console.log` / `console.error` 결과 표시

### 느림 / Timeout
- Server Component 의 fetch 가 오래 걸림
- DB 연결 풀 / 외부 API 지연 확인

---

## 12. 환경별 .env

```
.env                  ← 항상 적용 (commit OK, 비밀 X)
.env.local            ← 로컬만 (git ignore)
.env.development      ← npm run dev
.env.production       ← npm run build / start
```

우선순위: `.env.local` > `.env.{환경}` > `.env`

---

## 13. 보안 체크리스트

- [ ] 환경변수 `NEXT_PUBLIC_` 없는 것만 비밀
- [ ] API Route / Server Action 에 인증 검사
- [ ] CORS 헤더 (외부 도메인 호출 시만)
- [ ] CSP (Content Security Policy) 헤더
- [ ] DB query 의 SQL injection 방지 (ORM 또는 prepared)
- [ ] 사용자 입력 sanitize (XSS 방지)

---

## 14. 함정

| 증상 | 원인 |
| --- | --- |
| 환경변수 undefined | dev 재시작 / Vercel 에 추가 / `NEXT_PUBLIC_` |
| 외부 이미지 안 보임 | `next.config.ts` `images.remotePatterns` |
| 빌드 timeout | 정적 페이지 너무 많음, `generateStaticParams` 줄임 |
| 도메인 SSL 안 됨 | DNS propagation 1~48시간 대기 |
| Vercel 빌드 메모리 부족 | 큰 import 줄이거나 plan upgrade |

---

## 15. 다음 단계

1. **[[../nextjs|↑ hub]]** — 다른 영역 학습
2. **[[../../react/react|React]]** — 더 깊이

---

연결: [[../nextjs|↑ hub]] · [[../api-routes]]
