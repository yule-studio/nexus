---
title: "60-troubleshooting/web — 웹 운영 장애 카탈로그"
kind: knowledge
status: current
created_at: 2026-06-15
tags: [troubleshooting, web, cache, resilience, security, hub]
home_hub: "60-troubleshooting"
---

# 웹 운영 장애 카탈로그

> 특정 프레임워크가 아니라 **웹/분산 시스템 자체**에서 반복되는 실전 장애 —
> 캐시·재시도·정규식·CDN 캐시처럼 규모가 커지면 누구나 만나는 것들.
> 보안 계열의 *원리*는 [[../../20-areas/security/web-security/web-security|20-areas/security/web-security]] 에.

**[[../troubleshooting|↑ 60-troubleshooting]]**

---

## 카탈로그

| # | 증상 | 한 줄 원인 | 핵심 해결 | 깊이 노트 | 대표 실제 사례 |
| -- | --- | --- | --- | --- | --- |
| 1 | 핫 키 만료 순간 DB 부하 스파이크 | 동시 캐시 미스가 전부 원본으로 몰림 | single-flight / 조기만료(XFetch) / SWR / TTL 지터 | [[cache-stampede]] | FB memcache leases 논문 |
| 2 | 트리거 사라졌는데 과부하 고착 | 재시도 증폭 → 메타스테이블 평형 | 백오프+지터 / 재시도 예산 / 서킷브레이커 / load shedding | [[retry-storm-metastable]] | AWS S3 2017 us-east-1 |
| 3 | 특정 입력 하나에 CPU 100% 고착 | 정규식 카타스트로픽 백트래킹 | RE2(선형 엔진) / 패턴 수정 / 입력 길이·타임아웃 | [[redos-regex-dos]] | Cloudflare 2019 / SO 2016 |
| 4 | 한 사용자에게 보일 응답이 전체에 노출 | unkeyed 입력 반사 / 확장자 캐시 오판 | unkeyed 입력 제거 / 캐시 키 정규화 / no-store | [[cdn-cache-poisoning]] | Kettle·Gil 캐시 연구 |
| 5 | 방금 발급 토큰이 만료/미활성, 정확한 OTP가 거부 | 노드 간 시계 드리프트(NTP 실패) | NTP/chrony 동기화 / JWT leeway / TOTP ±1 | [[clock-skew-auth]] | — |
| 6 | 로그인 트래픽·401 폭증, 계정 탈취 물결 | 유출 비번 자동 대입(분산 봇) | 다축 레이트리밋 / 유출비번 차단 / MFA | [[credential-stuffing-wave]] | 산업 상시 위협 |
| 7 | 배포 때마다 502/503·connection reset 버스트 | LB 디레지스터 지연 / in-flight 강제 종료 | preStop 드레이닝 / graceful shutdown / deregistration delay | [[graceful-shutdown-draining]] | — |
| 8 | 대역폭 정상인데 신규 연결 거부, 워커 포화 | 느린 연결이 연결 슬롯 점유(low-and-slow) | 타임아웃 강화 / 비동기 프록시 버퍼링 / limit_conn | [[slowloris-slow-clients]] | Slowloris 2009 |
| 9 | 파티션 회복 후 데이터 충돌/유실, 2 leaders | 이중 프라이머리(split-brain) | 쿼럼(과반) / 펜싱(STONITH) / 리더 리스 | [[split-brain-network-partition]] | GitHub 2018 |
| 10 | 클러스터 한가한데 한 샤드만 100% throttle | 특정 키에 부하 편향(셀럽/바이럴) | 키 솔팅 / 핫키 캐싱 / 하이브리드 fanout | [[hot-partition-celebrity-key]] | DynamoDB hot partition |

## 이 카탈로그의 관통 주제

- **부하 증폭 연쇄**: 캐시 스탬피드 → DB 포화 → 타임아웃 → 재시도 폭풍. 1번과
  2번은 같은 사고의 앞뒤다. 한쪽 방어만으론 부족.
- **신뢰 못할 입력 = 가용성 위협**: ReDoS(정규식 입력), 캐시 포이즈닝(헤더) 처럼
  "보안 입력 검증"을 빼먹으면 곧 **운영 장애(DoS)**로 돌아온다.
- 그래서 이 폴더(운영 장애)와 [[../../20-areas/security/web-security/web-security|web-security]](보안 원리)는
  같은 문제를 두 각도에서 본다 — 서로 cross-link.

## 관련

- [[../troubleshooting|↑ 60-troubleshooting]]
- [[../../20-areas/security/web-security/web-security|web-security — OWASP 허브]]
- [[../network/network|network 트러블]] — TLS / DNS / proxy 경계 (DNS stale 유입은 [[../network/dns-resolve-fail]])
- [[../backend/backend|backend 트러블]] — thread pool / 결제 멱등성 경계
- [[../database/database|database 트러블]] — Redis OOM / connection pool 경계
