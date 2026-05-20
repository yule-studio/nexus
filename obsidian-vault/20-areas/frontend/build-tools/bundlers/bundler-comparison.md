---
title: "번들러 비교 — Vite vs Webpack vs Turbopack vs esbuild"
kind: knowledge
project: frontend
status: current
tags: [frontend, build-tools, vite, webpack, turbopack, comparison]
---

# 번들러 비교

**[[../build-tools|↑ Build tools hub]]**

---

## 1. 한눈에

| 도구 | Dev 속도 | Build 속도 | 생태계 | 권장 |
| --- | --- | --- | --- | --- |
| **Vite** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 넓음 | SPA / 대부분 |
| **Webpack** | ⭐⭐ | ⭐⭐ | 가장 넓음 | 기존 / 복잡한 케이스 |
| **Turbopack** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Next 만 | Next 14+ |
| **esbuild** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 좁음 | 라이브러리 |
| **Rollup** | N/A | ⭐⭐⭐⭐ | 좁음 | 라이브러리 |
| **Parcel** | ⭐⭐⭐⭐ | ⭐⭐⭐ | 좁음 | 시제품 |

---

## 2. Vite — 모던 표준

### 작동 원리
- Dev: 브라우저 native ESM + esbuild 로 dependencies 사전 번들 + on-demand 변환
- Build: Rollup

### 장점
- 빠른 dev start (수 초)
- 즉각 HMR
- 설정 간단
- React/Vue/Svelte/Solid 다 지원

### 단점
- esbuild 가 일부 신문법 안 됨 (드물게)
- 큰 monorepo 에서 한계

### 사용 시점
새 SPA / 학습용 / 대부분.

---

## 3. Webpack — 클래식

### 작동 원리
모든 모듈을 미리 번들링 → bundle.js.

### 장점
- 가장 오래됨, 모든 케이스 plugin 존재
- 복잡한 빌드 시나리오 가능
- enterprise / legacy 지원

### 단점
- 느림 (수십 초 ~ 분)
- 설정 복잡 (수백 줄도 흔함)
- 학습 곡선 큼

### 사용 시점
이미 webpack 으로 짠 큰 프로젝트 유지보수. 새 프로젝트엔 비추.

### CRA (Create React App)
Webpack 기반. 2024년 deprecated. **Vite 로 마이그레이션** 권장.

---

## 4. Turbopack — Next 의 새 도구

### 작동 원리
Rust 로 작성된 incremental bundler. Webpack 의 후속 (Vercel 이 만들었음).

### 장점
- Rust 라 매우 빠름
- Next 15+ 기본 dev (`--turbopack`)
- HMR 압도적

### 단점
- Next 전용 (다른 프레임워크 안 됨)
- 아직 일부 기능 부족 (build 는 webpack 도 같이)

### 사용 시점
Next 14+. 그냥 `next dev --turbopack`.

---

## 5. esbuild

### 작동 원리
Go 로 작성. 가장 빠른 번들러.

### 장점
- 압도적 빌드 속도
- 단순

### 단점
- 일부 신문법 / TS 기능 부족
- React HMR / 기타 frontend 기능 부족

### 사용 시점
- 라이브러리 빌드
- Vite 내부에서 dependency 번들링
- 단순 도구 / CLI

---

## 6. Rollup

### 작동 원리
ESM 친화 번들러. Vite 의 build 도 Rollup.

### 장점
- ESM 최적 (tree-shake 잘 됨)
- 라이브러리 / npm package 에 표준
- 결과 깔끔 (debug 쉬움)

### 단점
- Dev 서버 X
- Application 빌드는 Vite 권장

### 사용 시점
npm 패키지 만들 때.

---

## 7. SWC (Babel 대체)

### 작동 원리
JS/TS → JS (구버전) 트랜스파일러. Rust.

### Babel 대비
- 10~70배 빠름
- TS 같이 지원

### 사용 시점
- Next 의 내부 트랜스파일러 (자동)
- 직접 사용은 드뭄

---

## 8. Bun (실험적)

### 작동 원리
JS runtime + bundler + package manager 통합. Zig 기반.

### 장점
- 매우 빠름 (Node 보다 4배)
- npm 호환

### 단점
- 신규 — 검증 진행 중
- 일부 패키지 호환성

### 사용 시점
2026 시점에서 production 사용은 조심. 실험은 OK.

---

## 9. 어떤 걸 골라야?

### 새 프로젝트
- **SPA / 정적** → Vite
- **풀스택 / SSR** → Next + Turbopack
- **라이브러리** → Vite library mode 또는 tsup

### 기존 프로젝트
- **CRA → Vite 마이그레이션** 권장
- **Webpack 잘 동작** → 그대로 유지

### 학습용
- 어떤 도구든 OK. Vite 가 가장 부담 적음.

---

## 10. 마이그레이션 가이드 요약

### CRA → Vite
```bash
# 1. CRA 의존성 제거
npm uninstall react-scripts

# 2. Vite 설치
npm install -D vite @vitejs/plugin-react

# 3. index.html 을 root 로 이동
mv public/index.html .
# %PUBLIC_URL% → /
# %REACT_APP_X% → import.meta.env.VITE_X

# 4. vite.config.ts 생성
# 5. scripts 수정
"dev": "vite",
"build": "tsc && vite build",

# 6. 환경변수 REACT_APP_ → VITE_
```

자세히 https://vitejs.dev/guide/migration.html

### Webpack → Vite
복잡 — 큰 프로젝트면 점진 마이그레이션 (e.g. monorepo 새 app 부터).

---

## 11. 빌드 시간 측정

```bash
time npm run build
```

### 일반적 (medium 프로젝트)
- esbuild: 1s
- Vite: 2~5s
- Turbopack: 1~3s
- Webpack: 30s ~ 분

---

## 12. 다음 단계

1. **[[../vite/vite-basics]]** — Vite 깊이
2. **[[code-splitting]]** — 분할 / lazy 패턴

---

연결: [[../build-tools|↑ hub]] · [[../vite/vite-basics]]
