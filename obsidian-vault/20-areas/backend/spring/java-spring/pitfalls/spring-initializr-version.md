---
title: "Spring Initializr — boot 버전 hardcode 금지"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-20T00:30:00+09:00
tags: [pitfall, spring, initializr, setup]
home_hub: pitfalls
related:
  - "[[pitfalls]]"
  - "[[../java-spring]]"
  - "[[../../../../../10-projects/bkurs-be/research/research-spring-3-4-vs-3-5-gradle-dsl]]"
---

# Spring Initializr — boot 버전 hardcode 금지

**[[pitfalls|↑ Java Spring pitfalls]]**

## 증상

```bash
curl -sf "https://start.spring.io/starter.zip" -d "bootVersion=3.4.1&..." -o init.zip
```

→ 400 Bad Request:

```json
{"status":400,"error":"Bad Request",
 "message":"Invalid Spring Boot version '3.4.1',
            Spring Boot compatibility range is >=3.5.0"}
```

다운로드 받은 zip 이 사실은 JSON 에러 응답이라 unzip 실패:

```
End-of-central-directory signature not found.
Either this file is not a zipfile, ...
```

## 원인

Spring Initializr 의 compatibility range 는 정기 업데이트 — 옛 튜토리얼 / 메모리의 boot 버전을 hardcode 하면 6-12 개월 단위로 깨진다. 2026-05 기준 `>=3.5.0` 만 허용.

`curl -sf` (`-f` 는 4xx/5xx 에서 fail) 안 쓰면 에러 응답이 zip 파일로 저장돼 unzip 단계에서 fail — 진단이 한 단계 뒤로 밀린다.

## 해결

```bash
# 1. metadata 먼저 조회
curl -s "https://start.spring.io/metadata/client" | \
  python3 -c "import json,sys;d=json.load(sys.stdin);
              print([v['id'] for v in d['bootVersion']['values'][:6]])"
# → ['4.1.0.BUILD-SNAPSHOT', '4.0.6.RELEASE', '3.5.14.RELEASE', ...]

# 2. RELEASE 중 가장 최신을 골라 starter.zip 요청
curl -sf "https://start.spring.io/starter.zip" \
  -d "type=gradle-project&language=java&bootVersion=3.5.14&javaVersion=21&..." \
  -o init.zip
```

## 예방

- `curl -sf` 로 4xx/5xx 즉시 실패시키기
- 다운로드 직후 `file init.zip` 으로 실제 zip 인지 확인 (JSON 이면 "JSON data" 출력)
- 자동화 스크립트면 metadata 조회 → 가장 최신 RELEASE 자동 선택

## 출처 / cross-link

- 2026-05-20 [[../../../../../10-projects/bkurs-be/task-logs/task-log-setup-initial]] 에서 발생
- [[../../../../../10-projects/bkurs-be/research/research-spring-3-4-vs-3-5-gradle-dsl]]
