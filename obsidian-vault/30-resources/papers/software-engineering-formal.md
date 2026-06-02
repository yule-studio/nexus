---
title: "Software Engineering / Formal Methods — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, software-engineering, formal-methods, testing]
home_hub: resources
---

# Software Engineering / Formal Methods — Papers

**[[papers|↑ Papers hub]]**

> 프로그램의 정확성을 증명·검증하고 (model checking, abstract interpretation, proof) 결함을 자동으로 찾아내는 (symbolic execution, fuzzing, property testing) SE/형식 방법론의 정전 논문 모음.

## 1. 프로그램 의미론 / 정적 증명 기반

- **Assigning Meanings to Programs** (Robert W. Floyd, Proc. Symp. Applied Math, 1967) ⭐
  - 주제: 프로그램 지점에 단언을 붙여 정확성을 증명하는 inductive assertion 의 원류
  - 링크: [Floyd 1967 (PDF)](https://www.cs.virginia.edu/~weimer/2007-615/reading/FloydMeaning.pdf)
- **An Axiomatic Basis for Computer Programming** (C.A.R. Hoare, CACM, 1969) ⭐
  - 주제: Hoare triple `{P} C {Q}` 로 프로그램 의미를 형식화한 Hoare logic (cross-ref foundations)
  - 링크: [dl.acm.org/doi/10.1145/363235.363259](https://dl.acm.org/doi/10.1145/363235.363259)
- **Guarded Commands, Nondeterminacy and Formal Derivation of Programs** (E.W. Dijkstra, CACM, 1975) ⭐
  - 주제: weakest precondition 의미론과 프로그램의 형식적 유도
  - 링크: [dl.acm.org/doi/10.1145/360933.360975](https://dl.acm.org/doi/10.1145/360933.360975)
- **On the Criteria To Be Used in Decomposing Systems into Modules** (D.L. Parnas, CACM, 1972) ⭐
  - 주제: information hiding 기반 모듈 분해 — 소프트웨어 설계의 기초 (cross-ref foundations)
  - 링크: [dl.acm.org/doi/10.1145/361598.361623](https://dl.acm.org/doi/10.1145/361598.361623)

## 2. Abstract Interpretation

- **Abstract Interpretation: A Unified Lattice Model for Static Analysis of Programs by Construction or Approximation of Fixpoints** (P. Cousot, R. Cousot, POPL, 1977) ⭐ 📖
  - 주제: 정적 분석을 sound 근사로 통합한 추상 해석 이론의 출발점
  - 링크: [dl.acm.org/doi/10.1145/512950.512973](https://dl.acm.org/doi/10.1145/512950.512973)
- **Systematic Design of Program Analysis Frameworks** (P. Cousot, R. Cousot, POPL, 1979) ⭐
  - 주제: 추상 도메인과 Galois connection 으로 분석기를 체계적으로 설계
  - 링크: [dl.acm.org/doi/10.1145/567752.567778](https://dl.acm.org/doi/10.1145/567752.567778)

## 3. Model Checking

- **Design and Synthesis of Synchronization Skeletons Using Branching Time Temporal Logic** (E.M. Clarke, E.A. Emerson, Workshop on Logics of Programs, 1981) ⭐
  - 주제: CTL 명세를 자동 검증하는 model checking 의 기원
  - 링크: [doi.org/10.1007/BFb0025774](https://doi.org/10.1007/BFb0025774)
- **Specification and Verification of Concurrent Systems in CESAR** (J.P. Queille, J. Sifakis, Intl. Symp. Programming, 1982) ⭐
  - 주제: model checking 을 동시 발견한 또 다른 원류 (CESAR)
  - 링크: [doi.org/10.1007/3-540-11494-7_22](https://doi.org/10.1007/3-540-11494-7_22)
- **Symbolic Model Checking: 10^20 States and Beyond** (Burch, Clarke, McMillan, Dill, Hwang, LICS/Info.& Comp., 1992) ⭐
  - 주제: BDD 기반 symbolic model checking 으로 상태 폭발 완화
  - 링크: [dl.acm.org/doi/10.1006/inco.1992.1025](https://dl.acm.org/doi/10.1006/inco.1992.1025)
- **The Model Checker SPIN** (G.J. Holzmann, IEEE TSE, 1997) ⭐ 🆓
  - 주제: PROMELA + LTL 기반 동시 시스템 검증기 SPIN
  - 링크: [spinroot.com — Holzmann 1997 (PDF)](https://spinroot.com/spin/Doc/ieee97.pdf) 🆓

## 4. Temporal Logic / TLA+

- **The Temporal Logic of Programs** (Amir Pnueli, FOCS, 1977) ⭐
  - 주제: 반응형/동시 시스템에 temporal logic 을 도입 (Turing Award)
  - 링크: [dl.acm.org/doi/10.1109/SFCS.1977.32](https://dl.acm.org/doi/10.1109/SFCS.1977.32)
- **The Temporal Logic of Actions** (Leslie Lamport, ACM TOPLAS, 1994) ⭐
  - 주제: 상태+동작을 하나의 논리로 명세하는 TLA
  - 링크: [lamport.azurewebsites.net — TLA](https://lamport.azurewebsites.net/pubs/lamport-actions.pdf) 🆓
- **Specifying Systems: The TLA+ Language and Tools** (Leslie Lamport, Addison-Wesley, 2002) ⭐ 🆓 📖
  - 주제: TLA+ 명세 언어와 TLC 모델 체커 레퍼런스 (book)
  - 링크: [lamport.azurewebsites.net — TLA+ book](https://lamport.azurewebsites.net/tla/book.html) 🆓
- **How Amazon Web Services Uses Formal Methods** (Newcombe et al., CACM, 2015) 🆓
  - 주제: TLA+ 를 대규모 분산 시스템 설계에 실제 적용한 산업 사례
  - 링크: [dl.acm.org/doi/10.1145/2699417](https://dl.acm.org/doi/10.1145/2699417)

## 5. SAT/SMT 및 정밀 정적 분석

- **Z3: An Efficient SMT Solver** (L. de Moura, N. Bjørner, TACAS, 2008) ⭐
  - 주제: 검증·기호 실행의 핵심 엔진이 된 SMT 솔버 Z3
  - 링크: [doi.org/10.1007/978-3-540-78800-3_24](https://doi.org/10.1007/978-3-540-78800-3_24)
- **Chaff: Engineering an Efficient SAT Solver** (Moskewicz et al., DAC, 2001) ⭐
  - 주제: VSIDS/watched literals 로 현대 CDCL SAT 솔버 성능을 끌어올림
  - 링크: [dl.acm.org/doi/10.1145/378239.379017](https://dl.acm.org/doi/10.1145/378239.379017)
- **The SLAM Project: Debugging System Software via Static Analysis** (T. Ball, S.K. Rajamani, POPL, 2002) ⭐
  - 주제: 술어 추상화 기반 소프트웨어 model checking (Windows 드라이버 검증)
  - 링크: [dl.acm.org/doi/10.1145/503272.503274](https://dl.acm.org/doi/10.1145/503272.503274)
- **Checking System Rules Using System-Specific, Programmer-Written Compiler Extensions** (Engler et al., OSDI, 2000) ⭐ 🏆
  - 주제: 시스템 규칙을 컴파일러 확장으로 검사하는 정적 분석 (Meta-level compilation)
  - 링크: [usenix.org/conference/osdi-2000](https://www.usenix.org/conference/osdi-2000) 🆓
- **Bugs as Deviant Behavior: A General Approach to Inferring Errors in Systems Code** (Engler et al., SOSP, 2001) ⭐
  - 주제: 코드의 통계적 일탈을 결함으로 추론하는 명세-없는 분석
  - 링크: [dl.acm.org/doi/10.1145/502034.502041](https://dl.acm.org/doi/10.1145/502034.502041)

## 6. Symbolic Execution & Whitebox Test 생성

- **Symbolic Execution and Program Testing** (James C. King, CACM, 1976) ⭐
  - 주제: 입력을 기호값으로 실행해 경로 조건을 푸는 기호 실행의 원조
  - 링크: [dl.acm.org/doi/10.1145/360248.360252](https://dl.acm.org/doi/10.1145/360248.360252)
- **DART: Directed Automated Random Testing** (Godefroid, Klarlund, Sen, PLDI, 2005) ⭐
  - 주제: 구체 실행과 기호 제약을 결합한 concolic 테스트
  - 링크: [dl.acm.org/doi/10.1145/1065010.1065036](https://dl.acm.org/doi/10.1145/1065010.1065036)
- **KLEE: Unassisted and Automatic Generation of High-Coverage Tests for Complex Systems Programs** (Cadar, Dunbar, Engler, OSDI, 2008) ⭐ 🏆 🆓
  - 주제: 실제 시스템 프로그램에서 고커버리지 테스트를 자동 생성한 기호 실행기
  - 링크: [usenix.org/conference/osdi-08 — KLEE](https://www.usenix.org/conference/osdi-08/presentation/klee-unassisted-and-automatic-generation-high-coverage-tests-complex) 🆓
- **Automated Whitebox Fuzz Testing (SAGE)** (Godefroid, Levin, Molnar, NDSS, 2008) ⭐
  - 주제: 기호 실행으로 입력 파일을 진화시키는 화이트박스 퍼징 (SAGE)
  - 링크: [microsoft.com/research — SAGE](https://www.microsoft.com/en-us/research/publication/automated-whitebox-fuzz-testing/) 🆓

## 7. Fuzzing

- **An Empirical Study of the Reliability of UNIX Utilities** (Miller, Fredriksen, So, CACM, 1990) ⭐ 🆓
  - 주제: 무작위 입력으로 유닉스 유틸리티를 무더기로 crash 시킨 fuzzing 의 원점
  - 링크: [dl.acm.org/doi/10.1145/96267.96279](https://dl.acm.org/doi/10.1145/96267.96279)
- **AFL — american fuzzy lop** (Michał Zalewski, technical doc, 2014) ⭐ 🆓
  - 주제: 커버리지 유도 진화 퍼징의 사실상 표준 (coverage-guided fuzzing)
  - 링크: [lcamtuf.coredump.cx/afl](https://lcamtuf.coredump.cx/afl/) 🆓
- **The Art, Science, and Engineering of Fuzzing: A Survey** (Manès et al., IEEE TSE, 2019) 🆕 📖
  - 주제: 현대 fuzzing (libFuzzer/OSS-Fuzz 포함) 의 분류와 설계 공간 정리
  - 링크: [dl.acm.org/doi/10.1109/TSE.2019.2946563](https://dl.acm.org/doi/10.1109/TSE.2019.2946563) 🆕
- **Coverage-based Greybox Fuzzing as Markov Chain (AFLFast)** (Böhme, Pham, Roychoudhury, CCS, 2016) 🏆
  - 주제: 그레이박스 퍼징을 Markov chain 으로 모델링해 에너지 배분 최적화
  - 링크: [dl.acm.org/doi/10.1145/2976749.2978428](https://dl.acm.org/doi/10.1145/2976749.2978428)

## 8. Property-based & Mutation Testing, Delta Debugging

- **QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs** (K. Claessen, J. Hughes, ICFP, 2000) ⭐
  - 주제: 명세를 속성으로 기술하고 입력을 자동 생성하는 property-based testing
  - 링크: [dl.acm.org/doi/10.1145/351240.351266](https://dl.acm.org/doi/10.1145/351240.351266)
- **Hints on Test Data Selection: Help for the Practicing Programmer (Mutation Testing)** (DeMillo, Lipton, Sayward, IEEE Computer, 1978) ⭐
  - 주제: 코드를 변이시켜 테스트 스위트 적절성을 측정하는 mutation testing
  - 링크: [dl.acm.org/doi/10.1109/C-M.1978.218136](https://dl.acm.org/doi/10.1109/C-M.1978.218136)
- **Simplifying and Isolating Failure-Inducing Input (Delta Debugging)** (A. Zeller, R. Hildebrandt, IEEE TSE, 2002) ⭐
  - 주제: 실패를 유발하는 최소 입력으로 자동 축소하는 ddmin 알고리즘
  - 링크: [dl.acm.org/doi/10.1109/32.988498](https://dl.acm.org/doi/10.1109/32.988498)

## 9. 검증된 시스템 / Proof-Carrying Code / 증명 보조기

- **Formal Verification of a Realistic Compiler (CompCert)** (Xavier Leroy, CACM, 2009) ⭐ 🏆 🆓
  - 주제: 의미 보존을 Coq 로 증명한 실용 C 컴파일러 CompCert
  - 링크: [compcert.org — Leroy CACM 2009 (PDF)](https://xavierleroy.org/publi/compcert-CACM.pdf) 🆓
- **seL4: Formal Verification of an OS Kernel** (Klein et al., SOSP, 2009) ⭐ 🏆 🆓
  - 주제: OS 마이크로커널 구현을 Isabelle/HOL 로 전수 검증한 최초 사례
  - 링크: [sel4.systems — seL4 SOSP 2009](https://sel4.systems/Research/pubs.html) 🆓
- **Proof-Carrying Code** (George C. Necula, POPL, 1997) ⭐
  - 주제: 코드에 안전성 증명을 동봉해 수신자가 검증하게 하는 PCC
  - 링크: [dl.acm.org/doi/10.1145/263699.263712](https://dl.acm.org/doi/10.1145/263699.263712)
- **Formal Proof — The Four Color Theorem** (Georges Gonthier, Notices of the AMS, 2008) ⭐ 🆓
  - 주제: 4색 정리를 Coq 로 완전 형식 증명 (증명 보조기의 위력)
  - 링크: [ams.org — Gonthier 2008 (PDF)](https://www.ams.org/notices/200811/tx081101382p.pdf) 🆓
- **Isabelle/HOL: A Proof Assistant for Higher-Order Logic** (Nipkow, Paulson, Wenzel, Springer LNCS, 2002) 📖
  - 주제: 널리 쓰이는 고차 논리 증명 보조기 Isabelle/HOL 레퍼런스
  - 링크: [doi.org/10.1007/3-540-45949-9](https://doi.org/10.1007/3-540-45949-9)

## 10. 개발 프로세스 (cross-ref)

- **Continuous Integration** (Martin Fowler, online essay, 2006) 🆓
  - 주제: 자주 통합·자동 빌드/테스트하는 CI 의 정의와 실천
  - 링크: [martinfowler.com/articles/continuousIntegration.html](https://martinfowler.com/articles/continuousIntegration.html) 🆓
- **Extreme Programming Explained: Embrace Change** (Kent Beck, Addison-Wesley, 1999) 📖
  - 주제: TDD·짧은 반복·지속 통합을 묶은 agile 실천의 원전 (book)
  - 링크: [dl.acm.org — XP Explained](https://dl.acm.org/doi/book/10.5555/318762)

## 관련
- [[papers]] — Papers hub
- [[programming-languages]] — 타입/의미론/컴파일러 정전
- [[foundations-classics]] — Floyd/Hoare/Parnas 기초
- [[systems-os-network]] — seL4 등 검증된 시스템
- [[security-crypto]] — fuzzing/whitebox 가 닿는 보안
- [[../books/testing/testing]] — 테스트 실무서
