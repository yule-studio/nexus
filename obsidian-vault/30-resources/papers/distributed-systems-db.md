---
title: "분산 시스템 / DB — Papers"
kind: knowledge
status: current
created_at: 2026-05-27T11:45:00+09:00
tags: [resources, papers, distributed-systems, database]
home_hub: resources
---

# 분산 시스템 / DB — Papers

**[[papers|↑ Papers hub]]**

## 1. 합의 / Consensus

- ⭐ **The Part-Time Parliament** (Leslie Lamport, ACM TOCS, 1998) — Paxos 의 시발
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/part-time-parliament/)

- ⭐ **Paxos Made Simple** (Leslie Lamport, 2001)
  - 주제: Paxos 를 처음 보는 사람용 단순화
  - 링크: [공식 PDF](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) 🆓

- ⭐ **In Search of an Understandable Consensus Algorithm** (Diego Ongaro, John Ousterhout, USENIX ATC, 2014) — Raft 의 시발
  - 링크: [raft.github.io/raft.pdf](https://raft.github.io/raft.pdf) 🆓
  - 공식: [raft.github.io](https://raft.github.io/)

- **Viewstamped Replication Revisited** (Liskov, Cowling, 2012)
  - 링크: [MIT](https://pmg.csail.mit.edu/papers/vr-revisited.pdf) 🆓

- **PBFT — Practical Byzantine Fault Tolerance** (Miguel Castro, Barbara Liskov, OSDI, 1999)
  - 링크: [MIT](https://pmg.csail.mit.edu/papers/osdi99.pdf) 🆓

## 2. 분산 시스템 정통

- ⭐ **Time, Clocks, and the Ordering of Events in a Distributed System** (Leslie Lamport, CACM, 1978)
  - 주제: Lamport timestamp 시발
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/time-clocks-ordering-events-distributed-system/) 🆓

- ⭐ **The Byzantine Generals Problem** (Lamport, Shostak, Pease, ACM TOPLAS, 1982)
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/byzantine-generals-problem/) 🆓

- ⭐ **CAP Theorem — Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services** (Gilbert, Lynch, SIGACT, 2002)
  - 링크: [groups.csail.mit.edu/tds](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf) 🆓

- **Harvest, Yield, and Scalable Tolerant Systems** (Eric Brewer 외, 1999)
  - 링크: [berkeley](https://citeseerx.ist.psu.edu/document?doi=10.1.1.24.3690) 🆓

- ⭐ **A Note on Distributed Computing** (Waldo, Wyant, Wollrath, Kendall, Sun Microsystems, 1994)
  - 주제: RPC 의 fallacy
  - 링크: [Sun Labs](https://citeseerx.ist.psu.edu/document?doi=10.1.1.41.7628) 🆓

- **Fallacies of Distributed Computing Explained** (Arnon Rotem-Gal-Oz, 2008)
  - 링크: [PDF](https://www.rgoarchitects.com/Files/fallacies.pdf) 🆓

## 3. 분산 DB (Google 시발)

- ⭐ **The Google File System (GFS)** (Sanjay Ghemawat, Howard Gobioff, Shun-Tak Leung, SOSP, 2003)
  - 링크: [Google Research](https://research.google/pubs/pub51/) 🆓

- ⭐ **MapReduce: Simplified Data Processing on Large Clusters** (Jeffrey Dean, Sanjay Ghemawat, OSDI, 2004)
  - 링크: [Google Research](https://research.google/pubs/pub62/) 🆓

- ⭐ **Bigtable: A Distributed Storage System for Structured Data** (Chang 외, OSDI, 2006)
  - 링크: [Google Research](https://research.google/pubs/pub27898/) 🆓

- ⭐ **Spanner: Google's Globally Distributed Database** (Corbett 외, OSDI, 2012)
  - 주제: TrueTime 으로 strict serializability
  - 링크: [Google Research](https://research.google/pubs/pub39966/) 🆓

- **F1: A Distributed SQL Database That Scales** (Shute 외, VLDB, 2013)
  - 링크: [Google Research](https://research.google/pubs/pub41344/) 🆓

- **Percolator: Large-scale Incremental Processing Using Distributed Transactions and Notifications** (Peng, Dabek, OSDI, 2010)
  - 링크: [Google Research](https://research.google/pubs/pub36726/) 🆓

## 4. 분산 DB (Amazon / Microsoft)

- ⭐ **Dynamo: Amazon's Highly Available Key-value Store** (DeCandia 외, SOSP, 2007)
  - 주제: eventual consistency / vector clock / consistent hashing
  - 링크: [Amazon Science](https://www.amazon.science/publications/dynamo-amazons-highly-available-key-value-store) 🆓

- ⭐ **Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases** (Verbitski 외, SIGMOD, 2017)
  - 링크: [Amazon Science](https://www.amazon.science/publications/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases) 🆓

- **Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service** (Elhemali 외, USENIX ATC, 2022)
  - 링크: [Amazon Science](https://www.amazon.science/publications/amazon-dynamodb-a-scalable-predictably-performant-and-fully-managed-nosql-database-service) 🆓 🆕

- **Azure Cosmos DB** — 다양한 paper / blog — [Microsoft Research](https://www.microsoft.com/en-us/research/) (검색)

## 5. DB internals / Storage

- ⭐ **A Decomposition Storage Model** (Copeland, Khoshafian, SIGMOD, 1985)
  - 주제: column-oriented storage 시발
  - 링크: [ACM](https://dl.acm.org/doi/10.1145/971699.318923)

- ⭐ **The Log-Structured Merge-Tree (LSM-Tree)** (O'Neil 외, Acta Informatica, 1996)
  - 주제: LSM 시발
  - 링크: [공식 PDF](https://www.cs.umb.edu/~poneil/lsmtree.pdf) 🆓

- **Bw-Tree: A B-tree for New Hardware Platforms** (Levandoski 외, ICDE, 2013)
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/the-bw-tree-a-b-tree-for-new-hardware/) 🆓

- ⭐ **C-Store: A Column-oriented DBMS** (Stonebraker 외, VLDB, 2005)
  - 링크: [MIT](https://web.mit.edu/~yandong/www/Documents/papers/cstore_vldb.pdf) 🆓

- **MonetDB/X100: Hyper-Pipelining Query Execution** (Boncz, Zukowski, Nes, CIDR, 2005)
  - 링크: [CWI](https://www.cidrdb.org/cidr2005/papers/P19.pdf) 🆓

- **The Snowflake Elastic Data Warehouse** (Dageville 외, SIGMOD, 2016)
  - 링크: [Snowflake](https://www.snowflake.com/wp-content/uploads/2022/05/3-The-Snowflake-Elastic-DW.pdf) 🆓

## 6. Streaming / Event-based

- ⭐ **Kafka: a Distributed Messaging System for Log Processing** (Kreps, Narkhede, Rao, NetDB, 2011)
  - 링크: [Confluent](https://notes.stephenholiday.com/Kafka.pdf) 🆓

- ⭐ **The Log: What every software engineer should know about real-time data's unifying abstraction** (Jay Kreps, 2013)
  - 주제: log structured 의 정통 essay
  - 링크: [LinkedIn Engineering](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) 🆓

- **The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing** (Akidau 외, VLDB, 2015)
  - 링크: [Google Research](https://research.google/pubs/pub43864/) 🆓

- **Naiad: A Timely Dataflow System** (Murray 외, SOSP, 2013) 🏆
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/naiad-a-timely-dataflow-system/) 🆓

## 7. Transaction / Concurrency

- ⭐ **A Critique of ANSI SQL Isolation Levels** (Hal Berenson 외, SIGMOD, 1995)
  - 주제: isolation level 분석의 정통
  - 링크: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/) 🆓

- **Serializable Snapshot Isolation in PostgreSQL** (Ports, Grittner, VLDB, 2012)
  - 링크: [drkp.net](https://drkp.net/papers/ssi-vldb12.pdf) 🆓

- **Calvin: Fast Distributed Transactions for Partitioned Database Systems** (Thomson 외, SIGMOD, 2012)
  - 링크: [Yale](https://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) 🆓

- **Spanner — TrueTime** → §3

## 8. Caching / Distributed Coordination

- ⭐ **Chubby — The Chubby Lock Service for Loosely-coupled Distributed Systems** (Burrows, OSDI, 2006)
  - 링크: [Google Research](https://research.google/pubs/pub27897/) 🆓

- ⭐ **ZooKeeper: Wait-free coordination for Internet-scale systems** (Hunt, Konar, Junqueira, Reed, USENIX ATC, 2010)
  - 링크: [USENIX](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf) 🆓

- **Memcache at Facebook — Scaling Memcache at Facebook** (Nishtala 외, NSDI, 2013)
  - 링크: [USENIX](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala) 🆓

## 9. Replication / Consistency

- ⭐ **Eventually Consistent — Werner Vogels, ACM Queue, 2008**
  - 링크: [ACM Queue](https://queue.acm.org/detail.cfm?id=1466448) 🆓

- **Highly Available Transactions: Virtues and Limitations** (Bailis 외, VLDB, 2014)
  - 링크: [arXiv](https://arxiv.org/abs/1302.0309) 🆓

- **Conflict-Free Replicated Data Types (CRDTs)** (Shapiro, Preguiça, Baquero, Zawirski, SSS, 2011)
  - 링크: [INRIA HAL](https://inria.hal.science/inria-00609399v1/document) 🆓

## 10. Distributed Algorithms 학술

- **Impossibility of Distributed Consensus with One Faulty Process (FLP)** (Fischer, Lynch, Paterson, JACM, 1985)
  - 링크: [PDF](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) 🆓

- **Linearizability: A Correctness Condition for Concurrent Objects** (Herlihy, Wing, ACM TOPLAS, 1990)
  - 링크: [Brown CS](https://cs.brown.edu/people/mph/HerlihyW90/p463-herlihy.pdf) 🆓

## 11. 한국 관련

- 한국 학자 paper 찾을 때 [Google Scholar](https://scholar.google.com/) 검색

## 12. 관련

- [[papers]] — Papers hub
- [[ai-ml-papers]] — AI/ML 논문
- [[systems-os-network]] — Systems
- [[../books/architecture/architecture]] §1 — DDIA
- [[../books/database/database]] — DB 책
- [[../lectures/mooc-university]] §7 — MIT 6.824 (Lab 에 위 paper 다수 reading)
