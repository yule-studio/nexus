---
title: "Next.js — Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, nextjs, react, hub, beginner]
---

# Next.js — Hub

**[[../frontend|↑ frontend]]**

> React 위에 라우팅 / SSR / SSG / Server Actions / API 등을 얹은 **풀스택 React 프레임워크**. Vercel 이 관리.

---

## 0. 사용법

1. **React 기본** ([[../react/react]]) 을 먼저. Next 는 React 의 상위 호환.
2. App Router (Next 13+) 기준. Pages Router (구) 는 별도 표시.
3. 코드 예시는 그대로 복사해 작동.

---

## 1. 학습 로드맵 (3~4 주)

### Week 1 — 기초
- [[getting-started/what-is-nextjs]] — Next 가 뭐고 React 와 뭐가 다른가
- [[getting-started/first-app]] — `create-next-app` 으로 첫 앱
- [[getting-started/project-structure]] — App Router 폴더 구조

### Week 2 — 라우팅
- [[routing/app-router]] — 폴더 = 경로 / page.tsx / layout.tsx
- [[routing/dynamic-routes]] — `[id]` / `[...slug]`
- [[routing/loading-and-error]] — loading.tsx / error.tsx

### Week 3 — 데이터
- [[data/server-components]] — Server vs Client Component
- [[data/data-fetching]] — fetch 의 캐싱 / revalidation / SSG / SSR / ISR
- [[data/server-actions]] — form action 으로 서버 호출

### Week 4 — 배포 / API
- [[api-routes]] — `route.ts` 로 API 만들기
- [[deploy/vercel]] — Vercel 배포 / 환경변수

---

## 2. 자주 묻는 질문

**Q. React vs Next.js, 어느 걸 배워?**
→ React 의 핵심 (컴포넌트 / hook / state) 를 알아야 Next 도 가능. **React → Next 순서**.

**Q. SSR / SSG / ISR / CSR 뭐가 뭐예요?**
- **CSR** (Client-Side Rendering) — 브라우저가 JS 받고 그림 (React 기본)
- **SSR** (Server-Side Rendering) — 서버가 매 요청마다 HTML 만들어 보냄
- **SSG** (Static Site Generation) — 빌드 시점에 HTML 생성, 이후 그대로 서빙
- **ISR** (Incremental Static Regeneration) — SSG + 주기적으로 갱신
→ Next 는 4 가지 다 지원. 페이지마다 선택 가능.

**Q. App Router vs Pages Router?**
- App Router (Next 13+) — 새 표준. 모든 새 프로젝트는 이걸로.
- Pages Router — 옛 방식. 기존 프로젝트만.
→ **무조건 App Router**.

**Q. Vercel 말고 다른 곳에 배포 가능?**
→ 가능 (AWS, Netlify, Docker 등). 단 Vercel 이 가장 매끄러움 + 무료.

**Q. 본인 hompage 가 Vite + React 인데 Next 가 더 좋아요?**
→ 정적 사이트엔 Vite 면 충분. 동적 데이터 / 서버 로직 / SEO 가 중요하면 Next.

---

## 3. 추천 도구

- **에디터**: VSCode + ESLint 확장
- **연습**: https://stackblitz.com/edit/nextjs
- **공식 문서**: https://nextjs.org/docs (영어, 짧고 명확)
- **튜토리얼**: https://nextjs.org/learn (공식 단계별 강좌)
- **배포**: https://vercel.com (Next 만든 회사. 무료)

---

## 4. 관련

- [[../react/react|↑ React]] — Next 의 기반
- [[../typescript/|↑ TypeScript]] — Next 도 TS 기본 권장
- [[../html-css/html-css|↑ HTML/CSS]] — 모든 React/Next 컴포넌트의 출력
