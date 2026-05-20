---
title: "TypeScript 셋업 — tsc / Vite / Next"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, setup, beginner]
---

# TypeScript 셋업

**[[../typescript|↑ TS hub]]**

---

## 1. 빠른 체험 — tsc 만으로

```bash
mkdir my-ts && cd my-ts
npm init -y
npm install --save-dev typescript
npx tsc --init        # tsconfig.json 생성
```

`hello.ts`:
```typescript
const name: string = "유찬";
console.log(`안녕, ${name}`);
```

```bash
npx tsc hello.ts      # hello.js 생성
node hello.js         # 실행
```

`tsc` 가 .ts → .js 로 변환. `noEmit` 옵션이면 검사만.

---

## 2. tsconfig.json — 권장 설정

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",

    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,

    "esModuleInterop": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "allowImportingTsExtensions": false,
    "verbatimModuleSyntax": true,

    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,

    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src/**/*", "vite.config.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### 핵심 옵션
- `strict: true` — 가장 중요. null 체크 / any 금지 등 강제
- `noUncheckedIndexedAccess: true` — `arr[0]` 의 결과를 `T | undefined` 로 (실수 방지)
- `noUnusedLocals/Parameters` — 미사용 변수 경고

---

## 3. Vite + React + TS

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

Vite 가 `.tsx` 자동 처리. `tsconfig.json` 도 자동 생성.

---

## 4. Next.js (TS 기본)

```bash
npx create-next-app@latest my-app
# TypeScript? → Yes
```

자동 셋업. `tsconfig.json`, `next-env.d.ts` 다 생김.

---

## 5. 기존 JS 프로젝트에 TS 도입

### 단계 1 — 의존성
```bash
npm install --save-dev typescript @types/react @types/react-dom @types/node
```

### 단계 2 — tsconfig 생성
```bash
npx tsc --init
```

### 단계 3 — `.js` → `.ts` 점진 변경
한 파일씩. 처음엔 `allowJs: true` 로 섞어 쓰기.

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,    // 일단 JS 검사 X
    ...
  }
}
```

점진 마이그레이션 후 `allowJs: false`.

---

## 6. 자주 쓰는 명령

```bash
npx tsc                  # 전체 빌드 (dist/ 생성)
npx tsc --noEmit         # 검사만 (output X)
npx tsc --watch          # 변경 감지

# 자주 쓰는 npm script
"typecheck": "tsc --noEmit"
"build": "tsc && vite build"
```

---

## 7. VSCode 설정

`.vscode/settings.json`:
```json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": "explicit"
  }
}
```

→ 프로젝트의 TS 버전 사용 + 저장 시 import 자동 정리.

---

## 8. ESLint + TS

```bash
npm install --save-dev @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint
```

`.eslintrc.cjs`:
```javascript
module.exports = {
  parser: "@typescript-eslint/parser",
  parserOptions: { project: "./tsconfig.json" },
  plugins: ["@typescript-eslint"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
  ],
  rules: {
    "@typescript-eslint/no-unused-vars": ["warn", { argsIgnorePattern: "^_" }],
    "@typescript-eslint/consistent-type-imports": "error",
  },
};
```

---

## 9. 자주 만나는 셋업 함정

| 증상 | 원인 |
| --- | --- |
| `Cannot find module` 에러 | `paths` 설정 빠짐 / 재시작 필요 |
| JSX 에러 | `jsx: "react-jsx"` 설정 / `.tsx` 확장자 |
| 외부 라이브러리 타입 X | `@types/lib-name` 설치 또는 `// @ts-expect-error` |
| 빌드는 되는데 타입 에러 | `tsc --noEmit` 별도 실행. Vite 는 빌드 시 타입 안 검사 |
| import 가 .js 가 안 됨 | `verbatimModuleSyntax: true` + `import type` |

---

## 10. type-only import

```typescript
import type { User } from "./types";    // 타입만, 런타임 코드 X
import { type User, createUser } from "./api";   // 섞기
```

`verbatimModuleSyntax: true` 권장.

---

## 11. 다음 단계

1. **[[../types/primitives]]** — 기본 타입
2. **[[../types/objects-interfaces]]** — 객체 / interface

---

연결: [[../typescript|↑ hub]] · [[../../javascript/javascript|JS]]
