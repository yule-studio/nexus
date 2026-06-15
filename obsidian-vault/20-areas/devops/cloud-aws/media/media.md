---
title: "AWS Media Services (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:02:00+09:00
tags:
  - aws
  - media
  - streaming
  - hub
---

# AWS Media Services (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> 비디오 트랜스코딩 / 라이브 / 패키징 / 광고. AWS Elemental 브랜드.

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **MediaConvert** | [[mediaconvert]] | 파일 기반 트랜스코딩 (VOD) |
| **MediaLive** | [[medialive]] | 라이브 인코딩 |
| **MediaPackage** | [[mediapackage]] | 라이브 / VOD 패키징 (HLS / DASH / CMAF) |
| **MediaTailor** | [[mediatailor]] | 동적 광고 삽입 (SSAI) |
| **IVS** | [[ivs]] | 매니지드 저지연 라이브 (인터랙티브) |
| **MediaStore** | (hub 안 — deprecate 추세) | 저지연 origin (MediaPackage 통합 추세) |

---

## 2. 표준 라이브 흐름

```
카메라/스튜디오 → MediaLive (인코딩)
                       ↓
                  MediaPackage (HLS/DASH 패키징)
                       ↓
                  CloudFront (CDN)
                       ↓
                  플레이어
```

VOD: 원본 → MediaConvert → S3 → CloudFront.

---

## 3. 관련

- [[../cloud-aws|↑ AWS]]
- [[../storage/s3/s3]]
- [[../network/cloudfront/cloudfront]]
