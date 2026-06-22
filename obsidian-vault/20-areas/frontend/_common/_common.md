---
title: "Frontend 공통"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - frontend
  - _common
---

# Frontend 공통

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-22 | engineering-agent/tech-lead | framework-agnostic 공통 개념 인덱스로 재작성 |

**[[../frontend|↑ frontend]]**

> 특정 프레임워크와 무관한 프론트엔드 공통 개념 인덱스 — 컴포넌트/상태/라우팅/서버데이터/빌드/렌더링 전략.

## 1. 무엇인가

React·Vue·Svelte·Solid·Angular 는 문법이 달라도 같은 문제를 푼다. 프레임워크를 갈아타도 변하지 않는 공통 개념은 다음과 같다.

- **컴포넌트 모델** — UI 를 재사용 가능한 단위로 쪼개고, 데이터를 위→아래(props/입력)로 흘리고, 이벤트를 아래→위(emit/콜백)로 올린다. "단방향 데이터 흐름"이 공통 원칙.
- **반응성** — 상태가 바뀌면 화면이 따라 바뀐다. 차이는 그 갱신을 누가/언제 하느냐(런타임 VDOM diff vs fine-grained signal vs 컴파일 타임).
- **선언적 렌더링** — "어떻게 DOM 을 조작할지"가 아니라 "상태가 이러면 화면은 이렇다"를 선언한다.

## 2. 왜 / 언제 이 추상이 중요한가

(1) **강점** — 공통 모델을 알면 새 프레임워크 학습이 "문법 매핑"으로 줄어든다. `useState` ↔ `ref` ↔ `$state` ↔ `createSignal` 은 모두 "반응형 상태"라는 같은 개념의 표현일 뿐이다.

(2) **안 보면 생기는 문제** — 프레임워크 문법만 외우면, 도구가 바뀔 때마다 처음부터 배우게 되고 "이 상태를 어디에 둘지(로컬/전역/서버)" 같은 설계 판단을 못 한다. 상태 범주를 모르면 모든 걸 전역 스토어에 넣는 흔한 안티패턴에 빠진다.

(3) **공통 vs 프레임워크 고유** — 컴포넌트 분해·상태 범주·라우팅·데이터 페칭·렌더링 전략은 공통이고, 반응성을 처리하는 메커니즘과 문법은 프레임워크 고유다.

(4) **트레이드오프** — 공통 개념으로 추상화해 이해하면 이식성이 좋지만, 각 프레임워크의 고유 최적화(예: Solid 의 "재렌더 없음")를 무시하면 그 도구를 잘못 쓰게 된다. 공통 개념 위에 프레임워크별 특성을 얹어야 한다.

## 3. 핵심 개념 / 범주

**컴포넌트 모델**
- props/입력으로 데이터 하향, 이벤트/콜백으로 상향. children/slot 으로 합성(composition).

**상태 관리 범주** — "이 상태를 어디 둘지"가 핵심 판단.
- 로컬 상태 — 한 컴포넌트 안. (`useState`, `ref`, `$state`)
- 공유/전역 상태 — 여러 컴포넌트가 쓰는 클라이언트 상태. (Zustand/Jotai/Redux, Pinia, Svelte store)
- 서버 상태 — 백엔드가 진실인 데이터. 캐시/무효화/재요청이 별도 문제. (TanStack Query, RTK Query, useFetch)
- URL 상태 — 라우터의 path/query 가 보유.

**라우팅** — URL ↔ 화면 매핑. 코드 기반(react-router) vs 파일 기반(Next/Nuxt/SvelteKit `pages`/`routes`). 동적 파라미터, 중첩 라우트, lazy 로딩.

**서버 데이터 페칭** — 단순 `fetch`/`axios` 호출과, 캐시·로딩/에러 상태·중복요청 제거·재검증을 다루는 데이터 레이어(TanStack Query 등)는 다른 층이다.

**빌드 도구** — 번들러/개발 서버. **Vite**(esbuild+Rollup, 현대 표준), 구형 webpack, 그리고 esbuild/SWC/Rollup/Turbopack. dev 서버·HMR·트리셰이킹·코드 스플리팅을 담당.

```js
// 어느 프레임워크든 같은 개념: 상태 → 파생 → 화면
// React:  const [n,setN]=useState(0); const d=useMemo(()=>n*2,[n]);
// Vue:    const n=ref(0);            const d=computed(()=>n.value*2);
// Svelte: let n=$state(0);           let d=$derived(n*2);
// Solid:  const [n,setN]=createSignal(0); const d=createMemo(()=>n()*2);
```

## 4. 렌더링 전략 (CSR/SSR/SSG/ISR)

같은 앱도 "HTML 을 언제/어디서 만드느냐"가 다르다 — SEO·초기 로딩·서버 비용의 맞교환.

- **CSR(Client-Side Rendering)** — 빈 HTML + JS 번들을 받아 브라우저가 화면을 그림. 인증 뒤 대시보드/사내 도구에 적합. SEO·초기 페인트는 불리.
- **SSR(Server-Side Rendering)** — 요청 시 서버가 HTML 을 만들어 보냄(이후 hydration). SEO·초기 페인트 유리, 서버 부하·hydration 복잡도 증가. (Next/Nuxt/SvelteKit)
- **SSG(Static Site Generation)** — 빌드 시 HTML 을 미리 생성. 가장 빠르고 싸지만 콘텐츠가 자주 안 바뀌어야 함. 블로그/문서/마케팅.
- **ISR(Incremental Static Regeneration)** — SSG + 일정 주기/요청 시 백그라운드 재생성. 정적 속도와 신선도의 절충.

선택 기준: SEO/초기 로딩이 매출과 직결 → SSR/SSG, 항상 동적이고 인증 뒤 → CSR, 콘텐츠 거의 고정 → SSG, 정적이되 가끔 갱신 → ISR.

## 5. 관련

- [[../react/react|↗ React]]
- [[../vue/vue|↗ Vue]]
- [[../angular/angular|↗ Angular]]
- [[../svelte-sveltekit/svelte-sveltekit|↗ Svelte/SvelteKit]]
- [[../solidjs/solidjs|↗ SolidJS]]
- [[../nuxt/nuxt|↗ Nuxt]]
- [[../ui-libraries/ui-libraries|↗ UI 라이브러리]]
- [[../frontend|↑ frontend]]
- 공식 — Vite https://vitejs.dev / MDN https://developer.mozilla.org / web.dev (Core Web Vitals) https://web.dev/vitals
