---
title: "Testing — Hub (Vitest / RTL / Playwright)"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, testing, hub]
---

# Testing — Hub

**[[../frontend|↑ frontend]]**

> 코드를 자동으로 검증. 현업 PR 머지 조건.

---

## 1. 테스트 피라미드

```
       /\         E2E (느림, 적음) — Playwright / Cypress
      /  \
     /────\
    /      \      Integration — 컴포넌트 + API
   /        \
  /──────────\
 /            \   Unit — 함수 / 컴포넌트
/______________\
```

비율 권장: **Unit 70% / Integration 20% / E2E 10%**.

---

## 2. 학습 로드맵

### Tier 1 — 필수
- [[unit/vitest]] — Vitest + RTL 셋업 + 첫 테스트

### Tier 2 — 추가
- [[e2e/playwright]] — Playwright E2E

---

## 3. 도구 비교

| 도구 | 종류 | 추천 |
| --- | --- | --- |
| **Vitest** | Unit / Integration | 새 프로젝트 (Vite) |
| **Jest** | Unit / Integration | 기존 / Next |
| **React Testing Library** | Component | 위 둘과 함께 |
| **Playwright** | E2E | 새 프로젝트 |
| **Cypress** | E2E | 기존 / 학습 친화 |
| **Storybook** | Visual / Interaction | 디자인 시스템 |

---

## 4. 핵심 원칙 (RTL 모토)

> "사용자가 보는 것 테스트, 구현 상세 X"

### ❌
```tsx
expect(component.state.count).toBe(5);
```

### ✅
```tsx
expect(screen.getByText("Count: 5")).toBeInTheDocument();
expect(screen.getByRole("button", { name: "증가" })).toBeInTheDocument();
```

---

## 5. 무엇을 테스트할까

**Test**: 비즈니스 로직 / 사용자 인터랙션 / 에지 케이스 / 회귀.
**Skip**: React 자체 / 단순 표시 / 사소한 helper / 구현 디테일.

---

## 6. CI/CD

`.github/workflows/test.yml`:
```yaml
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint
      - run: npm test
      - run: npm run test:e2e   # 선택
```

테스트 실패 → PR 머지 X.

---

## 7. 관련

- [[../react/react|↑ React]]
- [[../nextjs/nextjs|↑ Next.js]]
