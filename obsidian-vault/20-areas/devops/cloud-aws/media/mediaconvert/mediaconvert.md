---
title: "AWS Elemental MediaConvert"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:30:00+09:00
tags:
  - aws
  - media
  - transcoding
  - mediaconvert
---

# AWS Elemental MediaConvert

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MediaConvert hub |

**[[../media|↑ Media]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

파일 기반 **VOD 트랜스코딩** — 원본 비디오 → HLS / DASH / MP4 의 여러 해상도 / 비트레이트. S3 → S3.

## 2. 왜 / 언제

- VOD 플랫폼 (Netflix-like) 의 트랜스코딩
- 사용자 업로드 영상 자동 처리
- ABR (Adaptive Bitrate) 패키징
- 자막 (caption) 처리 + DRM

## 3. 함정

- 분 단위 (Basic vs Pro tier) — HEVC / 4K = 비쌈
- FFmpeg + EC2 Spot 자체 운영이 더 저렴할 수도 (작은 규모)
- preset / job template 관리

## 4. 관련

- [[../media|↑ Media]]
- [[../mediapackage/mediapackage]]
- [[../../storage/s3/s3]]
