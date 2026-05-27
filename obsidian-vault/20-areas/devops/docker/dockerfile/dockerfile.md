---
title: "Dockerfile — 마스터 hub"
kind: knowledge
status: current
created_at: 2026-05-22T13:46:00+09:00
tags: [devops, docker, dockerfile, hub]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[../practice/practice]]"
---

# Dockerfile — 마스터 hub

**[[../docker|↑ docker]]**

> Dockerfile 작성 best practice + BuildKit 깊이. Dockerfile 은 코드 — 재현 가능한 image 정의.
> 본 hub 의 best-practices 는 매 Dockerfile 작성 시 꼭 통과.

## 1. 학습 순서

| 노트 | 내용 |
| --- | --- |
| [[best-practices]] ★ | multi-stage / cache 순서 / non-root / .dockerignore / health |
| [[buildkit-deep]] | BuildKit DAG / cache backend / mount 옵션 (cache/secret/bind/ssh) |

## 2. 핵심 규칙 (요약)

| 규칙 | 효과 |
| --- | --- |
| **base image 명시 + pin** | `FROM eclipse-temurin:21.0.5_11-jre-alpine` (digest 더 안전) |
| **multi-stage** | builder ↔ runtime 분리 — image size 90% 감소 |
| **`.dockerignore`** | `target/`, `node_modules/`, `.git/` 제외 → build context 작게 |
| **layer 캐시 순서** | 자주 변하지 않는 instruction 위로 |
| **single `RUN`** | `apt update && install && cleanup` 한 줄 |
| **non-root USER** | `USER appuser` + `chown` (보안) |
| **HEALTHCHECK** | 실패 자동 감지 |
| **`--mount=type=cache`** | dep 캐시 영구화 (BuildKit) |
| **`--mount=type=secret`** | 빌드 시 비밀 값 안전 주입 |
| **EXPOSE + ENTRYPOINT** | 마지막 stage 만 |

영상: [Dockerfile Best Practices — Bret Fisher](https://www.youtube.com/watch?v=8vXoMqWgbQQ)

## 3. 실습 매핑

| 실습 | doc |
| --- | --- |
| 04 multi-stage | [[../practice/04-multi-stage-build]] |
| 05 BuildKit cache + secret | [[../practice/05-buildkit-cache-and-secret]] |
| 06 multi-platform | [[../practice/06-multi-platform-build]] |
| 10 image tags / layers | [[../practice/10-image-tags-and-layers]] |

## 4. 학습 영상

| 영상 | 길이 |
| --- | --- |
| [Dockerfile Best Practices — Bret Fisher](https://www.youtube.com/watch?v=8vXoMqWgbQQ) | 45분 |
| [Multi-stage builds — Docker 공식](https://www.youtube.com/watch?v=zpkqNPwEzac) | 30분 |
| [BuildKit Deep Dive — Tonis Tiigi](https://www.youtube.com/watch?v=kkpQ_UZn2bo) | 50분 |
| [Distroless images — Google](https://www.youtube.com/watch?v=lviLZFciDv4) | 30분 |

## 5. 참고

- 공식 Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- BuildKit: https://github.com/moby/buildkit
- hadolint (Dockerfile linter): https://github.com/hadolint/hadolint
- dive (layer 분석): https://github.com/wagoodman/dive
- Distroless: https://github.com/GoogleContainerTools/distroless

## 6. 관련

- [[../docker]]
- [[../fundamentals/fundamentals]]
- [[../desktop/builder-and-buildkit]] (예정) — Desktop GUI 빌드
- [[../practice/practice]] — 실습 마스터 시리즈
