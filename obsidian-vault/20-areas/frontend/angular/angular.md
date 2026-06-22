---
title: "Angular"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - frontend
  - angular
---

# Angular

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-22 | engineering-agent/tech-lead | 개념 개요 hub 로 재작성 |

**[[../frontend|↑ frontend]]**

> 라우팅·폼·HTTP·DI·테스트를 한 패키지로 제공하는, TypeScript 우선의 풀 프레임워크(full-fledged framework).

## 1. 무엇인가

Angular(2+ , 흔히 "Angular")는 Google 이 관리하는 종합 프레임워크다. React/Vue 가 "라이브러리에 생태계를 조합"하는 방식이라면, Angular 는 라우터·폼·HTTP 클라이언트·테스트·CLI 를 공식으로 묶어 제공한다. 핵심 모델은 세 가지다.

- **컴포넌트 + 의존성 주입(DI)** — 컴포넌트/서비스를 클래스로 정의하고, 서비스는 DI 컨테이너가 주입한다. 비즈니스 로직을 서비스로 분리하는 구조가 강제에 가깝다.
- **변화 감지(Change Detection)** — 전통적으로 Zone.js 기반으로 비동기 이벤트 후 트리를 검사했고, 최근 버전은 **Signals**(fine-grained 반응성) 와 `OnPush` 로 갱신 범위를 좁힌다.
- **RxJS 기반 비동기** — HTTP·이벤트·상태 스트림을 Observable 로 다룬다.

TypeScript 가 선택이 아니라 기본이며, 데코레이터(`@Component`, `@Injectable`)와 강타입이 전제다.

## 2. 왜 / 언제 선택하나

(1) **강점** — "정답이 정해진" 프레임워크다. 폴더 구조·DI·라우팅·폼 검증·HTTP 인터셉터가 공식 규약으로 일관돼, 대규모 팀이 들어와도 코드가 비슷한 모양을 유지한다. 장기 유지보수·엔터프라이즈 LOB(사내 업무) 앱에 강하다.

(2) **안 맞는데 쓰면** — 작은 SPA·랜딩·프로토타입에는 무겁다. DI/RxJS/모듈 개념을 먼저 넘어야 첫 화면이 나오므로 초기 속도가 느리다. 반대로 Angular 가 강제하는 구조를 무시하고 쓰면 프레임워크의 이점이 사라지고 그냥 무거운 React 가 된다.

(3) **대안 비교**
- **vs React** — React 는 자유도가 높아 팀마다 구조가 갈리지만, Angular 는 규약으로 통일한다. React 가 생태계/채용에서 앞서고, Angular 가 일관성/내장 기능에서 앞선다.
- **vs Vue** — Vue 도 공식 라우터/상태가 있지만 "점진적 도입"이 핵심 철학이라 Angular 보다 가볍고 느슨하다.
- **vs Svelte/Solid** — 둘은 경량·고성능 지향으로 Angular 와 정반대 스펙트럼(미니멀 vs 종합).

(4) **트레이드오프** — 생태계는 안정적이지만 React 보다 작고, 러닝커브는 가장 가파른 축(DI+RxJS+모듈), 번들은 큰 편이나 최신 버전(standalone components, signals, esbuild)이 많이 개선했다. 큰 강제 구조 = 장기 유지보수 안정성과 초기 생산성의 맞교환.

## 3. 핵심 개념 / 어떻게

```ts
@Component({
  selector: 'app-counter',
  standalone: true,
  template: `<button (click)="inc()">{{ count() }} / {{ double() }}</button>`,
})
export class CounterComponent {
  count = signal(0);                          // 반응형 signal
  double = computed(() => this.count() * 2);  // 파생 값
  inc() { this.count.update(n => n + 1); }
}
```

- `signal()` / `computed()` 로 fine-grained 반응성을 선언한다(최신 표준).
- 서비스는 `@Injectable` 로 정의하고 생성자에서 주입받는다.

```ts
@Injectable({ providedIn: 'root' })
export class Api {
  constructor(private http: HttpClient) {}
  users() { return this.http.get<User[]>('/api/users'); } // Observable 반환
}
```

- 템플릿에서 `(event)` 는 출력 바인딩, `[prop]` 는 입력 바인딩, `{{ }}` 는 보간.

## 4. 적용 맥락

- 엔터프라이즈 어드민·금융/공공 SI·장수명 LOB 앱처럼 **팀 회전이 잦고 구조 일관성이 핵심**인 곳에 적합.
- 표준이 정해져 신규 인원 온보딩 시 "어디에 무엇을 둘지" 논쟁이 적다.
- **React 와 갈리는 지점** — 자유도·빠른 입문·큰 채용 풀을 원하면 React, 규약·DI·내장 일괄 제공·장기 일관성을 원하면 Angular. 한국 실무에서는 React 가 다수이고 Angular 는 대형 SI/사내 시스템에서 주로 보인다.

## 5. 관련

- [[../react/react|↗ React hub]]
- [[../vue/vue|↗ Vue]]
- [[../svelte-sveltekit/svelte-sveltekit|↗ Svelte]]
- [[../solidjs/solidjs|↗ SolidJS]]
- [[../_common/_common|↗ 프레임워크 공통 패턴]]
- [[../frontend|↑ frontend]]
- 공식 문서 — https://angular.dev / RxJS https://rxjs.dev
