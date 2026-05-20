---
title: "Build Tools — Hub (Vite / Webpack / Turbopack / esbuild / SWC)"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, build-tools, hub, vite]
---

# Build Tools — Hub

**[[../frontend|↑ frontend]]**

> JS/TS/CSS 를 묶어서 실행 가능한 형태로 만드는 도구. 현업 핵심.

---

## 1. 학습 로드맵

- [[vite/vite-basics]] — Vite (현업 표준)
- [[bundlers/bundler-comparison]] — Vite vs Webpack vs Turbopack 차이

---

## 2. 주요 도구 비교

| 도구 | 특징 | 사용 시점 |
| --- | --- | --- |
| **Vite** | 최신, 빠름, ESM dev / Rollup build | 새 SPA / 대부분 |
| **Webpack** | 가장 오래됨, 모든 케이스 가능 | 큰 기업 / legacy |
| **Turbopack** | Vercel, Rust 기반 | Next 14+ 기본 |
| **esbuild** | Go, 매우 빠름 | 라이브러리 / 가벼운 도구 |
| **SWC** | Rust 컴파일러 (Babel 대체) | Next 내부 |
| **Rollup** | ESM 친화 | npm 패키지 |

자세히 [[bundlers/bundler-comparison]].

---

## 3. 현업 권장

| 상황 | 도구 |
| --- | --- |
| React SPA | **Vite** |
| Next.js | **Turbopack** (자동) |
| 라이브러리 | **Vite library mode** / **tsup** |
| 큰 monorepo | **Webpack** 또는 **Turborepo + Vite** |

---

## 4. 핵심 개념

- **Tree Shaking** — 사용 안 하는 export 제거
- **Code Splitting** — 큰 번들을 작게 쪼개기 → 첫 로드 빠름. [[../performance-optimization/bundle/code-splitting]]
- **Minify** — 변수명 짧게 / 공백 제거
- **Source Map** — 빌드 파일 → 원본 .ts 매핑
- **HMR** — 저장 시 부분 reload (state 유지)

---

## 5. 관련

- [[../react/react|↑ React]]
- [[../nextjs/nextjs|↑ Next.js]]
- [[../performance-optimization/performance-optimization|↑ Performance]]
