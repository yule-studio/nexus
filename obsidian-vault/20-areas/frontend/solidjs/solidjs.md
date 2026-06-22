---
title: "SolidJS"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - frontend
  - solidjs
---

# SolidJS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-22 | engineering-agent/tech-lead | 개념 개요 hub 로 재작성 |

**[[../frontend|↑ frontend]]**

> 가상 DOM 없이 fine-grained signal 로 DOM 의 정확히 바뀐 부분만 갱신하는 고성능 반응형 라이브러리.

## 1. 무엇인가

SolidJS 는 JSX 문법(React 와 비슷한 모습)을 쓰지만 내부 모델은 React 와 정반대다.

- **No Virtual DOM** — React 가 상태 변경 시 컴포넌트 함수를 다시 실행하고 VDOM 을 diff 하는 반면, Solid 는 컴포넌트 함수를 **단 한 번만** 실행한다. 함수는 "어떤 signal 이 어떤 DOM 노드를 갱신하는가"의 배선을 만들 뿐이다.
- **fine-grained reactivity (signals)** — `createSignal` 로 만든 값이 바뀌면, 그 값을 읽은 정확한 effect/DOM 표현식만 다시 돈다. 컴포넌트 전체 재실행이 없다.
- **컴파일러 + 런타임** — JSX 를 효율적인 DOM 생성/갱신 코드로 컴파일하고, 작은 반응성 런타임이 의존성 추적을 맡는다.

결과적으로 React 의 멘탈 모델("뷰는 상태의 함수")과 표면 문법은 닮았지만, 재렌더 개념 자체가 없다.

## 2. 왜 / 언제 선택하나

(1) **강점** — 갱신이 외과수술처럼 정밀해 벤치마크 상위권 성능과 작은 번들을 낸다. React 의 `useMemo`/`useCallback`/의존성 배열/stale closure 같은 재렌더 관리 부담이 거의 없다 — 의존성은 자동 추적된다.

(2) **안 맞는데 쓰면** — 생태계가 작아, 검증된 컴포넌트 라이브러리·고용 풀·레퍼런스가 React 대비 빈약하다. 큰 팀/장기 제품에서 "필요한 라이브러리가 Solid 용으로는 없다"는 벽에 자주 부딪힌다. 또 JSX 가 React 와 닮아 보여 React 습관(props 구조분해, 조건부에서 변수 분해)을 그대로 쓰면 반응성이 끊겨 버그가 난다.

(3) **대안 비교**
- **vs React** — 문법은 유사, 모델은 정반대(재렌더 없음 vs VDOM diff). Solid 가 성능·번들·반응성 자동화에서 앞서고, React 가 생태계·채용·자료에서 압도.
- **vs Svelte** — 둘 다 VDOM 을 버린 고성능 진영. Svelte 는 컴파일 타임에 반응성 코드를 생성(런타임 최소), Solid 는 런타임 signal 로 추적. Solid 가 JSX/조합성을 선호하는 쪽, Svelte 가 전용 문법으로 더 간결한 쪽.
- **vs Vue** — Vue 도 반응형이지만 VDOM 을 유지한다. Solid 는 VDOM 자체가 없어 갱신이 더 정밀.

(4) **트레이드오프** — 성능/번들은 최상위, 러닝커브는 JSX 라 진입은 쉽지만 "재렌더가 없다"는 모델 전환이 필요, 생태계는 작은 편. 성능이 제품의 핵심 가치이고 의존 라이브러리가 적은 경우에 가장 빛난다.

## 3. 핵심 개념 / 어떻게

```jsx
import { createSignal, createMemo } from "solid-js";

function Counter() {
  const [count, setCount] = createSignal(0);   // [getter, setter]
  const double = createMemo(() => count() * 2); // 파생 값
  // 이 함수 본문은 단 한 번만 실행됨
  return <button onClick={() => setCount(c => c + 1)}>
    {count()} / {double()}
  </button>;
}
```

- signal 은 **함수 호출**로 읽는다(`count()`). 값을 변수로 빼서 쓰면 추적이 끊긴다.
- `createMemo` 는 파생 캐시, `createEffect` 는 부수효과(의존성 자동 추적).
- 리스트는 `<For>`, 조건은 `<Show>` 컴포넌트로 — 재실행이 없으므로 `.map`/삼항을 그냥 쓰지 않는다.
- 메타프레임워크는 **SolidStart**(Nuxt/Next 대응)가 SSR/라우팅을 제공한다.

## 4. 적용 맥락

- 인터랙션이 많고 갱신 빈도가 높은 대시보드·에디터·실시간 UI, 또는 번들/성능이 KPI 인 경량 위젯에 적합.
- **React 와 갈리는 지점** — 성능과 반응성 자동화를 최우선하고 생태계 규모를 양보할 수 있으면 Solid, 검증된 라이브러리·채용·자료가 우선이면 React. React 경험자라면 진입 문턱은 낮지만 "재렌더 없음" 모델을 반드시 다시 학습해야 한다.

## 5. 관련

- [[../react/react|↗ React hub (JSX 유사·모델 대비)]]
- [[../svelte-sveltekit/svelte-sveltekit|↗ Svelte (compile-time 진영)]]
- [[../vue/vue|↗ Vue]]
- [[../_common/_common|↗ 반응성/상태 공통 개념]]
- [[../frontend|↑ frontend]]
- 공식 문서 — https://www.solidjs.com / SolidStart https://start.solidjs.com
