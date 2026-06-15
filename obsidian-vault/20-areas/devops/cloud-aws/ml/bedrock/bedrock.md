---
title: "Amazon Bedrock"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:50:00+09:00
tags:
  - aws
  - ml
  - ai
  - llm
  - bedrock
---

# Amazon Bedrock

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Bedrock hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**파운데이션 모델 (Claude / Llama / Titan / Mistral / Cohere 등) 을 AWS API 한 번으로** 호출. IAM 권한 + VPC 안 호출 + 토큰 단위 과금.

---

## 2. 왜 / 언제

- LLM 응용 (챗봇 / RAG / 요약 / 분류 / 코드 생성)
- 모델 vendor 별 API 키 관리 안 하고 싶을 때
- 응답이 AWS 안에 머물러야 할 때 (보안 / 컴플라이언스)
- IAM 으로 권한 + CloudTrail 로 호출 감사
- 여러 모델을 한 인터페이스로 (vendor-lock 회피)

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Model** | Claude / Llama 3 / Titan / Mistral / Cohere / Stability ... |
| **Model access** | 콘솔에서 "Request access" 한 번 (region 별) |
| **Foundation model API** | `InvokeModel` (sync) / `InvokeModelWithResponseStream` (streaming) |
| **Converse API** | 모델 공통 인터페이스 (tool use / multi-turn 표준화) |
| **Knowledge Base** | 매니지드 RAG (S3 → embedding → OpenSearch → 검색) |
| **Agents** | tool use + planning 자동화 |
| **Guardrails** | 입출력 필터 (욕설 / PII / 주제 제한) |
| **Provisioned Throughput** | 모델 용량 예약 (대량 호출 보장 + 할인) |
| **Cross-region inference** | 여러 region 자동 분산 |

---

## 4. 최소 호출 예시 (Claude via Converse API)

```python
import boto3, json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

resp = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-5-20250929-v1:0",
    messages=[
        {"role": "user", "content": [{"text": "S3 와 EFS 차이를 한 줄로."}]}
    ],
    inferenceConfig={"maxTokens": 200, "temperature": 0.0},
)
print(resp["output"]["message"]["content"][0]["text"])
```

→ Anthropic SDK 의 `anthropic` 클라이언트도 Bedrock backend 지원 (`AnthropicBedrock`).

---

## 5. 표준 응용 — RAG 백엔드

```
사용자 질문
   ↓ API Gateway → Lambda
   ↓
1. Embedding (Bedrock Titan / Cohere)
2. Vector search (OpenSearch Serverless / pgvector / Knowledge Base)
3. context + 질문 → Claude (Bedrock)
4. 응답 stream
   ↓
DynamoDB (대화 이력)
```

→ **Knowledge Base** 가 1~2 단계를 매니지드로 해 준다 (S3 만 가리키면 끝).

---

## 6. 모델 선택 — 2026 시점

| 모델 | 강점 | 언제 |
| --- | --- | --- |
| Claude Sonnet 4.6 / Opus 4.7 (1M) | 강력한 추론 / 긴 context | **일반 응용 기본** |
| Claude Haiku 4.5 | 저렴 / 빠름 | 분류 / 단순 응용 / 대량 |
| Llama 3.x | 오픈 / 커스텀 | self-host 가능, fine-tuning |
| Titan Embeddings | 임베딩 전용 | vector search |
| Stability (SDXL) | 이미지 | 이미지 생성 |
| Cohere Command | 다국어 / RAG | 검색 친화 |

→ 응용 디폴트: **Claude (Sonnet / Haiku)**. 임베딩: **Titan / Cohere**.

자세히 → [[../../../ai-agent/ai-agent|↗ AI Agent 영역]]

---

## 7. 함정 / 비용

- **토큰 단위 과금** — input + output 따로. Claude Sonnet ~ $3/M in + $15/M out (대략).
- **모델 access 승인** — 콘솔에서 region 별 model 별 "Request access" 한 번. 안 하면 403.
- **region 제한** — 모델마다 지원 region 다름. `us-east-1` / `us-west-2` 가 가장 폭넓음.
- **prompt 캐싱** — 큰 system prompt 는 cache 활용 (지원 모델 / 5 분 TTL).
- **rate limit** — quota 초과 시 throttle. **Provisioned Throughput** 으로 보장.
- **Guardrails** — 응용 부담 ↓, 다만 false positive 주의.
- **VPC 안 호출** — VPC Endpoint (`com.amazonaws.{region}.bedrock-runtime`) 로 인터넷 우회.

---

## 8. 관련

- [[../ml|↑ ML & AI]]
- [[../../../ai-agent/ai-agent|↗ AI Agent 영역]]
- [[../../../ai/llm-prompting|↗ LLM prompting]]
- [[../../compute/lambda/lambda]] — 응용 백엔드
- [[../../storage/s3/s3]] — Knowledge Base 의 source
