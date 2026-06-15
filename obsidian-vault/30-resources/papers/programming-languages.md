---
title: "Programming Languages / Compilers — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, programming-languages, compilers, types]
home_hub: resources
---

# Programming Languages / Compilers — Papers

**[[papers|↑ Papers hub]]**

> 타입 이론·람다 계산·의미론·컴파일러·메모리 관리의 정전(canon)을 모은 PL 논문 카탈로그.

## 1. 타입 추론 / 다형성 (Type Inference & Polymorphism)

- **A Theory of Type Polymorphism in Programming** (Robin Milner, JCSS, 1978) ⭐ 🆓
  - 주제: Hindley-Milner 타입 시스템과 Algorithm W 의 토대
  - 링크: [JCSS DOI](https://doi.org/10.1016/0022-0000(78)90014-4)
- **Principal Type-Schemes for Functional Programs** (Luis Damas, Robin Milner, POPL, 1982) ⭐ 🆓
  - 주제: Damas-Milner 알고리즘과 principal type 정리
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/582153.582176)
- **Towards a Theory of Type Structure** (John C. Reynolds, 1974) ⭐
  - 주제: System F (다형 람다 계산)의 독립적 발견
  - 링크: [Springer DOI](https://doi.org/10.1007/3-540-06859-7_148)
- **Interprétation fonctionnelle et élimination des coupures** (Jean-Yves Girard, 1972) ⭐
  - 주제: System F / 2차 람다 계산의 원전, 정규화 증명
  - 링크: [Girard 자료](https://girard.perso.math.cnrs.fr/These.pdf) 🆓

## 2. 타입과 데이터 추상 (Types, Abstraction & Subtyping)

- **On Understanding Types, Data Abstraction, and Polymorphism** (Luca Cardelli, Peter Wegner, ACM Computing Surveys, 1985) ⭐ 📖
  - 주제: parametric/inclusion/ad-hoc 다형성과 bounded quantification 정리
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/6041.6042)
- **A Behavioral Notion of Subtyping** (Barbara Liskov, Jeannette Wing, TOPLAS, 1994) ⭐
  - 주제: LSP(리스코프 치환 원칙)의 형식적 정의
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/197320.197383)
- **Featherweight Java: A Minimal Core Calculus for Java and GJ** (Atsushi Igarashi, Benjamin Pierce, Philip Wadler, TOPLAS, 2001) ⭐
  - 주제: 제네릭 자바 증명을 위한 최소 객체 계산
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/503502.503505)
- **Gradual Typing for Functional Languages** (Jeremy Siek, Walid Taha, Scheme Workshop, 2006) ⭐ 🆓
  - 주제: 정적·동적 타입을 한 언어에 통합하는 gradual typing
  - 링크: [PDF](http://scheme2006.cs.uchicago.edu/13-siek.pdf)
- **Types and Programming Languages** (Benjamin C. Pierce, MIT Press, 2002) ⭐ 📖
  - 주제: 타입 시스템 표준 교과서 (TAPL) — 참고 레퍼런스
  - 링크: [MIT Press](https://mitpress.mit.edu/9780262162098/types-and-programming-languages/)

## 3. 함수형 / 모나드 / 람다 (Functional & Lambda Calculus)

- **Definitional Interpreters for Higher-Order Programming Languages** (John C. Reynolds, ACM, 1972) ⭐ 🆓
  - 주제: defunctionalization, CPS, meta-circular interpreter의 원전
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/800194.805852)
- **Comprehending Monads** (Philip Wadler, LFP, 1990) ⭐ 🆓
  - 주제: monad을 통한 함수형 프로그램 구조화
  - 링크: [Wadler 자료](https://homepages.inf.ed.ac.uk/wadler/papers/monads/monads.ps)
- **Monads for Functional Programming** (Philip Wadler, 1995) ⭐ 🆓 📖
  - 주제: 상태/예외/IO를 모나드로 다루는 튜토리얼
  - 링크: [Wadler 자료](https://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf)
- **Theorems for Free!** (Philip Wadler, FPCA, 1989) ⭐ 🆓
  - 주제: parametricity 로부터 무료로 얻는 정리 (free theorems)
  - 링크: [Wadler 자료](https://homepages.inf.ed.ac.uk/wadler/papers/free/free.ps)
- **Propositions as Types** (Philip Wadler, CACM, 2015) ⭐ 🆓
  - 주제: Curry-Howard 대응의 현대적 정리
  - 링크: [Wadler 자료](https://homepages.inf.ed.ac.uk/wadler/papers/propositions-as-types/propositions-as-types.pdf)
- **The Art of the Interpreter, or, the Modularity Complex** (Guy Steele, Gerald Sussman, MIT AI Memo, 1978) ⭐ 🆓
  - 주제: Scheme/Lambda 논문 — 클로저·평가기 설계의 정수
  - 링크: [MIT DSpace](https://dspace.mit.edu/handle/1721.1/6094)

## 4. 의미론 (Operational & Program Semantics)

- **A Structural Approach to Operational Semantics** (Gordon Plotkin, 기술보고서, 1981) ⭐ 🆓 📖
  - 주제: SOS(구조적 작동 의미론)의 정초 보고서
  - 링크: [Plotkin SOS PDF](https://homepages.inf.ed.ac.uk/gdp/publications/sos_jlap.pdf)
- **Call-by-Name, Call-by-Value and the λ-Calculus** (Gordon Plotkin, TCS, 1975) ⭐ 🆓
  - 주제: CBN/CBV 평가 전략과 λ_v 계산
  - 링크: [Elsevier DOI](https://doi.org/10.1016/0304-3975(75)90017-1)

## 5. 프로그램 논리 / 검증 (Program Logic & Verification)

- **Separation Logic: A Logic for Shared Mutable Data Structures** (John C. Reynolds, LICS, 2002) ⭐ 🏆
  - 주제: 힙·포인터 추론을 위한 분리 논리
  - 링크: [IEEE DOI](https://doi.org/10.1109/LICS.2002.1029817)
- **BI as an Assertion Language for Mutable Data Structures** (Samin Ishtiaq, Peter O'Hearn, POPL, 2001) ⭐
  - 주제: 분리 논리의 기반이 된 bunched implications
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/360204.375719)
- **RustBelt: Securing the Foundations of the Rust Programming Language** (R. Jung, J.-H. Jourdan, R. Krebbers, D. Dreyer, POPL, 2018) ⭐ 🏆 🆓
  - 주제: unsafe 포함 Rust 부분집합의 첫 기계 검증 안전성 증명 (Iris)
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/3158154)
- **Abstract Interpretation: A Unified Lattice Model** (Patrick Cousot, Radhia Cousot, POPL, 1977) ⭐ 📖
  - 주제: 정적 분석 이론의 토대 — 추상 해석
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/512950.512973)

## 6. 컴파일러 / 중간 표현 (Compilers & IR)

- **Efficiently Computing Static Single Assignment Form and the Control Dependence Graph** (Cytron, Ferrante, Rosen, Wegman, Zadeck, TOPLAS, 1991) ⭐
  - 주제: SSA 형식과 지배 경계(dominance frontier) 알고리즘
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/115372.115320)
- **LLVM: A Compilation Framework for Lifelong Program Analysis & Transformation** (Chris Lattner, Vikram Adve, CGO, 2004) ⭐ 🆓
  - 주제: LLVM IR와 전 생애주기 최적화 프레임워크
  - 링크: [LLVM Publications](https://llvm.org/pubs/2004-01-30-CGO-LLVM.html)
- **MLIR: A Compiler Infrastructure for the End of Moore's Law** (Lattner et al., CGO, 2021) 🆕 🆓
  - 주제: 다단계 중간 표현(dialect)을 위한 컴파일러 인프라
  - 링크: [arXiv:2002.11054](https://arxiv.org/abs/2002.11054)
- **Trace-based Just-in-Time Type Specialization for Dynamic Languages** (Gal et al., PLDI, 2009) ⭐
  - 주제: 동적 언어 trace JIT (TraceMonkey)
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/1542476.1542528)
- **Tracing the Meta-Level: PyPy's Tracing JIT Compiler** (Bolz, Cuni, Fijałkowski, Rigo, ICOOOLPS, 2009) 🆓
  - 주제: 인터프리터를 추적하는 메타-트레이싱 (PyPy)
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/1565824.1565827)

## 7. 메모리 관리 / GC (Memory Management & GC)

- **A Real-Time Garbage Collector Based on the Lifetimes of Objects** (Henry Lieberman, Carl Hewitt, CACM, 1983) ⭐
  - 주제: generational GC의 원전 — 객체 수명 기반 수집
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/358141.358147)
- **Generation Scavenging: A Non-disruptive High Performance Storage Reclamation Algorithm** (David Ungar, SDE, 1984) ⭐
  - 주제: 실용적 세대별 GC 구현 (Generation Scavenging)
  - 링크: [ACM DOI](https://dl.acm.org/doi/10.1145/800020.808261)
- **Region-Based Memory Management** (Mads Tofte, Jean-Pierre Talpin, Information and Computation, 1997) ⭐
  - 주제: GC 없이 영역(region)으로 메모리를 정적 관리
  - 링크: [Elsevier DOI](https://doi.org/10.1006/inco.1996.2613)
- **Simple Generational Garbage Collection and Fast Allocation** (Andrew Appel, Software: P&E, 1989) 🆓
  - 주제: 함수형 언어용 빠른 할당과 단순 세대별 GC
  - 링크: [Princeton 자료](https://www.cs.princeton.edu/~appel/papers/143.pdf)

## 8. 참고 교과서 (Reference Texts)

- **Compilers: Principles, Techniques, and Tools** (Aho, Lam, Sethi, Ullman, 2판, 2006) ⭐ 📖
  - 주제: 컴파일러 표준 교과서 (Dragon Book) — 파싱/최적화/코드 생성
  - 링크: [Pearson](https://www.pearson.com/en-us/subject-catalog/p/compilers-principles-techniques-and-tools/P200000003472)
- **Modern Compiler Implementation in ML** (Andrew Appel, Cambridge, 1998) 📖
  - 주제: SSA·레지스터 할당까지 다루는 실전 컴파일러 구현
  - 링크: [Princeton 책 페이지](https://www.cs.princeton.edu/~appel/modern/ml/)

## 관련
- [[papers]] — Papers hub
- [[foundations-classics]] — 계산 이론·람다 계산 고전
- [[theory-algorithms]] — 이론/알고리즘 논문
- [[software-engineering-formal]] — 형식 방법·검증
- [[../books/computer-science/computer-science]] — CS 도서 카탈로그
