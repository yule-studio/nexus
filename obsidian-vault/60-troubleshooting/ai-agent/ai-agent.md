---
title: "60-troubleshooting/ai-agent — AI Agent 트러블 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-27
tags: [troubleshooting, ai-agent, llm, claude, tool-call, prompt, hub]
home_hub: "60-troubleshooting"
---

# AI Agent 트러블 카탈로그

> LLM / Claude Code / Tool call / Prompt engineering 에서 자주 만나는 대표 트러블.

**[[../troubleshooting|↑ 60-troubleshooting]]** · **[[../../20-areas/ai-agent/ai-agent|↑↑ 20-areas/ai-agent]]**

---

## 카탈로그

| #  | 증상 / 에러                                                                  | 한 줄 원인                              | 핵심 해결                                                       | 깊이 노트                                |
| -- | ------------------------------------------------------------------------ | ---------------------------------- | ----------------------------------------------------------- | ------------------------------------ |
| 1  | **Context overflow**                                                     | 누적 history / tool result token 초과 | summarization / sub-agent / cache_control / sliding window  | [[context-overflow]]                 |
| 2  | **Tool call malformed JSON**                                             | LLM escape / schema mismatch        | strict JSON Schema / additionalProperties:false / retry      | [[tool-malformed-json]]              |
| 3  | **Hallucination — 존재 안 하는 file/function**                                | LLM train data 기반 추측             | ls/grep verify 룰 / RAG / linting auto-run                  | [[hallucination-loop]]               |
| 4  | **Prompt injection**                                                     | user/external content 가 instruction | system/user 분리 / tool 권한 minimize / output validation     | [[prompt-injection]]                 |
| 5  | **Rate limit 429**                                                       | 동시 호출 / 토큰 burst                | exponential backoff / semaphore / model fallback / caching | [[rate-limit-429]]                   |
| 6  | **Embeddings dim mismatch**                                              | 새 model 로 reindex 없이 query       | reindex / 두 model 병행 / index 에 model 메타                | [[embeddings-dim-mismatch]]          |
| 7  | **Streaming truncation**                                                 | max_tokens / network / proxy        | max_tokens 충분히 / stop_reason 검증 + continuation         | [[streaming-truncation]]             |
| 8  | **Function call schema drift**                                           | tool schema 변경 → 옛 호출 거절        | name versioning / backward-compat field 추가                | [[function-call-schema-drift]]       |
| 9  | **Agent infinite loop**                                                  | termination 부재 / hallucinated success | max_iterations / progress detection / explicit signal    | [[agent-infinite-loop]]              |
| 10 | **Cache key collision**                                                  | hash input 불완전                   | full context (system+user+tools+model) / user namespace    | [[cache-key-collision]]              |
| 11 | **Claude Code sandbox block**                                            | auto-mode classifier 가 의심         | scope 분리 / explicit prompt / approval card               | [[../network/claude-sandbox-block]]  |
| 12 | **Tool result 가 너무 큼**                                                 | 큰 grep / ls / curl response       | pagination / head+count / sub-agent / save_to_temp         | [[tool-result-too-large]]            |
| 13 | **Multi-turn 망각**                                                      | summarization 이 핵심 결정 drop      | sticky decision log / TaskList / MEMORY.md / explicit recap | [[multi-turn-forgetting]]            |
| 14 | **Anthropic overloaded_error 529**                                       | 모델 capacity 일시 부족              | backoff / model fallback / Bedrock/Vertex / status webhook | [[anthropic-overloaded-error]]       |
| 15 | **Tool 가 raw text 반환**                                                  | response 가 free-form text          | structured JSON / response schema validator / MCP          | [[tool-returns-raw-text]]            |

## Yule Studio Agent 컨텍스트 특정 사례

- [[troubleshooting-naver-search-clone-autonomous-intake-sandbox-block]] — sandbox block 사례
- [[troubleshooting-naver-search-clone-canonical-session-end-to-end]] — canonical session 복구

## 자주 쓰는 진단

```text
# Claude Code
- /status — 현재 mode / context size
- /memory — auto memory 검사
- /tasks — 진행 상태
- /agents — sub-agent 사용 가능 목록
- /cost — token 누적

# Anthropic API
- response headers: anthropic-ratelimit-* / x-request-id
- usage / cache_read / cache_creation token
```

## 관련

- [[../troubleshooting]]
- [[../../20-areas/ai-agent/ai-agent|20-areas/ai-agent]]
- [[../network/network|network 트러블]] — sandbox / API rate limit 경계
- yule-studio-agent: `hookify` mistake_ledger / `agents/lifecycle/troubleshooting_ledger`
