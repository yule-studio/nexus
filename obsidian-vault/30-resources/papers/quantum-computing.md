---
title: "Quantum Computing — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, quantum, complexity]
home_hub: resources
---

# Quantum Computing — Papers

**[[papers|↑ Papers hub]]**

> 양자 계산/양자 정보의 시조 논문 모음 — 알고리즘, 오류정정, 암호, 복잡도, NISQ 시대까지.

## 1. 기초 / 모델 (Foundations & Model)

- **Simulating Physics with Computers** (Richard Feynman, Int. J. Theoretical Physics, 1982) ⭐
  - 주제: 양자계 시뮬레이션은 양자 컴퓨터로만 효율적이라는 발상의 출발점
  - 링크: [doi.org/10.1007/BF02650179](https://doi.org/10.1007/BF02650179)
- **Quantum Theory, the Church-Turing Principle and the Universal Quantum Computer** (David Deutsch, Proc. R. Soc. Lond. A, 1985) ⭐
  - 주제: 보편 양자 튜링 기계와 Church-Turing 원리의 물리적 재해석
  - 링크: [doi.org/10.1098/rspa.1985.0070](https://doi.org/10.1098/rspa.1985.0070)
- **Rapid Solution of Problems by Quantum Computation (Deutsch-Jozsa)** (David Deutsch, Richard Jozsa, Proc. R. Soc. Lond. A, 1992) ⭐
  - 주제: 결정론적 양자 우위를 처음 보인 oracle 분리 알고리즘
  - 링크: [doi.org/10.1098/rspa.1992.0167](https://doi.org/10.1098/rspa.1992.0167)
- **Quantum Mechanical Models of Turing Machines That Dissipate No Energy** (Paul Benioff, Phys. Rev. Lett., 1982)
  - 주제: 가역·무소산 양자 계산 모델의 초기 구성
  - 링크: [doi.org/10.1103/PhysRevLett.48.1581](https://doi.org/10.1103/PhysRevLett.48.1581)

## 2. 복잡도 이론 (Quantum Complexity)

- **Quantum Complexity Theory** (Ethan Bernstein, Umesh Vazirani, SIAM J. Comput., 1997) ⭐
  - 주제: BQP 정의, 보편 양자 튜링 기계, Bernstein-Vazirani 문제
  - 링크: [doi.org/10.1137/S0097539796300921](https://doi.org/10.1137/S0097539796300921)
- **Strengths and Weaknesses of Quantum Computing** (Bennett, Bernstein, Brassard, Vazirani, SIAM J. Comput., 1997) ⭐
  - 주제: oracle 하에서 양자 가속의 근본 한계(Grover lower bound)
  - 링크: [doi.org/10.1137/S0097539796300933](https://doi.org/10.1137/S0097539796300933)
- **Quantum Lower Bounds by Polynomials** (Beals, Buhrman, Cleve, Mosca, de Wolf, JACM, 2001) 🆓
  - 주제: 다항식 방법으로 query complexity 하한 증명
  - 링크: [arxiv.org/abs/quant-ph/9802049](https://arxiv.org/abs/quant-ph/9802049) 🆓
- **Quantum Lower Bounds by Quantum Arguments (Adversary Method)** (Andris Ambainis, STOC 2000) 🆓
  - 주제: adversary 논증 기반 query 하한 기법
  - 링크: [arxiv.org/abs/quant-ph/0002066](https://arxiv.org/abs/quant-ph/0002066) 🆓

## 3. 핵심 알고리즘 (Seminal Algorithms)

- **Algorithms for Quantum Computation: Discrete Logarithms and Factoring** (Peter Shor, FOCS 1994) ⭐
  - 주제: 인수분해·이산로그를 다항시간에 푸는 양자 알고리즘 최초 발표
  - 링크: [doi.org/10.1109/SFCS.1994.365700](https://doi.org/10.1109/SFCS.1994.365700)
- **Polynomial-Time Algorithms for Prime Factorization and Discrete Logarithms** (Peter Shor, SIAM J. Comput., 1997) ⭐📖
  - 주제: Shor 알고리즘의 확장 저널판(주기 찾기·QFT 상세)
  - 링크: [arxiv.org/abs/quant-ph/9508027](https://arxiv.org/abs/quant-ph/9508027) 🆓
- **A Fast Quantum Mechanical Algorithm for Database Search (Grover)** (Lov Grover, STOC 1996) ⭐
  - 주제: 비정렬 탐색 √N 가속, amplitude amplification의 원형
  - 링크: [arxiv.org/abs/quant-ph/9605043](https://arxiv.org/abs/quant-ph/9605043) 🆓
- **Quantum Algorithm for Linear Systems of Equations (HHL)** (Harrow, Hassidim, Lloyd, Phys. Rev. Lett., 2009) 🆓
  - 주제: 선형 시스템 Ax=b를 지수적으로 빠르게 푸는 양자 알고리즘
  - 링크: [arxiv.org/abs/0811.3171](https://arxiv.org/abs/0811.3171) 🆓
- **Quantum Amplitude Amplification and Estimation** (Brassard, Høyer, Mosca, Tapp, 2000) 🆓
  - 주제: Grover의 일반화와 양자 카운팅/추정
  - 링크: [arxiv.org/abs/quant-ph/0005055](https://arxiv.org/abs/quant-ph/0005055) 🆓

## 4. 양자 정보 / 텔레포테이션 (Quantum Information)

- **Teleporting an Unknown Quantum State via Dual Classical and EPR Channels** (Bennett, Brassard, Crépeau, Jozsa, Peres, Wootters, Phys. Rev. Lett., 1993) ⭐
  - 주제: 양자 텔레포테이션 프로토콜 제안
  - 링크: [doi.org/10.1103/PhysRevLett.70.1895](https://doi.org/10.1103/PhysRevLett.70.1895)
- **A Single Quantum Cannot Be Cloned (No-Cloning Theorem)** (Wootters, Zurek, Nature, 1982) ⭐
  - 주제: 임의 미지 상태의 복제 불가능성 증명
  - 링크: [doi.org/10.1038/299802a0](https://doi.org/10.1038/299802a0)
- **Communication via One- and Two-Particle Operators on EPR States (Superdense Coding)** (Bennett, Wiesner, Phys. Rev. Lett., 1992)
  - 주제: 얽힘을 이용해 1 qubit로 2 classical bit 전송
  - 링크: [doi.org/10.1103/PhysRevLett.69.2881](https://doi.org/10.1103/PhysRevLett.69.2881)

## 5. 양자 암호 (Quantum Cryptography)

- **Quantum Cryptography: Public Key Distribution and Coin Tossing (BB84)** (Bennett, Brassard, IEEE Conf. 1984) ⭐
  - 주제: BB84 양자 키 분배 프로토콜의 원전
  - 링크: [doi.org/10.1016/j.tcs.2014.05.025](https://doi.org/10.1016/j.tcs.2014.05.025)
- **Quantum Cryptography Based on Bell's Theorem (E91)** (Artur Ekert, Phys. Rev. Lett., 1991) ⭐
  - 주제: 얽힘·Bell 부등식 기반 QKD 제안
  - 링크: [doi.org/10.1103/PhysRevLett.67.661](https://doi.org/10.1103/PhysRevLett.67.661)

## 6. 양자 오류정정 / 결함허용 (QEC & Fault Tolerance)

- **Scheme for Reducing Decoherence in Quantum Computer Memory** (Peter Shor, Phys. Rev. A, 1995) ⭐
  - 주제: 최초의 9-qubit 양자 오류정정 부호
  - 링크: [doi.org/10.1103/PhysRevA.52.R2493](https://doi.org/10.1103/PhysRevA.52.R2493)
- **Error Correcting Codes in Quantum Theory (Steane code)** (Andrew Steane, Phys. Rev. Lett., 1996) ⭐
  - 주제: CSS 계열 7-qubit 부호와 양자 부호 이론 정립
  - 링크: [doi.org/10.1103/PhysRevLett.77.793](https://doi.org/10.1103/PhysRevLett.77.793)
- **Good Quantum Error-Correcting Codes Exist (CSS codes)** (Calderbank, Shor, Phys. Rev. A, 1996) ⭐
  - 주제: 고전 부호로부터 양자 부호를 구성하는 CSS 틀
  - 링크: [arxiv.org/abs/quant-ph/9512032](https://arxiv.org/abs/quant-ph/9512032) 🆓
- **Stabilizer Codes and Quantum Error Correction** (Daniel Gottesman, PhD thesis, 1997) 📖🆓
  - 주제: 스태빌라이저 형식주의로 양자 부호 통합 기술
  - 링크: [arxiv.org/abs/quant-ph/9705052](https://arxiv.org/abs/quant-ph/9705052) 🆓
- **Fault-Tolerant Quantum Computation by Anyons** (Alexei Kitaev, Annals of Physics, 2003) ⭐📖
  - 주제: 토릭 코드·애니온 기반 위상학적 결함허용 계산
  - 링크: [arxiv.org/abs/quant-ph/9707021](https://arxiv.org/abs/quant-ph/9707021) 🆓
- **Topological Quantum Memory (Surface Codes)** (Dennis, Kitaev, Landahl, Preskill, J. Math. Phys., 2002) 🆓
  - 주제: 표면 부호와 임계값 정리의 정량적 분석
  - 링크: [arxiv.org/abs/quant-ph/0110143](https://arxiv.org/abs/quant-ph/0110143) 🆓
- **Surface Codes: Towards Practical Large-Scale Quantum Computation** (Fowler, Mariantoni, Martinis, Cleland, Phys. Rev. A, 2012) 📖🆓
  - 주제: 표면 부호의 실용적 구현·자원 추정 종합
  - 링크: [arxiv.org/abs/1208.0928](https://arxiv.org/abs/1208.0928) 🆓
- **Theory of Fault-Tolerant Quantum Computation** (Daniel Gottesman, Phys. Rev. A, 1998) 🆓
  - 주제: 임계값 정리와 결함허용 게이트 구성
  - 링크: [arxiv.org/abs/quant-ph/9702029](https://arxiv.org/abs/quant-ph/9702029) 🆓

## 7. NISQ / 변분 알고리즘 (NISQ & Variational)

- **Quantum Computing in the NISQ Era and Beyond** (John Preskill, Quantum, 2018) 🆕⭐🆓
  - 주제: NISQ 시대 정의와 근미래 양자 우위 전망
  - 링크: [arxiv.org/abs/1801.00862](https://arxiv.org/abs/1801.00862) 🆓
- **A Variational Eigenvalue Solver on a Photonic Quantum Processor (VQE)** (Peruzzo et al., Nature Communications, 2014) 🆓
  - 주제: 변분 양자 고유값 해법(VQE) 최초 제안·구현
  - 링크: [arxiv.org/abs/1304.3061](https://arxiv.org/abs/1304.3061) 🆓
- **A Quantum Approximate Optimization Algorithm (QAOA)** (Farhi, Goldstone, Gutmann, 2014) 🆓
  - 주제: 조합 최적화용 변분 양자 알고리즘 QAOA
  - 링크: [arxiv.org/abs/1411.4028](https://arxiv.org/abs/1411.4028) 🆓
- **Quantum Supremacy Using a Programmable Superconducting Processor (Sycamore)** (Arute et al. / Google, Nature, 2019) 🆕🏆⭐
  - 주제: 53-qubit Sycamore로 양자 우위 실험적 시연
  - 링크: [nature.com/articles/s41586-019-1666-5](https://www.nature.com/articles/s41586-019-1666-5) 🏆

## 8. 참고서 / 서베이 (Reference & Survey)

- **Quantum Computation and Quantum Information (Nielsen & Chuang)** (Nielsen, Chuang, Cambridge Univ. Press, 2000/2010) 📖⭐
  - 주제: 분야 표준 교재("Mike & Ike")
  - 링크: [doi.org/10.1017/CBO9780511976667](https://doi.org/10.1017/CBO9780511976667) 📖
- **Quantum Algorithms for Algebraic Problems** (Childs, van Dam, Rev. Mod. Phys., 2010) 📖🆓
  - 주제: hidden subgroup 등 대수적 양자 알고리즘 서베이
  - 링크: [arxiv.org/abs/0812.0380](https://arxiv.org/abs/0812.0380) 🆓
- **The Limits of Quantum Computers (BQP and complexity)** (Scott Aaronson, Scientific American / survey, 2008) 🆓
  - 주제: BQP의 능력과 한계에 대한 대중·연구 서베이
  - 링크: [arxiv.org/abs/0808.2669](https://arxiv.org/abs/0808.2669) 🆓
- **Elementary Gates for Quantum Computation** (Barenco et al., Phys. Rev. A, 1995) ⭐🆓
  - 주제: 보편 게이트 집합과 회로 분해의 기초
  - 링크: [arxiv.org/abs/quant-ph/9503016](https://arxiv.org/abs/quant-ph/9503016) 🆓

## 관련
- [[papers]] — Papers hub
- [[theory-algorithms]] — 알고리즘·복잡도 이론 (Shor/Grover/BQP 연계)
- [[security-crypto]] — 양자 암호·post-quantum 보안 연계
- [[foundations-classics]] — Turing/Church 등 계산 기초와 Deutsch 원리
- [[computer-architecture]] — 물리 큐비트·하드웨어(Sycamore) 구현 맥락
