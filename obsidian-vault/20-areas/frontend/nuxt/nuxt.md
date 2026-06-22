---
title: "Nuxt"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - frontend
  - nuxt
---

# Nuxt

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-22 | engineering-agent/tech-lead | 개념 개요 hub 로 재작성 |

**[[../frontend|↑ frontend]]**

> Vue 위에 SSR·라우팅·데이터 페칭·서버 API 를 얹은 메타프레임워크(meta-framework). React 진영의 Next.js 에 대응한다.

## 1. 무엇인가

Nuxt 는 Vue 단독으로는 직접 구성해야 하는 것들 — 서버 렌더링, 파일 기반 라우팅, 데이터 페칭 규약, 서버 API 엔드포인트, 빌드/배포 — 을 한 프레임워크로 묶는다. 핵심 모델은 다음과 같다.

- **렌더링 전략 선택** — Universal(SSR), SPA(CSR), SSG(정적 생성), 그리고 라우트별 하이브리드 렌더링을 설정으로 고른다.
- **파일 기반 라우팅** — `pages/` 폴더 구조가 그대로 URL 라우트가 된다.
- **Nitro 서버 엔진** — `server/api/` 에 둔 핸들러가 백엔드 엔드포인트가 되고, 다양한 배포 타깃(Node, edge, serverless)으로 빌드된다.
- **auto-import** — 컴포넌트·composable(`useFetch`, `useState` 등)을 import 문 없이 쓴다.

즉 Nuxt 는 새로운 렌더링 모델이 아니라, Vue 의 반응성/SFC 위에 "앱 한 채를 끝까지 만들 수 있게 하는 규약과 인프라"를 더한 층이다.

## 2. 왜 / 언제 선택하나

(1) **강점** — SEO·초기 로딩·소셜 미리보기가 중요한 페이지(마케팅, 커머스, 콘텐츠)에서 SSR/SSG 를 직접 배선하지 않아도 된다. 프론트와 가벼운 백엔드(BFF)를 한 레포에서 다루고, 배포 타깃 이식성이 좋다.

(2) **안 맞는데 쓰면** — 인증 뒤에 있는 순수 사내 대시보드처럼 SEO 가 무의미하고 항상 클라이언트 렌더면, SSR 의 복잡도(서버/클라이언트 양쪽에서 도는 코드, hydration 이슈)만 떠안는다. 이럴 땐 Vue SPA 가 단순하다.

(3) **대안 비교**
- **vs Next.js** — Nuxt:Vue = Next:React. 역할·기능 범위가 거의 1:1 대응. 베이스 프레임워크(Vue vs React) 선택이 곧 Nuxt vs Next 선택으로 이어진다.
- **vs Vue 단독** — Vue 단독은 SSR/라우팅/서버 API 를 직접 조립해야 한다. Nuxt 는 그걸 규약으로 제공한다.
- **vs SvelteKit** — SvelteKit 은 Svelte 의 메타프레임워크로 같은 자리이며, 컴파일 기반이라 더 가볍다.

(4) **트레이드오프** — Vue 생태계를 그대로 쓰지만 SSR 특유의 디버깅 난이도(서버/클라 환경 차이, hydration mismatch)가 추가된다. 러닝커브는 Vue + 서버 개념만큼 늘고, 성능은 SSG 시 매우 빠르나 SSR 은 서버 부하를 고려해야 한다.

## 3. 핵심 개념 / 어떻게

```vue
<!-- pages/users/[id].vue  →  /users/:id 라우트 -->
<script setup>
const route = useRoute()
// 서버에서 페칭 후 hydration, 중복 호출 방지까지 처리
const { data: user, pending } = await useFetch(`/api/users/${route.params.id}`)
</script>

<template>
  <div v-if="pending">로딩…</div>
  <h1 v-else>{{ user.name }}</h1>
</template>
```

```ts
// server/api/users/[id].ts  →  Nitro 서버 엔드포인트
export default defineEventHandler((event) => {
  const id = getRouterParam(event, 'id')
  return { id, name: 'Yule' }
})
```

- `useFetch` / `useAsyncData` 는 SSR-aware 데이터 페칭(서버에서 받아 클라로 전달).
- `useState` 는 SSR 안전한 공유 상태(서버/클라 동일 키).
- `nuxt.config.ts` 의 `ssr`, `nitro.preset` 으로 렌더링/배포 타깃을 정한다.

## 4. 적용 맥락

- 공개 웹(랜딩/블로그/커머스/문서)처럼 **SEO·초기 페인트가 매출/노출과 직결**되는 곳에 적합.
- 프론트와 BFF 를 한 곳에서 운영하고 싶을 때.
- **React(Next) 와 갈리는 지점** — 팀이 Vue 면 Nuxt, React 면 Next. 기능 범위는 비슷하므로 베이스 프레임워크 선호가 결정 기준이다. 순수 인증 뒤 SPA 면 Nuxt 대신 Vue 단독을 고려.

## 5. 관련

- [[../vue/vue|↗ Vue (베이스 프레임워크)]]
- [[../react/react|↗ React / Next.js 대응]]
- [[../svelte-sveltekit/svelte-sveltekit|↗ SvelteKit]]
- [[../_common/_common|↗ 렌더링 전략(CSR/SSR/SSG/ISR)]]
- [[../frontend|↑ frontend]]
- 공식 문서 — https://nuxt.com / Nitro https://nitro.build
