---
title: "TypeScript — Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, typescript, hub]
---

# TypeScript — Hub

**[[../frontend|↑ frontend]]**

> JavaScript + 타입 검사. 현업 거의 모든 React/Next 프로젝트의 표준. 처음엔 부담이지만 익숙해지면 안 쓰면 답답함.

---

## 0. 왜 TypeScript?

| | JS | TS |
| --- | --- | --- |
| 타입 검사 | 런타임 (실행해야 알음) | 컴파일 (작성 중) |
| 자동 완성 | 추측 | 정확 |
| 리팩토링 안전성 | 검색-치환 | 컴파일러 보장 |
| 협업 / 큰 프로젝트 | 자유롭지만 위험 | 강제 / 안전 |
| 학습 곡선 | 낮음 | 중간 |

**현업 권장**: 새 프로젝트는 TS. 학습은 JS → TS 순.

---

## 1. 학습 로드맵 (3~4 주)

### Week 1 — 기초 타입
- [[getting-started/setup]] — tsc / Vite / Next 셋업
- [[types/primitives]] — string / number / boolean / array / tuple
- [[types/objects-interfaces]] — type vs interface / object 타입

### Week 2 — 활용
- [[types/unions-intersections]] — union (`|`) / intersection (`&`) / literal
- [[types/generics]] — `<T>` 제네릭 / 제약 (`extends`)
- [[advanced/utility-types]] — Partial / Required / Pick / Omit / Record

### Week 3 — 고급
- [[advanced/type-guards-narrowing]] — typeof / instanceof / in / discriminated union
- [[advanced/mapped-conditional]] — mapped / conditional / template literal

### Week 4 — React + TS
- [[react/component-typing]] — Props / Event / Ref / children
- [[react/hooks-typing]] — useState / useReducer / useRef / Custom hook

---

## 2. 현업 핵심 (꼭 알아야)

- **strict mode** — `tsconfig.json` 에 `"strict": true`. 안 하면 TS 쓰는 의미 절반 X
- **no any** — `any` 는 패배. `unknown` 으로 시작 + narrowing
- **type vs interface** — 둘 다 OK. 객체는 interface, union 은 type
- **Discriminated Union** — switch-case + 타입 안전
- **`as const`** — 리터럴 타입 유지
- **satisfies** — 타입 강제 검사하면서 추론 유지
- **zod / valibot** — 런타임 schema 검증 + 타입 자동 생성

---

## 3. 자주 묻는 질문

**Q. TS 만 배워도 돼? JS 모르면 안 됨?**
→ 모르면 안 됨. TS 는 JS 위에 있음. JS 의 closure / this / Promise 다 동일.

**Q. type 과 interface 차이?**
→ 거의 같음.
- interface — 확장 (`extends`) / declaration merging. 객체 위주.
- type — 모든 표현 가능 (union / intersection / mapped / conditional).
- 팀 컨벤션 따라 일관성. 본인 hompage 는 type 사용.

**Q. `any` 써도 돼?**
→ 정말 모르겠을 때만 임시. `unknown` 부터.

**Q. zod 가 뭐예요?**
→ 런타임 + 컴파일 타임 검증. API 응답 / form data 처럼 외부에서 오는 데이터에 필수.

---

## 4. 도구

- **공식 문서**: https://www.typescriptlang.org/docs/ (영어, 깊음)
- **handbook (요약)**: https://www.typescriptlang.org/docs/handbook/intro.html
- **playground**: https://www.typescriptlang.org/play
- **type-challenges** (퀴즈): https://github.com/type-challenges/type-challenges
- **에디터**: VSCode (TS 가장 잘 지원)

---

## 5. 관련

- [[../javascript/javascript|↑ JavaScript]] — 전제
- [[../react/typescript/typescript|↑ React + TS]] — 컴포넌트 패턴
- [[../nextjs/nextjs|↑ Next.js]] — TS 기본 권장
