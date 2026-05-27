---
title: "troubleshooting · Hallucination loop — 존재하지 않는 file/function"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, llm, claude, hallucination, verification]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]"]
---

# Hallucination — LLM 이 존재 안 하는 file / function 호출

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

- LLM 이 `src/utils/legacy.py` 같은 file 추천했는데 실제로 없음
- "이 함수는 X 라이브러리에서 import" 했는데 그 함수 deprecated 또는 없음
- 코드 변경 후 import error / undefined

## 원인

LLM 의 training data 기준 추론. 다음에서 hallucination 흔함:
- 새 library 버전 (cutoff 이후)
- 회사 internal API
- 옛 deprecated API (옛 train data 가 우세)
- 비슷한 이름의 다른 library 혼동

## 해결

### Verify before recommend (룰)
```
Agent 룰:
1. file path 추천 전 ls / find 확인
2. function 사용 전 grep / API doc 확인
3. import 사용 전 package.json / requirements.txt 확인
```

Claude Code system prompt 에 포함:
```
"Before recommending a file path, verify it exists.
Before recommending a function, grep for its definition."
```

### Tool 이 먼저
```python
# Bad — LLM 이 직접 코드 작성
"Use the helper function in src/utils/legacy.py"

# Good — tool 먼저
1. Bash: ls src/utils/
2. (없으면) grep -r "def helper"
3. 발견 시 그 path 사용
```

### Citation 요구
```python
prompt = """
파일 / 함수를 언급할 때 항상 source 명시:
- file:line 형식 (file.py:42)
- 또는 "I'll check first with ls" 라고 verify 의도 표명
"""
```

### Linting / static check 자동
```bash
# 코드 변경 후 자동
ruff check
mypy src/
tsc --noEmit
```

→ 잘못된 import / undefined 즉시 catch. LLM 에 error 피드백 → 자체 수정.

### 작은 search radius
```
"이 코드베이스 구조에 익숙하지 않으면 작은 단위로 탐색.
한 번에 큰 추측 X."
```

→ Claude Code 가 작은 step 으로 (ls → grep → read → edit) 진행 시 hallucination 최소.

## 회사 internal API hallucination

```
Bad: LLM 이 internal API X 라는 게 있다고 가정
Good: 회사 docs / repo 의 README 를 RAG 로 retrieve 후 사용
```

→ RAG / context 에 회사 internal 정보 제공 — hallucination 차단.

## Library 버전 mismatch
```python
# requirements.txt
fastapi==0.115.0

# LLM 가 옛 (0.100) 또는 새 (0.120) 추론
# → 실제 0.115 의 API 와 mismatch
```

해결: package version 을 system prompt 또는 user message 에 명시.

## 진단

```bash
# 코드 변경 후 즉시 검증
git diff
ruff check
pytest

# 추천한 file 가 실제로 있나
ls src/utils/legacy.py 2>/dev/null || echo "NOT FOUND"

# 추천한 import 가 실제로 export 되나
python -c "from mylib import myfunc"
```

## 재발 방전

- agent 룰: "verify before recommend"
- Tool 우선 (ls/grep/read) 후 코드 작성
- linting / type check / test 자동 run
- RAG 로 internal docs 주입
- 모니터링: hallucination 사례 troubleshooting_ledger 에 capture

## 관련

- [[ai-agent]] / [[tool-malformed-json]] / [[multi-turn-forgetting]]
