---
title: "troubleshooting · Tool result 가 context 다 먹음"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, tool-use, context, token]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]", "[[context-overflow]]"]
---

# Tool result 가 너무 커서 context 다 먹음

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

- 큰 `grep` / `ls -la /` / `curl` 응답 1회 후 context overflow
- 다음 turn 응답이 매우 느려짐
- Cost 폭증

## 원인

LLM 에 raw tool output 그대로 전달. 50MB JSON / 10000 line grep 등이 context 차지.

## 해결

### Pagination / head + count
```python
def grep_smart(pattern, files):
    matches = grep(pattern, files)
    return {
        "total": len(matches),
        "first_20": matches[:20],
        "hint": f"To see more, call grep_paginated(page=2, page_size=20)" if len(matches) > 20 else None,
    }
```

→ 처음에 전체량 알려주고 일부만. agent 가 필요 시 다음 page.

### Summary 우선
```python
def read_large_file(path):
    size = os.path.getsize(path)
    if size > 100_000:
        # 처음 + 끝 + 요약
        with open(path) as f:
            head = f.read(5000)
            f.seek(-5000, 2)
            tail = f.read(5000)
        return f"File too large ({size} bytes). Showing head/tail:\n[HEAD]\n{head}\n[TAIL]\n{tail}"
    return open(path).read()
```

### Sub-agent 위임
```python
# Main agent — 큰 search 위임
result = await spawn_subagent(
    role="researcher",
    task="search and summarize",
    return_format="bullet points, max 500 tokens",
)
# Main context 에는 summary 만
```

→ Claude Code 의 `Explore` agent 같은 패턴.

### Tool 가 자체적으로 줄임
```python
@tool
def list_files(path: str, max_results: int = 100):
    files = os.listdir(path)
    if len(files) > max_results:
        return {
            "files": files[:max_results],
            "truncated": True,
            "total": len(files),
        }
    return {"files": files}
```

→ tool 책임으로 옮김. agent 가 max_results 늘리려면 명시.

### Compression
```python
import gzip, base64

def compress_for_llm(text):
    compressed = gzip.compress(text.encode())
    return base64.b64encode(compressed).decode()
```

→ LLM 가 압축 데이터 못 읽음. 의미 없음. 그냥 summary 가 옳음.

### 파일 / URL 만 반환 (LLM 가 필요 시 read)
```python
@tool
def search(query):
    matches = full_search(query)
    if len(matches) > 10:
        # 큰 결과 → file 에 저장 + path 반환
        path = save_to_temp(matches)
        return {"saved_to": path, "count": len(matches), "hint": "use read_file(path) for details"}
    return {"matches": matches}
```

## Yule Studio Agent 패턴

```python
# Agent 가 자기 일에 집중. 큰 작업은 sub-agent
agent.spawn(
    subagent="Explore",
    description="codebase 탐색",
    breadth="medium",
    return_under_words=300,
)
```

→ main agent 의 context 에는 300 단어 요약만.

## 진단

```python
# Tool result size 모니터
def log_tool_result(name, result):
    size = len(json.dumps(result))
    if size > 10_000:
        logger.warn("large_tool_result", tool=name, size=size)

# Token usage 추세
response.usage.input_tokens   # 이전 turn 의 tool_result 가 input 으로 들어옴
```

## 재발 방지

- 모든 tool 에 max_results / pagination 표준
- 큰 search 는 sub-agent 위임 patterns
- ESLint / lint 룰: raw `grep -r` / `ls -la /` 등 unbounded 호출 경고
- Agent 룰: "before large tool call, use wc/head to estimate size"

## 관련

- [[ai-agent]] / [[context-overflow]] / [[multi-turn-forgetting]]
