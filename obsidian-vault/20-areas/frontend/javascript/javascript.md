---
title: "JavaScript — Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, javascript, hub, beginner]
---

# JavaScript — Hub

**[[../frontend|↑ frontend]]**

> 웹 페이지의 동작 (클릭 / 입력 / 통신) 을 만드는 언어. HTML/CSS 다음 단계. React/Next 의 모든 코드가 이 위에서 동작.

---

## 0. 사용법

1. **HTML/CSS 기본** ([[../html-css/html-css]]) 먼저 읽고 오세요. JS 는 그 위에 동작.
2. 각 문서의 코드는 모두 브라우저 콘솔 (F12 → Console 탭) 에서 바로 실행 가능.
3. "함정" / "왜" 섹션 필독.

## 1. 학습 로드맵 (2~3 주)

### Week 1 — 기초 문법
- [[syntax/variables-types]] — 변수 / 타입 / 연산
- [[syntax/control-flow]] — if / for / while
- [[functions/functions-basics]] — 함수

### Week 2 — 자료구조
- [[objects-arrays/objects]] — 객체 (key-value)
- [[objects-arrays/arrays]] — 배열 + 자주 쓰는 method

### Week 3 — 비동기 / 모듈
- [[async/promises-async-await]] — Promise / async/await / fetch
- [[modules/modules-and-imports]] — import / export

### Week 4 — Modern JS (현업 필수)
- [[modern/es-modern]] — ES2020+ (`?.` / `??` / structuredClone / Array.at / Object.groupBy 등)

---

## 2. 자주 묻는 질문

**Q. JavaScript vs TypeScript?**
→ JS = 기본. TS = JS + 타입 검사. 처음엔 JS 익히고 → 큰 프로젝트는 TS.

**Q. 어디서 JS 실행?**
→ (1) 브라우저 (F12 Console / `<script>` 안). (2) Node.js (터미널). 같은 문법.

**Q. ES5 / ES6 / ES2024 차이?**
→ ECMAScript 표준 버전. ES6 (2015) 가 큰 변화 (`let/const`, arrow, class, module). 요즘은 다 ES6+ 가정.

**Q. jQuery 안 배워도 돼요?**
→ 안 배워도 됨. 옛날 도구. 요즘은 React/Vue/Svelte 같은 프레임워크 + 순수 JS.

---

## 3. 추천 도구

- **연습**: 브라우저 콘솔 (F12)
- **에디터**: VSCode + Live Server
- **온라인 실험**: https://stackblitz.com (가입 없이 npm 패키지까지)
- **레퍼런스**: https://developer.mozilla.org/ko/docs/Web/JavaScript (MDN)
- **튜토리얼**: https://ko.javascript.info (한국어 무료 책)

---

## 4. 관련

- [[../html-css/html-css|↑ HTML/CSS]] — 앞 단계
- [[../typescript/|↑ TypeScript]] — 다음 단계
- [[../react/react|↑ React]] — JS + 컴포넌트
- [[../nextjs/nextjs|↑ Next.js]] — React + 라우팅 + SSR
