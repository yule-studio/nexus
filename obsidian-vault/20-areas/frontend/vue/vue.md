---
title: "Vue"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - frontend
  - vue
---

# Vue

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-22 | engineering-agent/tech-lead | 개념 개요 hub 로 재작성 |

**[[../frontend|↑ frontend]]**

> 반응형(reactive) 데이터 모델과 단일 파일 컴포넌트(SFC)를 결합한 점진적 도입형 프론트엔드 프레임워크.

## 1. 무엇인가

Vue 는 선언적 렌더링과 반응성(reactivity)을 핵심으로 하는 UI 프레임워크다. 핵심 모델은 두 가지다.

- **반응성 시스템** — `ref` / `reactive` 로 만든 상태가 변하면, 그 상태를 읽은 렌더 함수가 자동으로 다시 실행된다. Vue 3 는 ES Proxy 기반으로 의존성을 추적해, 바뀐 값에 닿는 부분만 갱신한다.
- **단일 파일 컴포넌트(SFC, `.vue`)** — `<template>` / `<script setup>` / `<style scoped>` 를 한 파일에 모은다. 템플릿은 HTML 에 가깝고, 빌드 시 렌더 함수로 컴파일된다.

내부적으로는 가상 DOM(Virtual DOM)을 쓰되, 컴파일러가 템플릿의 정적 부분을 hoist 하고 동적 부분만 추적해 diff 비용을 줄이는 "compiler-informed VDOM" 방식이다. 상태 관리는 공식 라이브러리 **Pinia**(Vuex 후속)를 표준으로 쓴다.

## 2. 왜 / 언제 선택하나

(1) **강점** — 학습 곡선이 완만하다. HTML/CSS/JS 를 알면 SFC 의 `<template>` 을 거의 그대로 읽을 수 있다. 반응성이 "자동"이라 React 처럼 `useMemo`/`useCallback` 로 재렌더를 수동 제어할 일이 적다. 공식 라우터(vue-router)·상태(Pinia)·메타프레임워크(Nuxt)가 한 팀에서 관리돼 생태계 일관성이 높다.

(2) **안 맞는데 쓰면** — 초대형 조직에서 강한 구조 강제와 의존성 주입(DI)이 필요하면 Vue 의 유연함이 오히려 규약 분산을 낳는다. 또 React 중심 채용 시장(특히 한국 스타트업)에서는 Vue 경험자 풀이 더 좁아 채용/유지보수 리스크가 생긴다.

(3) **대안 비교**
- **vs React** — Vue 는 템플릿+자동 반응성, React 는 JSX+명시적 재렌더. Vue 가 보일러플레이트가 적고 입문이 쉽지만, React 는 생태계/채용/타입 추론에서 우위.
- **vs Svelte** — Svelte 는 컴파일 타임에 반응성을 제거(런타임 거의 없음)해 더 가볍지만, Vue 가 생태계와 SSR(Nuxt) 성숙도에서 앞선다.
- **vs Solid** — Solid 는 VDOM 없는 fine-grained signals 로 Vue 보다 갱신이 정밀하고 빠르지만, 생태계가 훨씬 작다.

(4) **트레이드오프** — 생태계는 React 다음으로 크고, 러닝커브는 가장 낮은 축, 번들/성능은 React 보다 가볍고 Svelte/Solid 보다는 무겁다. Options API ↔ Composition API 두 스타일이 공존해 오래된 코드베이스에서 혼란이 생길 수 있다.

## 3. 핵심 개념 / 어떻게

```vue
<script setup>
import { ref, computed } from 'vue'

const count = ref(0)                       // 반응형 상태
const double = computed(() => count.value * 2)  // 파생 값
function inc() { count.value++ }           // .value 로 접근
</script>

<template>
  <button @click="inc">{{ count }} / {{ double }}</button>
</template>
```

- `ref` 는 원시값, `reactive` 는 객체를 반응형으로 만든다. 템플릿에서는 `.value` 없이 자동 언래핑된다.
- `computed` 는 의존 값이 바뀔 때만 재계산되는 캐시된 파생 상태.
- `watch` / `watchEffect` 로 부수효과를 선언한다.
- 부모→자식은 `defineProps`, 자식→부모는 `defineEmits`.

```js
// Pinia store
import { defineStore } from 'pinia'
export const useCounter = defineStore('counter', () => {
  const n = ref(0)
  const inc = () => n.value++
  return { n, inc }
})
```

## 4. 적용 맥락

- SSR/SEO 가 필요하면 Vue 단독이 아니라 **Nuxt** 를 쓴다([[../nuxt/nuxt]]).
- 점진적 마이그레이션(기존 jQuery/멀티페이지 앱에 일부만 SPA 화)에 강하다 — `<script>` 태그 한 줄로도 붙일 수 있다.
- **React 와 갈리는 지점** — 팀이 템플릿 친화적이고 빠른 입문/적은 보일러플레이트를 원하면 Vue, 큰 생태계·채용 풀·풍부한 타입 도구를 원하면 React. 한국 실무에서는 React 가 다수지만, 어드민/사내 도구·중소 SPA 에서 Vue 채택이 꾸준하다.

## 5. 관련

- [[../react/react|↗ React hub]]
- [[../nuxt/nuxt|↗ Nuxt (Vue 메타프레임워크)]]
- [[../svelte-sveltekit/svelte-sveltekit|↗ Svelte]]
- [[../solidjs/solidjs|↗ SolidJS]]
- [[../_common/_common|↗ 프레임워크 공통 패턴]]
- [[../frontend|↑ frontend]]
- 공식 문서 — https://vuejs.org / Pinia https://pinia.vuejs.org
