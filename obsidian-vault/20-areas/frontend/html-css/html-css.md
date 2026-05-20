---
title: "HTML/CSS — Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, html, css, hub, beginner]
---

# HTML/CSS — Hub

**[[../frontend|↑ frontend]]**

> 웹 페이지의 뼈대 (HTML) 와 옷 (CSS). 모든 프론트엔드의 기초. **HTML 도 처음, CSS 도 처음** 인 사람이 직접 페이지를 만들 수 있을 때까지.

---

## 0. 사용법

1. 위에서부터 순서대로 — 각 문서는 앞 문서를 안다고 가정.
2. 모든 코드 예시는 그대로 복사해서 `.html` 파일에 붙여넣고 브라우저로 열어 보세요. 별다른 도구 필요 X.
3. "함정" / "왜" 섹션은 반드시 — 알면 디버깅 시간 1/10.

## 1. 학습 로드맵 (1~2 주)

### Day 1~2 — HTML 기초
- [[getting-started/what-is-html-css]] — HTML / CSS 가 뭐예요?
- [[getting-started/first-page]] — 첫 HTML 페이지 만들기
- [[html-tags-essentials]] — 자주 쓰는 HTML 태그 20 개

### Day 3~4 — CSS 기초
- [[css-basics]] — selector / 속성 / 박스 모델
- [[colors-and-typography]] — 색 / 폰트 / 크기

### Day 5~7 — 레이아웃
- [[layout/flexbox]] — Flexbox (1차원)
- [[layout/grid]] — Grid (2차원)
- [[layout/positioning]] — position / z-index

### Day 8~10 — 실전
- [[responsive/responsive-design]] — 반응형 / 미디어쿼리
- [[forms/forms-basics]] — input / form / button
- [[semantic-and-accessibility]] — header / nav / main / footer

### Day 11~12 — 모션
- [[animations/css-animations]] — transition / @keyframes / transform / View Transitions

---

## 2. 자주 묻는 질문

**Q. HTML / CSS / JavaScript 차이가 뭐예요?**
→ HTML = 화면 구조 (뼈대), CSS = 스타일 (색·여백·폰트), JavaScript = 동작 (클릭하면 뭐가 일어나는지). 이 셋이 웹 페이지의 3 요소.

**Q. CSS 만 알면 React 같은 거 안 배워도 돼요?**
→ 단순 사이트면 OK. 데이터가 자주 바뀌거나 페이지 많으면 React/Next 가 편함. 하지만 HTML/CSS 는 React 안에서도 그대로 쓰임 — **무조건 먼저 배우는 게 좋음**.

**Q. 디자인 감각이 없어요.**
→ 처음엔 따라하기로 충분. 본인 hompage 의 카드 / pill / 다크 톤 보고 비슷한 패턴 흉내 → 점차 자기 스타일.

**Q. 책 vs 실습?**
→ **무조건 실습**. 코드를 보고 흉내내고 → 손가락이 익으면 됨. 책은 막힐 때 참고.

---

## 3. 추천 도구

- **브라우저**: Chrome 또는 Safari (검사도구 좋음)
- **에디터**: VSCode (무료, 가장 인기) → 확장 "Live Server" 설치하면 저장 시 자동 새로고침
- **연습**: https://codepen.io (가입 없이 바로 코드 실험 가능)
- **레퍼런스**: https://developer.mozilla.org/ko/docs/Web/HTML (MDN)
- **CSS 게임**:
  - https://flexboxfroggy.com — Flexbox 학습 게임
  - https://cssgridgarden.com — Grid 학습 게임

---

## 4. 관련

- [[../javascript/javascript|↑ JavaScript]] — HTML/CSS 다음 단계
- [[../react/react|↑ React]] — 더 큰 사이트 만들기
- [[../_common/_common|↑ frontend 공통]]
