---
title: "도메인 — stt (Clova STT Gateway)"
kind: handover
status: stable
created_at: 2026-05-22T00:07:00+09:00
tags: [bkurs-be, handover, domain, gateway, stt, clova]
related:
  - "[[../../bkurs-be]]"
  - "[[../external-gateways]]"
home_hub: "[[../../bkurs-be]]"
---

# stt 도메인

> 음성 → 텍스트 변환 gateway. Naver Clova STT 실 구현 (사이클 2 PR #104). interview 도메인의
> audio response 가 의존 — audio URL → transcript.

## 1. 책임

- `SttService.transcribe(audioUrl) → TranscriptionResult`
- Mock ↔ Clova STT swap

## 2. 핵심 타입

| 타입 | 핵심 |
| --- | --- |
| `SttService` (interface) | `transcribe(audioUrl) → TranscriptionResult` |
| `TranscriptionResult` (record) | transcript / language / durationSec / confidence |
| `MockSttService` | `@Profile("!prod")` — fake text |
| `ClovaSttService` | `@Profile("prod")` — Clova REST adapter |
| `ClovaProperties` | `@ConfigurationProperties("clova.*")` |

## 3. 패키지 구성

```
stt/
└── gateway/
    ├── SttService                    # interface
    ├── TranscriptionResult           # record
    ├── MockSttService                # fake
    ├── ClovaSttService               # prod
    └── ClovaProperties               # config
```

## 4. 호출 패턴 (예정)

```
InterviewResponseService.attachTranscript(responseId)
  ▼
1. response = repo.findById(responseId)
2. CompletableFuture.runAsync(() -> {
       TranscriptionResult t = sttService.transcribe(response.audioUrl());
       response.setTranscript(t.transcript());
       repo.save(response);
   }, externalCallExecutor);
```

비동기 — audio 길이에 따라 수십 초 가능.

## 5. Clova 설정 (prod)

| 키 | 값 |
| --- | --- |
| `clova.client-id` | NCP App Client ID |
| `clova.client-secret` | NCP App Client Secret |
| `clova.endpoint` | `https://naveropenapi.apigw.ntruss.com/recog/v1/stt` |
| `clova.language` | ko (default) |

`.env.example` 에 키만, 실제 값은 secret manager.

## 6. 의존

- **의존**: common.config.AsyncConfig
- **의존받음**: interview (audio URL transcript)

## cross-link

- [[../external-gateways#stt]]
- [[interview]]
