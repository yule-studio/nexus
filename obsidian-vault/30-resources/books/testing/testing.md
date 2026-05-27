---
title: "Testing / Quality — 책 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-26T19:45:00+09:00
updated_at: 2026-05-26T20:45:00+09:00
tags: [resources, books, testing, quality]
home_hub: resources
---

# Testing / Quality — 책 카탈로그

**[[../books|↑ Books hub]]**

> 외부 요약 X. 책 + 구매 링크.
> 🇰🇷 = 교보문고 검색, 원서 = 출판사 공식 또는 Amazon 검색.
> ⭐ essential, 🆕 최근 5년, 🆓 무료, 📖 두꺼움.

## 1. Testing 일반

### 입문
- ⭐ **Test-Driven Development: By Example** (Kent Beck, Addison-Wesley, 2002)
  - 주제: TDD 의 정통 (古典)
  - 🇰🇷 [테스트 주도 개발 Kent Beck](https://search.kyobobook.co.kr/search?keyword=테스트+주도+개발+Kent+Beck)
  - 원서: [Amazon](https://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530)

- **Growing Object-Oriented Software, Guided by Tests** (Steve Freeman, Nat Pryce, Addison-Wesley, 2009)
  - 주제: GOOS — TDD + mock 의 정통
  - 원서: [Amazon](https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627)
  - 🇰🇷 [GOOS 테스트 주도 개발로 배우는 객체 지향 설계와 실천](https://search.kyobobook.co.kr/search?keyword=테스트+주도+개발로+배우는+객체+지향+설계와+실천)

### 중급
- ⭐ **Unit Testing Principles, Practices, and Patterns** (Vladimir Khorikov, Manning, 2020) 🆕
  - 주제: modern unit testing (mock 의 함정, classical school)
  - 🇰🇷 [단위 테스트](https://search.kyobobook.co.kr/search?keyword=단위+테스트+Khorikov)
  - 원서: [Manning](https://www.manning.com/books/unit-testing)

- **xUnit Test Patterns** (Gerard Meszaros, Addison-Wesley, 2007) 📖
  - 주제: xUnit pattern 의 정통 (Test Double 의 origin)
  - 공식: [xunitpatterns.com](http://xunitpatterns.com/)
  - 원서: [Amazon](https://www.amazon.com/xUnit-Test-Patterns-Refactoring-Code/dp/0131495054)

### 고급
- **Working Effectively with Unit Tests** (Jay Fields, Leanpub, 2014)
  - 공식: [Leanpub](https://leanpub.com/wewut)

- **Working Effectively with Legacy Code** → [[../architecture/architecture]] §9

## 2. Java Testing

- ⭐ **Effective Software Testing** (Maurício Aniche, Manning, 2022) 🆕
  - 주제: Java 기반 testing (TU Delft 교수)
  - 원서: [Manning](https://www.manning.com/books/effective-software-testing)

- **JUnit in Action (3rd)** (Catalin Tudose, Manning, 2020) 🆕
  - 원서: [Manning](https://www.manning.com/books/junit-in-action-third-edition)

- 🇰🇷 [JUnit 5 활용](https://search.kyobobook.co.kr/search?keyword=JUnit+5)
- 🇰🇷 [Mockito](https://search.kyobobook.co.kr/search?keyword=Mockito)

## 3. JavaScript / TypeScript Testing

- ⭐ **Testing JavaScript** (Kent C. Dodds, 2024) 🆕
  - 주제: Vitest / Jest / Testing Library / Playwright / Cypress
  - 공식: [testingjavascript.com](https://testingjavascript.com/)

- **JavaScript Testing Recipes** (James Coglan, 2014)
  - 원서: [Amazon](https://www.amazon.com/JavaScript-Testing-Recipes-James-Coglan/dp/1493593587)

## 4. Python Testing

- **Python Testing with pytest (2nd)** (Brian Okken, Pragmatic, 2022) 🆕
  - 주제: pytest 의 정통
  - 원서: [Pragmatic](https://pragprog.com/titles/bopytest2/python-testing-with-pytest-second-edition/)
  - 🇰🇷 [파이썬 테스팅 pytest](https://search.kyobobook.co.kr/search?keyword=파이썬+테스팅+pytest)

- **Architecture Patterns with Python** → [[../backend/backend]] §4 — TDD + DDD + Python

## 5. End-to-End / Browser — 공식 docs

- ⭐ **Playwright** — [playwright.dev](https://playwright.dev/)
- **Cypress** — [docs.cypress.io](https://docs.cypress.io/)
- **Cypress.io Real World App** — [GitHub](https://github.com/cypress-io/cypress-realworld-app)

## 6. Performance / Load Testing

- **The Art of Application Performance Testing (2nd)** (Ian Molyneaux, O'Reilly, 2014)
  - 원서: [O'Reilly](https://www.oreilly.com/library/view/the-art-of/9781491900536/)

- **Web Performance in Action** → [[../frontend/frontend]] §9

- **k6** — [k6.io/docs](https://k6.io/docs/)
- **Locust** — [docs.locust.io](https://docs.locust.io/)
- **Gatling** — [gatling.io/docs](https://gatling.io/docs/)

## 7. Property-Based Testing

- ⭐ **Property-Based Testing with PropEr, Erlang, and Elixir** (Fred Hebert, Pragmatic, 2019)
  - 주제: PBT 의 정통
  - 원서: [Pragmatic](https://pragprog.com/titles/fhproper/property-based-testing-with-proper-erlang-and-elixir/)

- **Hypothesis (Python)** — [hypothesis.readthedocs.io](https://hypothesis.readthedocs.io/)
- **fast-check (JS)** — [GitHub](https://github.com/dubzzz/fast-check)

## 8. Mutation Testing

- **PITest (Java)** — [pitest.org](https://pitest.org/)
- **Stryker (JS / .NET / Scala)** — [stryker-mutator.io](https://stryker-mutator.io/)

## 9. Contract Testing

- **Pact** — [docs.pact.io](https://docs.pact.io/)
- **Spring Cloud Contract** — [공식 docs](https://spring.io/projects/spring-cloud-contract)

## 10. Quality / Software Engineering

- ⭐ **Accelerate** → [[../devops/devops]] §1 — DORA metric
- **The Phoenix Project** → [[../devops/devops]] §1
- **Clean Code** → [[../backend/backend]] §1
- **Code Complete (2nd)** → [[../backend/backend]] §1

## 11. Chaos Engineering

- ⭐ **Chaos Engineering** (Casey Rosenthal, Nora Jones, O'Reilly, 2020) 🆕 🆓
  - 주제: Netflix 의 chaos engineering 정통
  - 원서: [O'Reilly](https://www.oreilly.com/library/view/chaos-engineering/9781492043861/)

- **Learning Chaos Engineering** (Russ Miles, O'Reilly, 2019)
  - 원서: [O'Reilly](https://www.oreilly.com/library/view/learning-chaos-engineering/9781492051000/)

## 12. 관련

- [[../books]] — Books hub
- [[../backend/backend]] — 언어별 testing
- [[../frontend/frontend]] §11 — frontend testing
- [[../devops/devops]] §1 — DORA metric
- [[../architecture/architecture]] §9 — legacy code
