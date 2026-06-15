---
title: "Theory / Algorithms / Complexity — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, theory, algorithms, complexity]
home_hub: resources
---

# Theory / Algorithms / Complexity — Papers

**[[papers|↑ Papers hub]]**

> 계산 복잡도와 알고리즘 설계의 토대를 세운 핵심 논문 모음 — NP-완전성, 자료구조, 무작위·근사·스트리밍 알고리즘까지.

## 1. 복잡도 이론의 기초 (Foundations of Complexity)

- **On the Computational Complexity of Algorithms** (Hartmanis, Stearns, *Trans. AMS*, 1965) ⭐
  - 주제: 시간 복잡도 계층(time hierarchy)을 정의해 복잡도 이론을 창시
  - 링크: [AMS doi](https://doi.org/10.1090/S0002-9947-1965-0170805-7)
- **The Intrinsic Computational Difficulty of Functions** (Cobham, *Logic, Methodology and Phil. of Science*, 1965) ⭐
  - 주제: 다항 시간(P)을 "다루기 쉬움"의 기준으로 처음 제안
  - 링크: [dblp landing](https://dblp.org/rec/conf/lmps/Cobham65.html)
- **Paths, Trees, and Flowers** (Edmonds, *Canadian J. Math.*, 1965) ⭐ 🆓
  - 주제: 매칭 알고리즘과 함께 다항 시간 = 효율의 개념을 정식화
  - 링크: [Cambridge doi](https://doi.org/10.4153/CJM-1965-045-4)

## 2. NP-완전성 (NP-Completeness)

- **The Complexity of Theorem-Proving Procedures** (Cook, *STOC*, 1971) ⭐ 🏆
  - 주제: SAT의 NP-완전성을 증명, Cook–Levin 정리
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/800157.805047)
- **Reducibility Among Combinatorial Problems** (Karp, 1972) ⭐ 🆓
  - 주제: 21개 조합 문제가 NP-완전임을 보여 영향력 확산
  - 링크: [Berkeley PDF](https://people.eecs.berkeley.edu/~luca/cs172/karp.pdf)
- **Universal Search Problems** (Levin, *Probl. Peredachi Inf.*, 1973) ⭐
  - 주제: Cook과 독립적으로 NP-완전성(보편 탐색 문제)을 정식화
  - 링크: [Boston Univ PDF](https://www.cs.bu.edu/fac/lnd/expo/levin/Universal.pdf)

## 3. 정렬·검색·문자열 (Sorting, Searching, Strings)

- **Quicksort** (Hoare, *The Computer Journal*, 1962) ⭐
  - 주제: 분할 정복 기반 평균 O(n log n) 정렬의 고전
  - 링크: [Oxford doi](https://doi.org/10.1093/comjnl/5.1.10)
- **Fast Pattern Matching in Strings** (Knuth, Morris, Pratt, *SIAM J. Comput.*, 1977) ⭐
  - 주제: 실패 함수로 선형 시간 문자열 매칭(KMP)
  - 링크: [SIAM doi](https://doi.org/10.1137/0206024)
- **A Fast String Searching Algorithm** (Boyer, Moore, *CACM*, 1977) ⭐
  - 주제: 뒤에서부터 비교해 sublinear 평균을 내는 매칭
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/359842.359859)
- **An O(ND) Difference Algorithm and Its Variations** (Myers, *Algorithmica*, 1986)
  - 주제: diff/편집 거리의 실용적 알고리즘
  - 링크: [Springer doi](https://doi.org/10.1007/BF01840446)

## 4. 자료구조 (Data Structures)

- **Self-Adjusting Binary Search Trees** (Sleator, Tarjan, *JACM*, 1985) ⭐
  - 주제: Splay tree, amortized 분석의 정수
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/3828.3835)
- **Fibonacci Heaps and Their Uses in Improved Network Optimization Algorithms** (Fredman, Tarjan, *JACM*, 1987) ⭐
  - 주제: Fibonacci heap으로 Dijkstra/MST를 개선
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/28869.28874)
- **Skip Lists: A Probabilistic Alternative to Balanced Trees** (Pugh, *CACM*, 1990) ⭐ 🆓
  - 주제: 확률적 균형 자료구조, 구현이 단순
  - 링크: [CMU course PDF](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf)
- **Design and Implementation of an Efficient Priority Queue** (van Emde Boas, *Math. Systems Theory*, 1977)
  - 주제: vEB tree, O(log log u) 우선순위 큐
  - 링크: [Springer doi](https://doi.org/10.1007/BF01683268)
- **Amortized Computational Complexity** (Tarjan, *SIAM J. Alg. Disc. Meth.*, 1985) ⭐
  - 주제: amortized 분석의 정식 틀(potential method)
  - 링크: [SIAM doi](https://doi.org/10.1137/0606031)

## 5. 해싱·스트리밍·확률 자료구조 (Hashing & Streaming)

- **Space/Time Trade-offs in Hash Coding with Allowable Errors** (Bloom, *CACM*, 1970) ⭐
  - 주제: Bloom filter — 확률적 집합 멤버십
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/362686.362692)
- **Consistent Hashing and Random Trees** (Karger, Lehman, Leighton, Levine, Lewin, Panigrahy, *STOC*, 1997) ⭐
  - 주제: 분산 캐시/샤딩의 기반인 consistent hashing
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/258533.258660)
- **Probabilistic Counting Algorithms for Data Base Applications** (Flajolet, Martin, *JCSS*, 1985) ⭐
  - 주제: 카디널리티 추정(FM sketch), HyperLogLog의 뿌리
  - 링크: [Elsevier doi](https://doi.org/10.1016/0022-0000(85)90041-8)
- **HyperLogLog: The Analysis of a Near-Optimal Cardinality Estimation Algorithm** (Flajolet, Fusy, Gandouet, Meunier, *AofA*, 2007) 🆓
  - 주제: 작은 메모리로 distinct count 추정
  - 링크: [HAL PDF](https://hal.science/file/index/docid/406166/filename/FlFuGaMe07.pdf)
- **An Improved Data Stream Summary: The Count-Min Sketch** (Cormode, Muthukrishnan, *J. Algorithms*, 2005) ⭐
  - 주제: 스트리밍 빈도 추정 sketch
  - 링크: [Elsevier doi](https://doi.org/10.1016/j.jalgor.2003.12.001)

## 6. 그래프·수치 알고리즘 (Graph & Numerical Algorithms)

- **A Note on Two Problems in Connexion with Graphs** (Dijkstra, *Numer. Math.*, 1959) ⭐
  - 주제: 최단 경로 알고리즘의 고전
  - 링크: [Springer doi](https://doi.org/10.1007/BF01386390)
- **Algorithm 97: Shortest Path** (Floyd, *CACM*, 1962) ⭐
  - 주제: 모든 쌍 최단 경로(Floyd–Warshall)
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/367766.368168)
- **On a Routing Problem** (Bellman, *Quart. Appl. Math.*, 1958) ⭐
  - 주제: 음수 간선을 다루는 최단 경로(Bellman–Ford)
  - 링크: [AMS doi](https://doi.org/10.1090/qam/102435)
- **An Algorithm for the Machine Calculation of Complex Fourier Series** (Cooley, Tukey, *Math. Comp.*, 1965) ⭐
  - 주제: FFT — O(n log n) 이산 푸리에 변환
  - 링크: [AMS doi](https://doi.org/10.1090/S0025-5718-1965-0178586-1)

## 7. 무작위·근사·암호 알고리즘 (Randomized, Approximation, Crypto)

- **Probabilistic Algorithm for Testing Primality** (Rabin, *J. Number Theory*, 1980) ⭐
  - 주제: Miller–Rabin 무작위 소수 판정
  - 링크: [Elsevier doi](https://doi.org/10.1016/0022-314X(80)90084-0)
- **A Fast Monte-Carlo Test for Primality** (Solovay, Strassen, *SIAM J. Comput.*, 1977) ⭐
  - 주제: 무작위 소수 판정의 초기 결과
  - 링크: [SIAM doi](https://doi.org/10.1137/0206006)
- **A Simple Unpredictable Pseudo-Random Number Generator** (Blum, Blum, Shub, *SIAM J. Comput.*, 1986) ⭐
  - 주제: 이차 잉여 기반 암호학적 PRNG(BBS)
  - 링크: [SIAM doi](https://doi.org/10.1137/0215025)
- **Improved Approximation Algorithms for Max Cut and Satisfiability Using SDP** (Goemans, Williamson, *JACM*, 1995) ⭐ 🏆
  - 주제: SDP 라운딩으로 0.878 Max-Cut 근사
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/227683.227684)

## 8. 복잡도의 정점 (Hardness, PCP, Counting)

- **PRIMES is in P** (Agrawal, Kayal, Saxena, *Annals of Math.*, 2004) ⭐ 🏆 🆓
  - 주제: AKS — 결정론적 다항 시간 소수 판정
  - 링크: [Princeton Annals PDF](https://annals.math.princeton.edu/wp-content/uploads/annals-v160-n2-p12.pdf)
- **Proof Verification and the Hardness of Approximation Problems** (Arora, Lund, Motwani, Sudan, Szegedy, *JACM*, 1998) ⭐ 🏆 📖
  - 주제: PCP 정리와 근사 불가능성
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/278298.278306)
- **Probabilistic Checking of Proofs: A New Characterization of NP** (Arora, Safra, *JACM*, 1998) ⭐ 🏆
  - 주제: PCP[O(log n), O(1)] = NP
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/273865.273901)
- **The Complexity of Computing the Permanent** (Valiant, *Theor. Comput. Sci.*, 1979) ⭐
  - 주제: #P-완전성과 counting 복잡도
  - 링크: [Elsevier doi](https://doi.org/10.1016/0304-3975(79)90044-6)

## 9. 분석 패러다임 (Analysis Paradigms)

- **Probabilistic Computations: Toward a Unified Measure of Complexity** (Yao, *FOCS*, 1977) ⭐
  - 주제: Yao의 minimax 원리, 통신 복잡도의 토대
  - 링크: [IEEE doi](https://doi.org/10.1109/SFCS.1977.24)
- **Some Complexity Questions Related to Distributive Computing** (Yao, *STOC*, 1979) ⭐
  - 주제: 통신 복잡도 모델의 정식 제안
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/800135.804414)
- **Amortized Efficiency of List Update and Paging Rules** (Sleator, Tarjan, *CACM*, 1985) ⭐
  - 주제: 온라인 알고리즘과 경쟁 분석(competitive analysis)
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/2786.2793)
- **Smoothed Analysis of Algorithms: Why the Simplex Algorithm Usually Takes Polynomial Time** (Spielman, Teng, *JACM*, 2004) ⭐ 🏆 📖
  - 주제: worst/average 사이를 잇는 smoothed analysis
  - 링크: [dl.acm.org doi](https://dl.acm.org/doi/10.1145/990308.990310)

## 관련
- [[papers]] — Papers hub
- [[foundations-classics]] — 계산 모델·튜링·정보이론의 고전
- [[programming-languages]] — 타입·계산·언어 이론 논문
- [[distributed-systems-db]] — consistent hashing·스트리밍이 쓰이는 시스템
- [[ai-ml-papers]] — 근사·무작위 알고리즘이 만나는 학습 이론
- [[../books/computer-science/computer-science]] — CLRS 등 알고리즘 교재
