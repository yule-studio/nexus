---
title: "Information Retrieval / Web / Data Mining — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, information-retrieval, web, data-mining]
home_hub: resources
---

# Information Retrieval / Web / Data Mining — Papers

**[[papers|↑ Papers hub]]**

> 검색·랭킹·웹 구조 분석·연관 규칙·추천 시스템의 토대를 세운 IR/Web/DM 정전(canon) 논문 모음.

## 1. 검색 랭킹 모델 (Term Weighting & Retrieval Models)

- **A Vector Space Model for Automatic Indexing** (Salton, Wong, Yang; CACM, 1975) ⭐ 🆓
  - 주제: 문서·질의를 벡터로 표현하고 코사인 유사도로 랭킹하는 VSM의 원형
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/361219.361220)
- **A Statistical Interpretation of Term Specificity and Its Application in Retrieval** (Spärck Jones; J. Documentation, 1972) ⭐
  - 주제: 희귀어에 가중치를 주는 IDF의 통계적 기초
  - 링크: [emerald DOI](https://www.emerald.com/insight/content/doi/10.1108/eb026526/full/html)
- **The Probabilistic Relevance Framework: BM25 and Beyond** (Robertson, Zaragoza; FnTIR, 2009) ⭐ 🆓 📖
  - 주제: Okapi BM25의 도출과 확률적 적합성 프레임워크 정리
  - 링크: [staff.city.ac.uk BM25 PDF](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf)
- **A Language Modeling Approach to Information Retrieval** (Ponte, Croft; SIGIR, 1998) ⭐
  - 주제: 문서별 언어 모델로 질의 생성 확률을 추정하는 검색 패러다임
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/290941.291008)

## 2. 잠재 의미·토픽 모델 (Latent Semantics & Topic Models)

- **Indexing by Latent Semantic Analysis** (Deerwester, Dumais, Furnas, Landauer, Harshman; JASIS, 1990) ⭐
  - 주제: SVD로 단어-문서 행렬을 저차원화한 LSI/LSA
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.5555/1461544.1461545)
- **Probabilistic Latent Semantic Indexing** (Hofmann; SIGIR, 1999) ⭐
  - 주제: 잠재 토픽을 확률 혼합으로 모델링한 pLSI
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/312624.312649)
- **Latent Dirichlet Allocation** (Blei, Ng, Jordan; JMLR, 2003) ⭐ 🆓 (cross-ref AI)
  - 주제: 디리클레 사전으로 문서별 토픽 분포를 추정하는 LDA
  - 링크: [jmlr PDF](https://www.jmlr.org/papers/volume3/blei03a/blei03a.pdf)

## 3. 웹 구조·링크 분석 (Web Structure & Link Analysis)

- **The Anatomy of a Large-Scale Hypertextual Web Search Engine** (Brin, Page; WWW7, 1998) ⭐ 🆓 📖
  - 주제: PageRank·anchor text 기반 Google 검색 엔진 프로토타입
  - 링크: [infolab.stanford.edu PDF](http://infolab.stanford.edu/pub/papers/google.pdf)
- **The PageRank Citation Ranking: Bringing Order to the Web** (Page, Brin, Motwani, Winograd; Stanford TR, 1999) ⭐ 🆓
  - 주제: 랜덤 워크로 웹 페이지 중요도를 매기는 PageRank 정식 정의
  - 링크: [stanford ilpubs](http://ilpubs.stanford.edu:8090/422/)
- **Authoritative Sources in a Hyperlinked Environment** (Kleinberg; JACM, 1999) ⭐
  - 주제: hub/authority 상호 강화로 링크 권위를 매기는 HITS
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/324133.324140)
- **Topic-Sensitive PageRank** (Haveliwala; WWW, 2002) ⭐
  - 주제: 주제별 편향 텔레포트로 개인화된 PageRank 계산
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/511446.511513)

## 4. 중복 탐지·해싱 (Near-Duplicate Detection & Hashing)

- **Syntactic Clustering of the Web** (Broder, Glassman, Manasse, Zweig; WWW6, 1997) ⭐
  - 주제: shingling + sketch로 유사 웹 문서를 군집화
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.5555/2842995.2842997)
- **On the Resemblance and Containment of Documents** (Broder; SEQUENCES, 1997) ⭐
  - 주제: MinHash로 자카드 유사도를 추정하는 근사 중복 탐지
  - 링크: [cs.princeton.edu broder PDF](https://www.cs.princeton.edu/courses/archive/spring13/cos598C/broder97resemblance.pdf)
- **Approximate Nearest Neighbors: Towards Removing the Curse of Dimensionality** (Indyk, Motwani; STOC, 1998) ⭐ (cross-ref theory)
  - 주제: 고차원 최근접 이웃을 위한 LSH의 기원
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/276698.276876)
- **Detecting Near-Duplicates for Web Crawling** (Manku, Jain, Das Sarma; WWW, 2007) 🆓
  - 주제: simhash 지문으로 대규모 웹 크롤 중복 제거
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/1242572.1242592)

## 5. 데이터 마이닝·연관 규칙 (Data Mining & Association Rules)

- **Fast Algorithms for Mining Association Rules** (Agrawal, Srikant; VLDB, 1994) ⭐ 🆓
  - 주제: 빈발 항목집합을 단계적으로 가지치기하는 Apriori
  - 링크: [vldb PDF](https://www.vldb.org/conf/1994/P487.PDF)
- **Mining Frequent Patterns without Candidate Generation** (Han, Pei, Yin; SIGMOD, 2000) ⭐
  - 주제: 후보 생성 없이 FP-tree로 빈발 패턴 채굴 (FP-growth)
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/342009.335372)
- **BIRCH: An Efficient Data Clustering Method for Very Large Databases** (Zhang, Ramakrishnan, Livny; SIGMOD, 1996) ⭐
  - 주제: CF-tree로 대규모 데이터를 단일 스캔 군집화
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/233269.233324)

## 6. 협업 필터링·행렬 분해 (Collaborative Filtering & Matrix Factorization)

- **Item-Based Collaborative Filtering Recommendation Algorithms** (Sarwar, Karypis, Konstan, Riedl; WWW, 2001) ⭐ 🆓
  - 주제: 아이템 간 유사도로 확장성 좋은 추천 생성
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/371920.372071)
- **Matrix Factorization Techniques for Recommender Systems** (Koren, Bell, Volinsky; IEEE Computer, 2009) ⭐ 🆓 🏆
  - 주제: Netflix Prize에서 검증된 잠재 인자 행렬 분해 추천
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1109/MC.2009.263)
- **BPR: Bayesian Personalized Ranking from Implicit Feedback** (Rendle, Freudenthaler, Gantner, Schmidt-Thieme; UAI, 2009) ⭐
  - 주제: 암묵 피드백용 쌍별 랭킹 최적화 목적함수
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.5555/1795114.1795167)
- **Factorization Machines** (Rendle; ICDM, 2010) ⭐
  - 주제: 희소 특징 상호작용을 인수분해로 모델링하는 범용 FM
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1109/ICDM.2010.127)

## 7. 학습 기반 랭킹 (Learning to Rank)

- **Learning to Rank using Gradient Descent** (Burges et al.; ICML, 2005) ⭐
  - 주제: 쌍별 확률 손실로 신경 랭커를 학습하는 RankNet
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/1102351.1102363)
- **From RankNet to LambdaRank to LambdaMART: An Overview** (Burges; MSR-TR, 2010) 🆓
  - 주제: NDCG를 직접 겨냥한 LambdaRank/LambdaMART 정리
  - 링크: [microsoft research](https://www.microsoft.com/en-us/research/publication/from-ranknet-to-lambdarank-to-lambdamart-an-overview/)

## 8. 딥러닝 추천·신경 검색 (Neural Recommendation & Retrieval)

- **Deep Neural Networks for YouTube Recommendations** (Covington, Adams, Sargin; RecSys, 2016) ⭐ 🆓
  - 주제: 후보 생성 + 랭킹 2단계 딥러닝 산업 추천 구조
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/2959100.2959190)
- **Wide & Deep Learning for Recommender Systems** (Cheng et al.; DLRS, 2016) ⭐
  - 주제: 암기(wide)와 일반화(deep)를 결합한 추천 모델
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/2988450.2988454)
- **Neural Collaborative Filtering** (He, Liao, Zhang, Nie, Hu, Chua; WWW, 2017) ⭐
  - 주제: 내적 대신 신경망으로 user-item 상호작용 학습
  - 링크: [dl.acm.org DOI](https://dl.acm.org/doi/10.1145/3038912.3052569)
- **Passage Re-ranking with BERT** (Nogueira, Cho; arXiv, 2019) 🆕
  - 주제: BERT 교차 인코더로 1차 검색 결과 재정렬
  - 링크: [arxiv abs/1901.04085](https://arxiv.org/abs/1901.04085)
- **Dense Passage Retrieval for Open-Domain Question Answering** (Karpukhin et al.; EMNLP, 2020) ⭐ 🆕
  - 주제: 듀얼 인코더 밀집 임베딩으로 BM25를 능가한 DPR
  - 링크: [arxiv abs/2004.04906](https://arxiv.org/abs/2004.04906)

## 9. 레퍼런스 교과서 (Reference)

- **Introduction to Information Retrieval** (Manning, Raghavan, Schütze; Cambridge, 2008) 🆓 📖
  - 주제: 색인·랭킹·분류·웹 검색을 망라한 표준 IR 교과서
  - 링크: [nlp.stanford.edu IR-book](https://nlp.stanford.edu/IR-book/)
- **Mining of Massive Datasets** (Leskovec, Rajaraman, Ullman; Cambridge, 3rd ed.) 🆓 📖
  - 주제: MapReduce·LSH·연관 규칙·추천을 다루는 대규모 데이터 마이닝 교재
  - 링크: [mmds.org](http://www.mmds.org/)

## 관련
- [[papers]] — Papers hub
- [[ai-ml-papers]] — LDA·신경 추천·DPR 등 ML 교차 참조
- [[theory-algorithms]] — LSH·근사 최근접 이웃 이론
- [[distributed-systems-db]] — 대규모 색인·MapReduce 인프라
- [[hci-visualization]] — 검색 UX·추천 인터페이스
