---
title: "troubleshooting · Tool call malformed JSON"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, llm, tool-use, json, schema]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]"]
---

# Tool call — `invalid_tool_input`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
Error: invalid_tool_input
  Schema validation failed: required field 'path' missing
  Or: unexpected field 'foo' in object
```

- 또는 silent fail — tool 가 wrong arg 로 실행
- LLM 이 escape 잘못 (`"` vs `\"`)
- nested object / array 의 구조 mismatch

## 원인

LLM 이 tool schema 를 100% 정확히 못 따름. 특히:
- escape (`\n`, `\\`, `\"`) 실수
- enum value 의 case
- nested object 의 required field 누락
- 큰 array 의 마지막 element truncation

## 해결

### Strict schema — JSON Schema 명시
```python
tool = {
    "name": "search",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "minLength": 1, "maxLength": 500},
            "limit": {"type": "integer", "minimum": 1, "maximum": 100, "default": 10},
            "filters": {
                "type": "object",
                "properties": {
                    "type": {"type": "string", "enum": ["doc", "code", "issue"]},
                },
                "additionalProperties": false,    # ← extra field 거절
            },
        },
        "required": ["query"],
        "additionalProperties": false,
    },
}
```

→ LLM 가 schema 보고 정확히 생성하도록 강제.

### Tool description 보강
```python
"description": "검색을 실행합니다.\n- query: 자연어 검색어 (한국어 OK)\n- limit: 최대 결과 수 (1-100, 기본 10)\n- filters.type: 'doc' / 'code' / 'issue' 중 하나",
```

→ 모호한 description → 잘못된 호출 흔함.

### Parser at orchestrator
```python
import jsonschema

def validate_tool_call(tool_name: str, args: dict, schema: dict):
    try:
        jsonschema.validate(args, schema)
    except jsonschema.ValidationError as e:
        # LLM 에 정확한 error 피드백
        return {"error": f"Tool call validation failed: {e.message}. Schema: {schema}"}
```

→ validation error 를 다음 turn 의 message 로 전달 → LLM 가 자체 수정.

### Retry with explicit correction
```python
result = await llm.call_tool(...)
if result.error == "invalid_tool_input":
    # 정확한 schema 와 example 다시 전달
    messages.append({
        "role": "user",
        "content": f"이전 tool call 실패. schema: {schema}\nexample: {{\"query\": \"...\", \"limit\": 10}}",
    })
    result = await llm.call_tool(...)
```

### Structured output (선호)
Claude 의 `tool_use` block 은 자체적으로 JSON. 일반 text completion 보다 정확. 직접 JSON in text 강제 X.

### Tool name versioning
```
search_v2  (new schema)
search_v1  (legacy, backward compat)
```

→ schema breaking change 시 새 tool. 옛 한동안 유지.

## 진단

```python
# Tool input 자체 log
logger.info("tool_call", tool=tool.name, input=tool.input)

# Validation error rate
# anthropic_tool_validation_errors_total / tool_calls_total
```

```
# Claude Code
- /agents — tool list + schema 확인
```

## 재발 방지

- 모든 tool 에 strict JSON Schema (`additionalProperties: false`)
- Tool description 에 example + 제약 명시
- Orchestrator 가 validation + retry with correction
- 새 tool 도입 시 schema test fixture

## 관련

- [[ai-agent]] / [[function-call-schema-drift]] / [[hallucination-loop]]
