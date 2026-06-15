---
title: "Foundations / CS 古典 — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, foundations, classics]
home_hub: resources
---

# Foundations / CS 古典 — Papers

**[[papers|↑ Papers hub]]**

> 계산 가능성 · 정보이론 · 언어/검증 · 시스템 설계의 뿌리를 만든 컴퓨터 과학 古典 논문 모음.

## 1. 계산 가능성 / 계산 이론 (Computability)

- **On Computable Numbers, with an Application to the Entscheidungsproblem** (A. M. Turing, Proc. London Math. Society, 1936) ⭐ 🆓
  - 주제: 튜링 머신·계산 가능성·정지 문제 — 계산 이론 자체를 창시.
  - 링크: [cs.virginia.edu (PDF)](https://www.cs.virginia.edu/~robins/Turing_Paper_1936.pdf) 🆓
- **An Unsolvable Problem of Elementary Number Theory** (A. Church, American J. of Mathematics, 1936) ⭐
  - 주제: λ-계산으로 결정 불가능성 증명 — Church–Turing 논제의 한 축.
  - 링크: [JSTOR / DOI](https://doi.org/10.2307/2371045)
- **Computability and λ-Definability** (A. M. Turing, J. Symbolic Logic, 1937) 
  - 주제: 튜링 계산 가능성 = λ-정의 가능성 등가 증명 — Church–Turing 통합.
  - 링크: [DOI 10.2307/2268280](https://doi.org/10.2307/2268280)
- **Gödel's incompleteness theorems** (K. Gödel, 1931, 맥락 참고) ⭐ 📖
  - 주제: 형식 체계의 한계 — 계산 가능성·결정 불가능성 논의의 사상적 배경.
  - 링크: [SEP 개설](https://plato.stanford.edu/entries/goedel-incompleteness/)

## 2. 정보 이론 (Information Theory)

- **A Symbolic Analysis of Relay and Switching Circuits** (C. E. Shannon, Trans. AIEE, 1937/1938) ⭐
  - 주제: 불 대수 = 스위칭 회로 — 디지털 논리 회로 설계의 출발점(석사 논문).
  - 링크: [DOI 10.1109/T-AIEE.1938.5057767](https://doi.org/10.1109/T-AIEE.1938.5057767)
- **A Mathematical Theory of Communication** (C. E. Shannon, Bell System Tech. J., 1948) ⭐ 🆓 📖
  - 주제: 엔트로피·채널 용량·부호화 — 정보 이론 전체를 창시.
  - 링크: [Wiley DOI](https://doi.org/10.1002/j.1538-7305.1948.tb01338.x) · [Harvard 사본 (PDF)](https://people.math.harvard.edu/~ctm/home/text/others/shannon/entropy/entropy.pdf) 🆓

## 3. 컴퓨터 구조 / 시스템 (Architecture & Systems)

- **First Draft of a Report on the EDVAC** (J. von Neumann, 1945) ⭐ 🆓 📖
  - 주제: 저장 프로그램 구조 — 현대 컴퓨터 아키텍처(폰 노이만 구조)의 원형.
  - 링크: [archive.org 사본 (PDF)](https://web.mit.edu/STS.035/www/PDFs/edvac.pdf) 🆓
- **The Structure of the "THE"-Multiprogramming System** (E. W. Dijkstra, CACM, 1968) ⭐
  - 주제: 계층적 OS 설계·세마포어 — 운영체제 추상화 계층의 효시.
  - 링크: [dl.acm.org/doi/10.1145/363095.363143](https://dl.acm.org/doi/10.1145/363095.363143)
- **Self-stabilizing Systems in Spite of Distributed Control** (E. W. Dijkstra, CACM, 1974) ⭐
  - 주제: 자기 안정화 — 결함 허용·분산 제어 이론의 토대.
  - 링크: [dl.acm.org/doi/10.1145/361179.361202](https://dl.acm.org/doi/10.1145/361179.361202)
- **Time, Clocks, and the Ordering of Events in a Distributed System** (L. Lamport, CACM, 1978) ⭐ 🏆
  - 주제: happens-before·논리 시계 — 분산 시스템 인과성 정렬의 기초.
  - 링크: [dl.acm.org/doi/10.1145/359545.359563](https://dl.acm.org/doi/10.1145/359545.359563)

## 4. 알고리즘 / 그래프 古典

- **A Note on Two Problems in Connexion with Graphs** (E. W. Dijkstra, Numerische Mathematik, 1959) ⭐
  - 주제: 최단 경로·최소 신장 트리 — Dijkstra 알고리즘 원전.
  - 링크: [dl.acm.org/doi/10.1007/BF01386390](https://dl.acm.org/doi/10.1007/BF01386390)
- **Cooperating Sequential Processes** (E. W. Dijkstra, EWD123, 1965) ⭐ 🆓
  - 주제: 임계 구역·세마포어·동기화 — 동시성 이론의 출발점.
  - 링크: [EWD Archive](https://www.cs.utexas.edu/~EWD/transcriptions/EWD01xx/EWD123.html) 🆓

## 5. 프로그래밍 언어 / 패러다임 (Languages & Paradigms)

- **Recursive Functions of Symbolic Expressions and Their Computation by Machine, Part I** (J. McCarthy, CACM, 1960) ⭐
  - 주제: LISP·기호 처리·재귀 — 함수형 언어와 AI 언어의 효시.
  - 링크: [dl.acm.org/doi/10.1145/367177.367199](https://dl.acm.org/doi/10.1145/367177.367199)
- **Go To Statement Considered Harmful** (E. W. Dijkstra, CACM, 1968) ⭐ 🆓
  - 주제: 구조적 프로그래밍 — 제어 흐름 규율의 시발점.
  - 링크: [EWD215 Archive](https://www.cs.utexas.edu/~EWD/transcriptions/EWD02xx/EWD215.html) 🆓
- **Program Development by Stepwise Refinement** (N. Wirth, CACM, 1971) ⭐
  - 주제: 단계적 정제 — 구조적 설계 방법론의 정전.
  - 링크: [dl.acm.org/doi/10.1145/362929.362947](https://dl.acm.org/doi/10.1145/362929.362947)
- **Can Programming Be Liberated from the von Neumann Style?** (J. Backus, CACM, 1978) ⭐ 🏆
  - 주제: 함수형 프로그래밍·FP 대수 — 1977 튜링상 강연, 패러다임 비판.
  - 링크: [dl.acm.org/doi/10.1145/1283920.1283933](https://dl.acm.org/doi/10.1145/1283920.1283933)
- **Programming with Abstract Data Types** (B. Liskov, S. Zilles, ACM SIGPLAN, 1974) ⭐
  - 주제: 추상 데이터 타입 — 캡슐화·타입 추상화의 토대.
  - 링크: [dl.acm.org/doi/10.1145/942572.807045](https://dl.acm.org/doi/10.1145/942572.807045)
- **Literate Programming** (D. E. Knuth, The Computer Journal, 1984) ⭐
  - 주제: 문학적 프로그래밍 — 코드·문서 일체화 사상.
  - 링크: [DOI 10.1093/comjnl/27.2.97](https://doi.org/10.1093/comjnl/27.2.97)

## 6. 형식 검증 / 정확성 (Formal Methods)

- **An Axiomatic Basis for Computer Programming** (C. A. R. Hoare, CACM, 1969) ⭐ 🏆
  - 주제: Hoare 논리·전후 조건 — 프로그램 정확성 증명의 기초.
  - 링크: [dl.acm.org/doi/10.1145/363235.363259](https://dl.acm.org/doi/10.1145/363235.363259)
- **Proof of a Program: FIND** (C. A. R. Hoare, CACM, 1971)
  - 주제: 실제 알고리즘의 형식 증명 — Hoare 논리의 구체 적용 사례.
  - 링크: [dl.acm.org/doi/10.1145/362452.362489](https://dl.acm.org/doi/10.1145/362452.362489)
- **Assigning Meanings to Programs** (R. W. Floyd, 1967) ⭐
  - 주제: 흐름도 단언·귀납 — 프로그램 의미론·검증의 선구.
  - 링크: [DOI 10.1090/psapm/019](https://doi.org/10.1090/psapm/019)
- **Guarded Commands, Nondeterminacy and Formal Derivation of Programs** (E. W. Dijkstra, CACM, 1975) ⭐
  - 주제: 보호 명령·약최약 선조건 — 프로그램 유도 calculus.
  - 링크: [dl.acm.org/doi/10.1145/360933.360975](https://dl.acm.org/doi/10.1145/360933.360975)

## 7. 데이터베이스 / 데이터 모델

- **A Relational Model of Data for Large Shared Data Banks** (E. F. Codd, CACM, 1970) ⭐ 🏆
  - 주제: 관계 모델·정규화·데이터 독립성 — 관계형 DB의 창시.
  - 링크: [dl.acm.org/doi/10.1145/362384.362685](https://dl.acm.org/doi/10.1145/362384.362685)

## 8. 소프트웨어 공학 古典

- **On the Criteria To Be Used in Decomposing Systems into Modules** (D. L. Parnas, CACM, 1972) ⭐
  - 주제: 정보 은닉 기준 모듈화 — 모듈 설계 원리의 정전.
  - 링크: [dl.acm.org/doi/10.1145/361598.361623](https://dl.acm.org/doi/10.1145/361598.361623)
- **No Silver Bullet — Essence and Accidents of Software Engineering** (F. P. Brooks, IEEE Computer, 1987) ⭐
  - 주제: 본질적 복잡성 vs 우발적 복잡성 — SW 생산성의 한계 진단.
  - 링크: [DOI 10.1109/MC.1987.1663532](https://doi.org/10.1109/MC.1987.1663532)
- **The Mythical Man-Month** (F. P. Brooks, 1975, 참고) ⭐ 📖
  - 주제: Brooks의 법칙·SW 프로젝트 관리 — 공정 관리 古典(저서).
  - 링크: [출판사 페이지](https://www.oreilly.com/library/view/mythical-man-month-the/0201835959/)

## 9. 인공지능 / 튜링상 강연 (AI & Turing Lectures)

- **Computing Machinery and Intelligence** (A. M. Turing, Mind, 1950) ⭐ 🆓
  - 주제: 튜링 테스트·기계 지능 — AI 철학의 출발점.
  - 링크: [DOI 10.1093/mind/LIX.236.433](https://doi.org/10.1093/mind/LIX.236.433)
- **The Emperor's Old Clothes** (C. A. R. Hoare, CACM, 1981) ⭐ 🏆 🆓
  - 주제: 단순성·신뢰성 설교 — 1980 튜링상 강연, 엔지니어링 교훈.
  - 링크: [amturing.acm.org Hoare](https://amturing.acm.org/award_winners/hoare_4622167.cfm) 🆓
- **The Humble Programmer** (E. W. Dijkstra, CACM, 1972) ⭐ 🏆 🆓
  - 주제: 프로그래밍의 지적 겸손 — 1972 튜링상 강연.
  - 링크: [EWD340 Archive](https://www.cs.utexas.edu/~EWD/transcriptions/EWD03xx/EWD340.html) 🆓
- **ACM Turing Award Lectures (landmark set)** (amturing.acm.org, 참고 인덱스) ⭐ 🆓
  - 주제: 분야별 정전 강연 모음 — 古典 논문 탐색의 출발 허브.
  - 링크: [amturing.acm.org](https://amturing.acm.org/) 🆓

## 관련
- [[papers]] — Papers hub
- [[theory-algorithms]] — 계산 이론·알고리즘 심화
- [[programming-languages]] — 언어 설계·타입·의미론
- [[software-engineering-formal]] — 형식 검증·모듈화·SW 공학
- [[distributed-systems-db]] — 분산 시스템·데이터베이스
- [[../books/computer-science/computer-science]] — CS 도서 카탈로그
