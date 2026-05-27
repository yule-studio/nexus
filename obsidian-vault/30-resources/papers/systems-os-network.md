---
title: "Systems / OS / Network — Papers"
kind: knowledge
status: current
created_at: 2026-05-27T11:55:00+09:00
tags: [resources, papers, systems, os, network]
home_hub: resources
---

# Systems / OS / Network — Papers

**[[papers|↑ Papers hub]]**

## 1. OS 古典

- ⭐ **The UNIX Time-Sharing System** (Dennis Ritchie, Ken Thompson, CACM, 1974) 🏆
  - 링크: [Bell Labs](https://dsf.berkeley.edu/cs262/unix.pdf) 🆓

- **End-to-End Arguments in System Design** (Saltzer, Reed, Clark, ACM TOCS, 1984)
  - 링크: [MIT](https://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf) 🆓

- **The Linux Operating System** (Linus Torvalds 외, 1990s 초창기 USENET / paper) — [linux.org](https://www.linux.org/)

- **Hints for Computer System Design** (Butler Lampson, OSDI / ACM, 1983)
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/hints-for-computer-system-design-2/) 🆓

## 2. Container / Virtualization

- ⭐ **Borg, Omega, and Kubernetes** (Brendan Burns 외, ACM Queue, 2016)
  - 주제: Google 의 orchestrator 진화
  - 링크: [Google Research](https://research.google/pubs/borg-omega-and-kubernetes/) 🆓

- ⭐ **Large-scale cluster management at Google with Borg** (Verma 외, EuroSys, 2015)
  - 링크: [Google Research](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/) 🆓

- **Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center** (Hindman 외, NSDI, 2011)
  - 링크: [USENIX](https://www.usenix.org/legacy/event/nsdi11/tech/full_papers/Hindman_new.pdf) 🆓

- **Firecracker: Lightweight Virtualization for Serverless Applications** (Agache 외, NSDI, 2020)
  - 링크: [USENIX](https://www.usenix.org/conference/nsdi20/presentation/agache) 🆓

- **gVisor — The Design of a Practical System for Fault-Tolerant Virtual Machines** (Bressoud, Schneider, SOSP, 1995)
  - 링크: [ACM](https://dl.acm.org/doi/10.1145/224057.224058)

## 3. Storage / FS

- ⭐ **A Fast File System for UNIX (FFS)** (McKusick 외, ACM TOCS, 1984)
  - 링크: [Berkeley](https://www.cs.princeton.edu/courses/archive/fall03/cs518/papers/ffs.pdf) 🆓

- **The Design and Implementation of a Log-Structured File System (LFS)** (Rosenblum, Ousterhout, ACM TOCS, 1992)
  - 링크: [Stanford](https://people.eecs.berkeley.edu/~brewer/cs262/LFS.pdf) 🆓

- **ZFS — The Z File System** (Bonwick, Moore, Sun, 2007)
  - 링크: [Sun Microsystems whitepaper](https://www.osnews.com/story/12471/zfs-the-last-word-in-file-systems-part-2/) 🆓

- **Ceph: A Scalable, High-Performance Distributed File System** (Weil 외, OSDI, 2006)
  - 링크: [USENIX](https://www.usenix.org/legacy/event/osdi06/tech/full_papers/weil/weil.pdf) 🆓

- **Haystack — Finding a needle in Haystack: Facebook's photo storage** (Beaver 외, OSDI, 2010)
  - 링크: [USENIX](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf) 🆓

## 4. Concurrency / Threading

- ⭐ **Communicating Sequential Processes (CSP)** (C.A.R. Hoare, CACM, 1978)
  - 링크: [공식 PDF](http://www.usingcsp.com/cspbook.pdf) 🆓

- **A History of Modern 64-bit Computing** — 다양
- **The C10K problem** (Dan Kegel, 1999) — 1만 connection 처리
  - 링크: [dankegel.com/c10k.html](http://www.kegel.com/c10k.html) 🆓

- **Scalable Concurrent Hash Tables via Relativistic Programming** (Triplett, McKenney, Walpole, USENIX ATC, 2010)
  - 링크: [USENIX](https://www.usenix.org/legacy/event/usenix10/tech/full_papers/Triplett.pdf) 🆓

## 5. Network / Protocol

- ⭐ **A Protocol for Packet Network Intercommunication (TCP/IP 시발)** (Cerf, Kahn, IEEE TOC, 1974)
  - 링크: [IETF](https://datatracker.ietf.org/doc/html/rfc7042) (관련) / 검색

- **Congestion Avoidance and Control** (Jacobson, Karels, SIGCOMM, 1988)
  - 주제: TCP congestion control 시발
  - 링크: [공식](https://ee.lbl.gov/papers/congavoid.pdf) 🆓

- **BBR: Congestion-Based Congestion Control** (Cardwell 외, ACM Queue, 2016)
  - 링크: [Google Research](https://research.google/pubs/pub45646/) 🆓

- ⭐ **The QUIC Transport Protocol — RFC 9000** → [[../official-docs/standards-rfc]] §1

## 6. Distributed Computing — General

→ [[distributed-systems-db]] §1-2

## 7. 캐시 / Memory

- **Caching in the Sprite Network File System** (Nelson, Welch, Ousterhout, ACM TOCS, 1988)
  - 링크: [Stanford](https://people.eecs.berkeley.edu/~kubitron/courses/cs262a-F11/papers/sprite-cache.pdf) 🆓

- ⭐ **What Every Programmer Should Know About Memory** (Ulrich Drepper, Red Hat, 2007)
  - 주제: CPU cache / NUMA / 메모리 hierarchy
  - 링크: [PDF](https://www.akkadia.org/drepper/cpumemory.pdf) 🆓 📖

- **MICA: A Holistic Approach to Fast In-Memory Key-Value Storage** (Lim 외, NSDI, 2014)
  - 링크: [USENIX](https://www.usenix.org/conference/nsdi14/technical-sessions/presentation/lim) 🆓

## 8. eBPF / 시스템 관측

- **The BSD Packet Filter: A New Architecture for User-level Packet Capture** (McCanne, Jacobson, USENIX, 1993)
  - 주제: BPF 시발
  - 링크: [공식](https://www.tcpdump.org/papers/bpf-usenix93.pdf) 🆓

- **eBPF — A Brief History of eBPF and How Linux Networking has Evolved** — [ebpf.io](https://ebpf.io/) (paper 모음)

## 9. Hardware / Performance

- **Latency Numbers Every Programmer Should Know** (Jeff Dean, 2010+) — 표 만 (paper 아님)
  - 링크: [GitHub gist](https://gist.github.com/jboner/2841832) 🆓

- ⭐ **Mechanical Sympathy** (Martin Thompson, blog) — 古典 essay 모음
  - 링크: [mechanical-sympathy.blogspot.com](http://mechanical-sympathy.blogspot.com/) 🆓

- **The LMAX Architecture (Disruptor)** (Martin Fowler, 2011)
  - 링크: [martinfowler.com](https://martinfowler.com/articles/lmax.html) 🆓

## 10. Cloud / Datacenter

- **Tail at Scale** (Jeff Dean, Luiz André Barroso, CACM, 2013)
  - 주제: latency 의 long tail
  - 링크: [Google Research](https://research.google/pubs/the-tail-at-scale/) 🆓

- **Resource Central: Understanding and Predicting Workloads for Improved Resource Management** (Cortez 외, SOSP, 2017)
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/resource-central-understanding-predicting-workloads-improved-resource-management-large-cloud-platforms/) 🆓

- **AWS S3 — Building and operating a pretty big storage system** (Andy Warfield, AWS, ACM Queue, 2023) 🆕
  - 링크: [ACM Queue](https://queue.acm.org/detail.cfm?id=3636635) 🆓

## 11. 학회 / Archive

- ⭐ **SOSP** (Symposium on Operating Systems Principles) — [공식](https://sigops.org/s/conferences/sosp/)
- ⭐ **OSDI** (USENIX) — [공식](https://www.usenix.org/conferences/byname/179)
- **NSDI** (USENIX) — [공식](https://www.usenix.org/conferences/byname/121)
- **EuroSys** — [공식](https://www.eurosys.org/)
- **SIGCOMM** (Networking) — [공식](https://www.sigcomm.org/)
- **SIGMETRICS** (Performance) — [공식](https://www.sigmetrics.org/)
- **USENIX ATC** — [공식](https://www.usenix.org/conferences/byname/108)

## 12. 관련

- [[papers]] — Papers hub
- [[distributed-systems-db]] — 분산 / DB
- [[venues-archives]] — 학회 / archive 종합
- [[../books/computer-science/computer-science]] §3-5 — OS / Architecture / Network 책
- [[../books/devops/devops]] §10 — Linux internal
