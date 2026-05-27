---
title: "bkurs-fe — Build / CI"
kind: handover
status: stable
created_at: 2026-05-21T23:20:00+09:00
tags: [bkurs-fe, handover, build, vite, typescript, ci]
related:
  - "[[bkurs-fe]]"
  - "[[directory-structure]]"
home_hub: bkurs-fe
---

# bkurs-fe — Build / CI

> Vite 8 + TypeScript 6 strict + ESLint flat config + GitHub Actions. `tsBuildInfoFile` 위치 함정,
> alias `@/*` 동기화, lint 룰 — 사실상 "왜 빌드 / typecheck 가 깨지지?" 의 95% 가 여기.

## 1. 한 장 요약

| 단계 | 명령 | 결과 |
| --- | --- | --- |
| dev | `npm run dev` | Vite HMR — http://localhost:5173 |
| typecheck | `npm run typecheck` | `tsc -b --noEmit --force` (.cache/tsbuildinfo) |
| build | `npm run build` | `tsc -b && vite build` → `dist/` |
| lint | `npm run lint` / `lint:fix` | ESLint flat config |
| format | `npm run format` / `format:check` | Prettier |
| preview | `npm run preview` | `dist/` 로컬 미리보기 |

CI: GitHub Actions `ci.yml` 의 `quality` job — Node 20 + lint + typecheck + build.

## 2. 의존성 핵심

```
react              19.2.4
react-dom          19.2.4
react-router-dom    7.14
@tanstack/react-query  5.97
jotai               2.19
antd                6.3       # ConfigProvider + ko_KR locale
axios               1.15
recharts            3.8       # ScoreRadar
@supabase/supabase-js 2.104   # 점진 제거 ([[state-and-auth#6]])

typescript           ~6.0      # strict + verbatimModuleSyntax
vite                 8.0
tailwindcss          3.4
eslint               9.39      # flat config
prettier             3.8
```

Node 20 (`.nvmrc`). engine: `>=20`.

## 3. Vite 설정 (`vite.config.ts`)

```ts
export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
  server:  { host: '0.0.0.0', port: 5173 },
  build:   { outDir: 'dist', sourcemap: false, chunkSizeWarningLimit: 1500 },
})
```

- `@/*` → `src/*` — `tsconfig.app.json` 의 paths 와 **반드시 일치**.
- `host: 0.0.0.0` — Docker / 다른 디바이스에서도 접속 가능.
- `sourcemap: false` — 번들 크기 절약 (prod). 디버깅 필요 시 `true` 또는 `hidden`.
- `chunkSizeWarningLimit: 1500` — 1.4MB 번들 경고 회피용 (lazy split 도입 전).

## 4. TypeScript 설정 (`tsconfig.app.json`)

핵심:

```json
{
  "tsBuildInfoFile": "./.cache/tsbuildinfo/tsconfig.app.tsbuildinfo",
  "target": "es2023",
  "lib": ["ES2023", "DOM", "DOM.Iterable"],
  "module": "esnext",
  "moduleResolution": "bundler",
  "allowImportingTsExtensions": true,
  "verbatimModuleSyntax": true,
  "moduleDetection": "force",
  "noEmit": true,
  "jsx": "react-jsx",

  "strict": true,
  "noUnusedLocals": true,
  "noUnusedParameters": true,
  "noFallthroughCasesInSwitch": true,
  "noImplicitReturns": true,
  "erasableSyntaxOnly": true,

  "paths": { "@/*": ["./src/*"] }
}
```

### 4.1 `tsBuildInfoFile` 위치 (.cache)

`./.cache/tsbuildinfo/tsconfig.app.tsbuildinfo` — node_modules **외부**.

**왜 외부?** CI 가 `node_modules/` 를 cache 하면서 안에 tsbuildinfo 두면:
- npm ci 가 node_modules 를 wipe → tsbuildinfo 사라짐 → 매번 full rebuild.
- 또는 cache restore 후 stale tsbuildinfo 가 잘못된 incremental → 미감지 에러.

`.cache/tsbuildinfo/` 로 옮긴 후:
- CI 가 `.cache/tsbuildinfo` 만 별도 cache 가능
- node_modules wipe 와 무관

> .gitignore 에 `.cache/` 포함됨 (또는 `.cache/tsbuildinfo/`). 신경 쓸 필요 없음.

### 4.2 `verbatimModuleSyntax: true` — type-only import 분리 필수

```ts
// ❌ Error: ApiEnvelope is used only as a type
import { ApiEnvelope, type ApiError } from './envelope'

// ✅
import type { ApiEnvelope, ApiError } from './envelope'

// 또는 mix:
import { ApiClientError } from './envelope'
import type { ApiEnvelope, ApiError } from './envelope'
```

axios 도 PR #127 에서 type-only import 분리:

```ts
import axios, { type AxiosInstance, type InternalAxiosRequestConfig } from 'axios'
```

### 4.3 `erasableSyntaxOnly` — class field 초기화 회피

TS 6 의 신규 옵션. `private foo = 1` 같은 class field 초기화를 emit 단계에서 erase 불가능한
형태로 못 쓰게 함. Lombok-like 코드 회피 — 대안: `private foo: number; constructor() { this.foo = 1 }`.

### 4.4 strict 옵션

| 옵션 | 효과 |
| --- | --- |
| `strict` | strictNullChecks + noImplicitAny 등 일괄 |
| `noUnusedLocals` | 사용 안 한 변수 error |
| `noUnusedParameters` | 사용 안 한 함수 인자 error (prefix `_` 로 회피) |
| `noFallthroughCasesInSwitch` | switch fallthrough error |
| `noImplicitReturns` | 모든 path 가 return |

**끄지 말 것**. 회피하려면 `// eslint-disable-next-line` 또는 prefix `_`.

### 4.5 alias `@/*` 동기화

| 파일 | 설정 |
| --- | --- |
| `tsconfig.app.json` | `"paths": { "@/*": ["./src/*"] }` |
| `vite.config.ts` | `alias: { '@': path.resolve(__dirname, './src') }` |

둘 중 한 곳만 바꾸면 빌드는 OK 인데 typecheck 깨짐 (또는 반대). 항상 같이.

## 5. ESLint (`eslint.config.js`)

flat config 형식. 주요 룰:

- TypeScript strict (`@typescript-eslint/recommended`)
- React 17+ jsx runtime (`react/jsx-uses-react` 끔)
- react-hooks (rules-of-hooks + exhaustive-deps)
- react-refresh (HMR 호환)
- Prettier 와 충돌 룰은 `eslint-config-prettier` 가 disable

수정 자주 받는 오류:
- exhaustive-deps — useEffect dep 누락
- no-unused-vars — `_` prefix 또는 ESLint disable
- jsx-no-undef — 컴포넌트 import 누락

## 6. Prettier

`.prettierrc` 의 기본:
- printWidth, semi, singleQuote 등 표준
- 한국어 코멘트 / 문자열은 그대로

CI 는 `format:check` (`prettier --check`) 가 fail 하면 빌드 fail. 로컬에서 `npm run format` 한 번
돌리면 통과.

## 7. GitHub Actions — `ci.yml`

```yaml
on:
  push:    branches: [main, dev, 'release/**']
  pull_request: branches: [main, dev, 'release/**']

jobs:
  quality:
    name: lint + typecheck + build
    runs-on: ubuntu-latest
    steps:
      - actions/checkout@v4
      - actions/setup-node@v4 (with cache: npm)
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck  ← PR #130 에서 `--force` 추가
      - run: npm run build
```

`concurrency.cancel-in-progress: true` — 같은 PR 의 새 push 가 들어오면 이전 run 취소.

### 7.1 typecheck `--force` 의 이유 (#130)

incremental cache 가 stale 일 때 일부 변경을 못 감지 → typecheck 가 false GREEN. `--force` 로
매번 full rebuild. CI 시간 약간 늘지만 정확.

## 8. 자주 마주치는 빌드 오류

| 오류 | 원인 | 해결 |
| --- | --- | --- |
| `Cannot find module '@/...'` | tsconfig paths / vite alias 불일치 | 둘 다 갱신 |
| `'X' is declared but never used` | strict noUnusedLocals | `_` prefix 또는 사용 |
| `Import 'X' is only used as a type` | verbatimModuleSyntax | `import type { X }` |
| `Property 'foo' does not exist on type 'never'` | union narrow 누락 | type guard 추가 |
| `Module 'foo' has no default export` | named import 으로 변경 | (#128 router 사례) |
| `tsbuildinfo`가 stale | 캐시 꼬임 | `rm -rf .cache/tsbuildinfo && npm run typecheck` |

## 9. 빌드 산출물

`npm run build`:
- `dist/index.html` — entry
- `dist/assets/index-{hash}.js` — main bundle (~1.4MB)
- `dist/assets/index-{hash}.css` — Tailwind + AntD reset
- 압축: gzip 380KB

**lazy split (route-level) 미구현** — 후속 ([[bkurs-fe#5]]). 도입 시 chunk 분할 + `<Suspense>`.

## 10. 로컬 / dev / prod 환경

| 환경 | env | BE base URL |
| --- | --- | --- |
| local | `.env.local` | `VITE_API_BASE_URL=http://localhost:8080` (또는 default `/api` + Vite proxy) |
| dev | (배포 시 빌드 env) | `https://api-dev.bkurs.kr/api` |
| prod | secret 주입 | `https://api.bkurs.kr/api` |

`.env.example`:

```
VITE_API_BASE_URL=
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
```

supabase 키는 점진 제거 ([[state-and-auth#6]]).

## 11. dev 서버 동작

```bash
npm run dev
# → http://localhost:5173
```

- HMR — 파일 변경 시 즉시 reload
- proxy 없음 — BE base URL 은 env 로
- StrictMode 활성 (`main.tsx`) — 일부 effect 가 2회 실행 (의도). 디버깅 시 console 중복 정상.

## 12. 배포 (현재 미정)

- dist/ 를 정적 호스팅 (Vercel / Netlify / S3+CloudFront 등) 으로 — 후속.
- 현재 GHCR 같은 image build 는 BE 만 ([[../bkurs-be/stacked-pr-workflow]]).

## cross-link

- [[bkurs-fe]]
- [[directory-structure]] — tsconfig / vite / package.json 위치
- `docs/RELEASE_FLOW.md` (repo) — 배포 흐름 SSoT (후속 작성)
- [[../../memory/feedback_pr_issue_labels_assignees]] — CI 가 통과 후에도 author label 강제
