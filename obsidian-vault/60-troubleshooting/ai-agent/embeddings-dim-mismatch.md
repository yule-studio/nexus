---
title: "troubleshooting · Embeddings dim mismatch"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, embeddings, vector-db, rag]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]"]
---

# Embeddings — dim 1536 ≠ 768

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
vector dimension mismatch: expected 768, got 1536
ValueError: index dimensions (768) do not match query dimensions (1536)
```

- 새 embedding model 로 query 인데 index 는 옛 model
- 검색 결과 무관한 것만 나옴

## 원인

각 embedding model 은 고유 dim:
- `text-embedding-ada-002` (OpenAI legacy) — 1536
- `text-embedding-3-small` — 1536 (또는 reduce)
- `text-embedding-3-large` — 3072
- `voyage-3` — 1024
- `cohere-embed-v3` — 1024
- BERT / Sentence-Transformers — 768

다른 model 의 vector 는 비교 불가 (space 자체 다름).

## 해결

### 전체 reindex
```python
# 새 model 로 모든 document re-embed
for doc in all_docs:
    vector = new_model.embed(doc.text)
    new_index.add(doc.id, vector)

# 옛 index 삭제
old_index.delete_all()
```

→ 가장 깔끔. 시간 / 비용 큼.

### 두 model 병행 운영
```python
class HybridIndex:
    def __init__(self):
        self.old_idx = PineconeIndex("docs-ada", dim=1536)
        self.new_idx = PineconeIndex("docs-3-small", dim=1536)

    def search(self, query):
        new_results = self.new_idx.search(new_model.embed(query))
        if len(new_results) < 10:
            old_results = self.old_idx.search(old_model.embed(query))
            return merge(new_results, old_results)
        return new_results
```

→ 점진적 migration. 새 ingestion 은 new, query 는 양쪽.

### Model 메타 저장 (index 에)
```python
# Vector DB metadata 에 model name 포함
doc_payload = {
    "id": doc.id,
    "vector": embed_v1(doc.text),
    "metadata": {
        "embedding_model": "text-embedding-3-small",
        "embedding_dim": 1536,
        "embedded_at": "2026-05-27",
    },
}
```

→ debugging 시 어느 model 의 vector 인지 추적 가능.

### Dim reduction (`text-embedding-3-*` 만)
```python
# OpenAI text-embedding-3-* 는 Matryoshka — dim 줄여도 의미 보존
response = openai.embeddings.create(
    input="text",
    model="text-embedding-3-small",
    dimensions=512,   # 1536 → 512
)
```

→ index 작게 만들 때 옵션. 단 다른 dim 의 query 와 호환 X.

### Normalize
```python
import numpy as np

vec = vec / np.linalg.norm(vec)   # L2 normalize
# Cosine similarity 가 dot product 와 동치 → 빠름
```

dim 문제 아니지만 일관성 위해.

## RAG retrieval 품질 평가

```python
# Fixture 50 query → expected top-1 doc
queries = [
    {"q": "JPA N+1 해결", "expected_doc_id": "n-plus-one-query.md"},
    ...
]

# Recall@5 / MRR 계산
results = [index.search(embed(q["q"]), k=5) for q in queries]
recall_at_5 = sum(q["expected_doc_id"] in [r.id for r in res] for q, res in zip(queries, results)) / len(queries)
```

→ embedding model 교체 시 회귀 평가. [[../../../docs/memory.md]] 의 retrieval eval 참조.

## 진단

```python
# Vector dim 확인
vector = embed(text)
print(vector.shape)   # (1536,) 등

# Index 의 dim
print(index.config.dimension)

# Mismatch 시 사전 검증
assert vector.shape[0] == index.config.dimension, f"dim mismatch: {vector.shape[0]} vs {index.config.dimension}"
```

## 재발 방지

- Index 에 model 메타 저장
- Embedding model 변경 시 reindex 계획 + retrieval eval 회귀
- CI: embedding API call → dim 검증 / cost / latency

## 관련

- [[ai-agent]] / [[context-overflow]] / [[function-call-schema-drift]]
