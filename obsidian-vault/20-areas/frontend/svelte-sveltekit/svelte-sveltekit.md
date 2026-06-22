---
title: "Svelte + SvelteKit"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - frontend
  - svelte-sveltekit
---

# Svelte + SvelteKit

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-22 | engineering-agent/tech-lead | 개념 개요 hub 로 재작성 |

**[[../frontend|↑ frontend]]**

> 반응성을 런타임이 아니라 컴파일 타임에 처리하는 컴파일러 기반 프레임워크. SvelteKit 은 그 위의 메타프레임워크.

## 1. 무엇인가

Svelte 는 "프레임워크를 빌드 단계에서 사라지게 한다"는 발상이다.

- **컴파일 타임 반응성** — React/Vue 는 브라우저에서 도는 런타임이 상태 변화를 감지하지만, Svelte 는 컴파일러가 `.svelte` 컴포넌트를 분석해 "이 상태가 바뀌면 이 DOM 을 업데이트"하는 평범한 JS 를 생성한다. 런타임 프레임워크 코드가 거의 없어 번들이 작다.
- **runes (Svelte 5)** — `$state`, `$derived`, `$effect` 룬으로 반응성을 명시 선언한다. (예전 버전의 암묵적 `let` 반응성을 대체.)
- **SvelteKit** — 파일 기반 라우팅, SSR/SSG/CSR, `load` 함수 기반 데이터 페칭, 폼 액션, 서버 엔드포인트를 제공하는 공식 메타프레임워크(Nuxt/Next 자리).

## 2. 왜 / 언제 선택하나

(1) **강점** — 같은 화면을 만들 때 코드가 가장 짧은 축이고, 런타임이 얇아 번들/초기 로딩이 가볍다. 보일러플레이트가 적어 작은 팀·빠른 프로토타입·성능 민감 위젯에 생산성이 높다. 스코프 CSS 가 컴포넌트에 내장돼 스타일링 도구가 따로 덜 필요하다.

(2) **안 맞는데 쓰면** — 생태계가 React 대비 작아, 검증된 컴포넌트/통합 라이브러리·채용 풀·트러블슈팅 자료가 부족하다. 대형 조직에서 "필요한 라이브러리가 Svelte 용으로 없다" 또는 "Svelte 개발자 채용이 어렵다"는 운영 리스크가 생긴다. 전용 `.svelte` 문법이라 React/JSX 자산을 재활용하기 어렵다.

(3) **대안 비교**
- **vs React** — Svelte 가 코드량·번들·입문 친화성에서 앞서고, React 가 생태계·채용·자료에서 압도. React 의 재렌더/메모이제이션 고민이 Svelte 엔 거의 없다.
- **vs Solid** — 둘 다 VDOM 을 버린 고성능 진영. Svelte 는 컴파일 타임 처리(전용 문법), Solid 는 런타임 signal(JSX). Svelte 가 더 간결, Solid 가 JSX/조합성 친화.
- **vs Vue** — Vue 도 SFC 와 반응성을 쓰지만 런타임 VDOM 을 유지. Svelte 는 컴파일로 그 런타임을 제거.

(4) **트레이드오프** — 번들/성능/코드량은 최상위, 러닝커브는 입문은 쉬우나 runes·SvelteKit 규약 학습 필요, 생태계는 작은 편. "작고 빠르고 적은 코드" 대 "큰 생태계와 채용 안정성"의 맞교환.

## 3. 핵심 개념 / 어떻게

```svelte
<script>
  let count = $state(0);                 // 반응형 상태 (Svelte 5 runes)
  let double = $derived(count * 2);      // 파생 값
  $effect(() => console.log(count));     // 부수효과
</script>

<button on:click={() => count++}>{count} / {double}</button>

<style>
  button { color: tomato; }  /* 이 컴포넌트에만 적용 (scoped) */
</style>
```

```js
// SvelteKit: src/routes/users/[id]/+page.server.js
export async function load({ params, fetch }) {
  const res = await fetch(`/api/users/${params.id}`);
  return { user: await res.json() }; // 페이지에 data 로 전달, SSR 대응
}
```

- `$state`/`$derived`/`$effect` 룬으로 반응성을 선언. 직접 대입(`count++`)이 곧 업데이트.
- SvelteKit 의 `load` 는 서버/클라 모두에서 페칭, `+page.server.js` 는 서버 전용.
- `src/routes/` 폴더 구조가 라우트.

## 4. 적용 맥락

- 마케팅/콘텐츠 사이트, 임베드 위젯, 성능·번들이 중요한 제품, 빠른 MVP 에 적합.
- SSR/SEO 가 필요하면 Svelte 단독이 아니라 **SvelteKit** 을 쓴다.
- **React 와 갈리는 지점** — 적은 코드·작은 번들·빠른 입문을 원하면 Svelte, 검증된 생태계·채용·라이브러리 호환을 원하면 React. 한국 실무 다수는 React 이며 Svelte 는 사이드/경량 프로젝트에서 선택되는 경향.

## 5. 관련

- [[../react/react|↗ React hub]]
- [[../solidjs/solidjs|↗ SolidJS (compile/runtime 대비)]]
- [[../vue/vue|↗ Vue]]
- [[../nuxt/nuxt|↗ Nuxt (메타프레임워크 대비)]]
- [[../_common/_common|↗ 렌더링 전략·반응성 공통]]
- [[../frontend|↑ frontend]]
- 공식 문서 — https://svelte.dev / SvelteKit https://kit.svelte.dev
