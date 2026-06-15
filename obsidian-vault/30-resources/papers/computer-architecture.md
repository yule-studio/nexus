---
title: "Computer Architecture / Hardware — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, architecture, hardware, hpc]
home_hub: resources
---

# Computer Architecture / Hardware — Papers

**[[papers|↑ Papers hub]]**

> 컴퓨터 구조의 토대 — ISA 설계, ILP, 메모리 일관성, 성능 모델, 가속기, 하드웨어 보안까지 seminal 논문 모음.

## 1. 성능 법칙 / 모델

- **Validity of the Single Processor Approach to Achieving Large Scale Computing Capabilities** (Gene Amdahl, AFIPS, 1967) ⭐
  - 주제: 병렬화의 이론적 상한 (Amdahl's Law)
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/1465482.1465560)
- **Reevaluating Amdahl's Law** (John Gustafson, CACM, 1988)
  - 주제: 문제 크기 확장 기반 scaled speedup 반론
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/42411.42415)
- **Roofline: An Insightful Visual Performance Model for Multicore Architectures** (Williams, Waterman, Patterson, CACM, 2009) ⭐
  - 주제: 연산 강도 vs 메모리 대역폭 시각적 성능 모델
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/1498765.1498785)

## 2. ISA / RISC / RISC-V

- **The Case for the Reduced Instruction Set Computer** (Patterson, Ditzel, SIGARCH, 1980) ⭐
  - 주제: RISC 철학의 원전
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/641914.641917)
- **Computer Architecture: A Quantitative Approach** (Hennessy, Patterson, 교과서 ref)
  - 주제: 정량적 컴퓨터 구조 표준 교재
  - 링크: [[../books/computer-science/computer-science|CS books hub]]
- **The RISC-V Instruction Set Manual, Volume I: Unprivileged ISA** (Waterman, Asanović 외)
  - 주제: 개방형 ISA 공식 명세 🆓
  - 링크: [riscv.org/technical/specifications](https://riscv.org/technical/specifications/)
- **The Case for Open Instruction Sets** (Asanović, Patterson, 2014) 🆓
  - 주제: 개방형 ISA 당위성 (RISC-V 동기)
  - 링크: [people.eecs.berkeley.edu](https://people.eecs.berkeley.edu/~krste/papers/CaseForOpenISA-microreport.pdf)

## 3. ILP — 파이프라인 / 비순차 실행 / 분기 예측

- **An Efficient Algorithm for Exploiting Multiple Arithmetic Units** (Robert Tomasulo, IBM J. R&D, 1967) ⭐
  - 주제: 동적 스케줄링 / 레지스터 리네이밍의 원조 (Tomasulo 알고리즘)
  - 링크: [ieeexplore.ieee.org](https://ieeexplore.ieee.org/document/5392028)
- **A Study of Branch Prediction Strategies** (James E. Smith, ISCA, 1981) ⭐
  - 주제: 2-bit saturating counter 등 기본 분기 예측 전략
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.5555/800052.801871)
- **Two-Level Adaptive Training Branch Prediction** (Yeh, Patt, MICRO, 1991)
  - 주제: 히스토리 기반 적응형 2-레벨 예측기
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/123465.123475)

## 4. 메모리 일관성 / 캐시 코히어런스

- **How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs** (Leslie Lamport, IEEE TC, 1979) ⭐
  - 주제: 순차 일관성 (sequential consistency) 정의
  - 링크: [ieeexplore.ieee.org](https://ieeexplore.ieee.org/document/1675439)
- **A New Solution to Coherence Problems in Multicache Systems** (Censier, Feautrier, IEEE TC, 1978) ⭐
  - 주제: 디렉터리 기반 캐시 코히어런스 원전
  - 링크: [ieeexplore.ieee.org](https://ieeexplore.ieee.org/document/1675057)
- **Using Cache Memory to Reduce Processor-Memory Traffic** (James Goodman, ISCA, 1983)
  - 주제: snoopy write-once 프로토콜 (MESI 선구)
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/800046.801647)
- **Shared Memory Consistency Models: A Tutorial** (Adve, Gharachorloo, IEEE Computer, 1996) ⭐
  - 주제: relaxed 일관성 모델 정리 튜토리얼 🆓
  - 링크: [cs.utexas.edu (PDF)](https://www.cs.utexas.edu/~bornholt/papers/memory-tutorial.pdf)

## 5. 병렬 / HPC

- **The Landscape of Parallel Computing Research: A View from Berkeley** (Asanović 외, UCB Tech Report, 2006) ⭐
  - 주제: 병렬 컴퓨팅 13 dwarfs 분류 (Berkeley View) 🆓
  - 링크: [www2.eecs.berkeley.edu](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2006/EECS-2006-183.html)
- **Why Systolic Architectures?** (H. T. Kung, IEEE Computer, 1982) ⭐
  - 주제: 시스톨릭 어레이 — TPU 등 가속기의 뿌리
  - 링크: [ieeexplore.ieee.org](https://ieeexplore.ieee.org/document/1653825)
- **The Datacenter as a Computer** (Barroso, Hölzle, Synthesis Lectures)
  - 주제: warehouse-scale 컴퓨팅 설계 원리
  - 링크: [google research](https://research.google/pubs/the-datacenter-as-a-computer-an-introduction-to-the-design-of-warehouse-scale-machines/)

## 6. GPU / 도메인 특화 가속기

- **In-Datacenter Performance Analysis of a Tensor Processing Unit** (Jouppi 외, ISCA, 2017) 🏆🆓
  - 주제: 구글 TPU v1 추론 가속기 정량 분석
  - 링크: [arxiv.org/abs/1704.04760](https://arxiv.org/abs/1704.04760) · [dl.acm.org](https://dl.acm.org/doi/10.1145/3079856.3080246)
- **NVIDIA Tesla: A Unified Graphics and Computing Architecture** (Lindholm 외, IEEE Micro, 2008) ⭐
  - 주제: 통합 셰이더 / CUDA 가능 GPU 구조
  - 링크: [ieeexplore.ieee.org](https://ieeexplore.ieee.org/document/4523358)
- **Scalable Parallel Programming with CUDA** (Nickolls, Buck, Garland, Skadron, ACM Queue, 2008)
  - 주제: CUDA 프로그래밍 모델 소개
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/1365490.1365500)

## 7. 기술 한계 / 미래 전망

- **Dark Silicon and the End of Multicore Scaling** (Esmaeilzadeh 외, ISCA, 2011) 🏆
  - 주제: Dennard 스케일링 종료와 dark silicon 한계
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/2000064.2000108)
- **The Future of Microprocessors** (Borkar, Chien, CACM, 2011)
  - 주제: 에너지 한계 시대 멀티코어 진화 전망
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/1941487.1941507)
- **A New Golden Age for Computer Architecture** (Hennessy, Patterson Turing Lecture, CACM, 2019) ⭐🏆🆕
  - 주제: 도메인 특화 구조 / 개방 ISA / 보안의 새 황금기
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/3282307)

## 8. 하드웨어 보안 / 신뢰성

- **Flipping Bits in Memory Without Accessing Them (RowHammer)** (Kim 외, ISCA, 2014) ⭐🏆
  - 주제: DRAM disturbance 비트 플립 결함
  - 링크: [ieeexplore.ieee.org](https://ieeexplore.ieee.org/document/6853210)
- **Spectre Attacks: Exploiting Speculative Execution** (Kocher 외, IEEE S&P, 2019) ⭐🏆
  - 주제: 추측 실행 사이드채널 공격
  - 링크: [arxiv.org/abs/1801.01203](https://arxiv.org/abs/1801.01203)
- **Meltdown: Reading Kernel Memory from User Space** (Lipp 외, USENIX Security, 2018) ⭐🏆
  - 주제: 비순차 실행으로 커널 메모리 누출
  - 링크: [arxiv.org/abs/1801.01207](https://arxiv.org/abs/1801.01207)

## 9. 인터커넥트 / 메모리 시스템

- **A Case for Redundant Arrays of Inexpensive Disks (RAID)** (Patterson, Gibson, Katz, SIGMOD, 1988) ⭐
  - 주제: RAID 스토리지 신뢰성 / 성능 분류
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/50202.50214)
- **The Gem5 Simulator** (Binkert 외, SIGARCH CAN, 2011)
  - 주제: 사이클 정확 구조 시뮬레이터 표준 도구 🆓
  - 링크: [gem5.org](https://www.gem5.org/) · [arxiv.org/abs/2007.03152](https://arxiv.org/abs/2007.03152)
- **The Tail at Scale** (Dean, Barroso, CACM, 2013) ⭐
  - 주제: 대규모 시스템 tail latency 관리
  - 링크: [dl.acm.org](https://dl.acm.org/doi/10.1145/2408776.2408794)

## 관련
- [[papers]] — Papers hub
- [[systems-os-network]] — OS / 분산 / 네트워크 논문
- [[security-crypto]] — Spectre/Meltdown/RowHammer 보안 측면
- [[ai-ml-papers]] — TPU / GPU 가속기와 ML 연결
- [[../books/computer-science/computer-science]] — Hennessy-Patterson 등 교재
