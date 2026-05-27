---
title: "도메인 — ai (Claude AI Gateway)"
kind: handover
status: stable
created_at: 2026-05-22T00:06:00+09:00
tags: [bkurs-be, handover, domain, gateway, ai, claude, anthropic]
related:
  - "[[../../bkurs-be]]"
  - "[[../external-gateways]]"
home_hub: "[[../../bkurs-be]]"
---

# ai 도메인

> Claude AI gateway. 자소서 첨삭 / 면접 답변 분석 / 토론 평가 보조 등의 도메인이 호출 예정.
> 사이클 2 PR #103 에서 실 Claude SDK plug-in. 모델 id 는 `claude-sonnet-4-6` / `claude-opus-4-7` 권장.

## 1. 책임

- `AiAssistant.analyze(request) → response` — system + user prompt + max_tokens
- 도메인이 prompt 조립 후 호출 — gateway 는 SDK 호출만 담당
- Mock ↔ Claude SDK swap

## 2. 핵심 타입

| 타입 | 핵심 |
| --- | --- |
| `AiAssistant` (interface) | `analyze(AiAnalysisRequest) → AiAnalysisResponse` |
| `AiAnalysisRequest` (record) | type / systemPrompt / userMessages / maxTokens / temperature |
| `AiAnalysisResponse` (record) | content / model / usage |
| `AiAnalysisType` (enum) | ESSAY_REVIEW / INTERVIEW_ANALYSIS / DISCUSSION_EVAL / 등 |
| `MockAiAssistant` | `@Profile("!prod")` — echo 또는 fixture |
| `ClaudeAiAssistant` | `@Profile("prod")` — Anthropic SDK |
| `AnthropicProperties` | `@ConfigurationProperties("claude.*")` |

## 3. 패키지 구성

```
ai/
└── gateway/
    ├── AiAssistant                  # interface
    ├── AiAnalysisRequest            # record
    ├── AiAnalysisResponse           # record
    ├── AiAnalysisType               # enum
    ├── MockAiAssistant              # fake
    ├── ClaudeAiAssistant            # prod
    └── AnthropicProperties          # config
```

## 4. 호출 패턴 (예정)

```
StatementService.requestAiReview(statementId, teacherId)
  ▼
1. statement = repo.findById(...)
2. AiAnalysisRequest req = AiAnalysisRequest.essayReview(
       systemPrompt = "당신은 학종 자소서 첨삭 전문가입니다 ...",
       userContent = statement.getContent(),
       maxTokens = 2000);
3. CompletableFuture.runAsync(() -> {
       AiAnalysisResponse res = aiAssistant.analyze(req);
       statementRepo.save(statement.attachAiReview(res.content()));
   }, externalCallExecutor);
```

비동기 + `externalCallExecutor` — Claude latency 1~60s.

## 5. 주의사항 (실 SDK 사용 시)

- **모델 id**: `claude-sonnet-4-6` (default) / `claude-opus-4-7` (고품질). retired 모델 회피.
- **prompt caching**: system prompt 가 길고 자주 재사용되면 cache_control 활성. 비용 80%+ 절감.
- **streaming**: 본 backend 는 동기 호출. SSE streaming 은 후속.
- **rate limit**: tier 별 RPM/TPM 제한. saturation 시 Resilience4j retry + backoff.

## 6. 의존

- **의존**: common.config.AsyncConfig
- **의존받음**: (후속) statement, interview, discussion, simulation 의 보조 분석

## cross-link

- [[../external-gateways#ai]]
- 외부: [Anthropic API docs](https://docs.anthropic.com)
