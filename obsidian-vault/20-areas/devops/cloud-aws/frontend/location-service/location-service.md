---
title: "Amazon Location Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:27:00+09:00
tags:
  - aws
  - frontend
  - maps
  - location
---

# Amazon Location Service

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Location Service hub |

**[[../frontend|↑ Frontend]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

지도 / 지오코딩 / 경로 / 지오펜스 — Esri / HERE 데이터 기반. **Google Maps / Mapbox 의 AWS 대안**.

## 2. 기능

| | 의미 |
| --- | --- |
| **Maps** | 타일 (raster / vector) |
| **Places** | 지오코딩 / 역지오코딩 / POI 검색 |
| **Routes** | 경로 / ETA / 매트릭스 |
| **Trackers + Geofences** | 디바이스 위치 추적 + 진입/이탈 이벤트 |

## 3. 함정

- Google Maps 데이터 풍부도 / 정확도 비교
- 한국 지도 / POI 커버리지 — 카카오 / 네이버 / 구글 검토
- 데이터 권한 (Esri vs HERE) 라이센스 확인

## 4. 관련

- [[../frontend|↑ Frontend]]
- [[../../iot/iot]] — 디바이스 tracking
