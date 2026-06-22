---
title: "네트워크 보안"
kind: knowledge
project: security
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - security
  - network-security
---

# 네트워크 보안

> 정의: 네트워크 트래픽의 기밀성·무결성·가용성을 보호하기 위해 전송 암호화(TLS/mTLS), 접근 통제(방화벽/세그멘테이션), 경계·내부 신뢰 모델을 설계·운영하는 분야.

**[[../security|↑ security]]**

## 1. 무엇인가

네트워크 보안은 시스템 간 "이동하는 데이터(data in transit)"와 "누가 어디에 접근할 수 있는가(연결성)"를 통제한다. 코드가 아니라 패킷·경로·연결 수준의 보안이다.

핵심 구성요소:
- 전송 암호화 — TLS, mTLS(상호 인증), VPN으로 도청·변조 차단.
- 접근 통제 — 방화벽, 보안 그룹, ACL로 허용된 연결만.
- 세그멘테이션 — 네트워크를 구역으로 나눠 횡적 이동(lateral movement) 차단.
- 탐지·차단 — IDS/IPS, NDR로 악성 트래픽 식별.
- 신뢰 모델 — 전통적 경계(perimeter) vs 제로 트러스트.

## 2. 왜 / 언제 필요한가

(1) 왜 필요한가 — 네트워크는 본질적으로 도청·중간자 공격에 노출돼 있다. 평문 트래픽은 같은 네트워크의 누구나 읽을 수 있고, 통제 없는 연결성은 한 대가 뚫리면 전체로 번지는 통로가 된다.

(2) 안 하면 무슨 문제가 생기나 — 구체적 사고 예시:
- 평문 도청: HTTP/평문 프로토콜은 공용 와이파이에서 세션 쿠키·자격증명이 그대로 노출된다.
- 중간자 공격(MITM): 인증서 검증을 건너뛰면 공격자가 트래픽을 가로채 위조한다.
- 횡적 이동: 평평한(flat) 네트워크에서 한 워크스테이션이 감염되면 랜섬웨어가 전 사내로 퍼진다. WannaCry(2017)가 SMB 취약점으로 내부망을 휩쓴 전형.
- 과대 개방: 보안 그룹을 0.0.0.0/0으로 열어두면 인터넷 전체가 DB 포트에 접근한다.
- DDoS: 트래픽 폭주로 서비스 마비.

(3) 대안/다른 접근과 비교 — 전통적 경계 모델("내부는 신뢰, 외부는 차단")은 VPN으로 들어오면 내부를 자유롭게 다닐 수 있어 횡적 이동에 취약하다. 제로 트러스트(Zero Trust, NIST SP 800-207)는 "절대 신뢰하지 말고 항상 검증"으로 위치와 무관하게 모든 연결을 인증/인가한다. 다만 전환 비용과 복잡도가 크다.

(4) 트레이드오프 — 엄격한 세그멘테이션/방화벽 규칙은 보안을 높이지만 운영 복잡도와 장애 디버깅 난이도를 키운다. mTLS는 강력하지만 인증서 발급·회전·관리 부담이 있다. IPS는 능동 차단이 강점이나 오탐 시 정상 트래픽을 끊는다.

## 3. 핵심 개념 / 어떻게

| 계층 | 위협 | 방어 |
| --- | --- | --- |
| 전송 | 도청, MITM | TLS 1.3, mTLS, VPN(WireGuard) |
| 경계/접근 | 무단 연결 | 방화벽, 보안 그룹, deny-by-default |
| 내부 | 횡적 이동 | 마이크로세그멘테이션, VLAN |
| 가용성 | DDoS | Rate limit, CDN/Anti-DDoS |
| 가시성 | 은밀한 통신 | IDS/IPS(Suricata), NDR, 플로우 로그 |

핵심 개념:
- TLS vs mTLS: TLS는 서버만 인증, mTLS는 클라이언트도 인증서로 인증(서비스 메시 내부 통신 표준).
- Defense in Depth(다층 방어): 단일 통제 실패를 다른 계층이 보완.
- Zero Trust: ID/디바이스 기반 인증, 최소 권한, 마이크로세그멘테이션.
- IDS(탐지) vs IPS(차단): 인라인 여부 차이.
- 세그멘테이션: DMZ, private 서브넷, 워크로드 격리로 폭발 반경 축소.

실무: 인터넷 노출 표면 최소화(불필요 포트 닫기), 모든 외부 통신 TLS 강제, 내부도 mTLS, 보안 그룹은 화이트리스트, 플로우 로그로 가시성 확보.

## 4. 적용 맥락 / 경계

- network-security: 트래픽 경로·연결성·전송 보안. "데이터가 안전하게 이동하나, 누가 연결되나."
- [[../cryptography/cryptography|cryptography]]: TLS가 쓰는 암호 프리미티브 자체. network는 그것을 경로에 적용.
- [[../application-security/application-security|application-security]]: TLS로 보호된 연결로 들어온 요청의 코드 처리. 네트워크는 페이로드 내용까지 못 막는다.
- [[../cloud-security/cloud-security|cloud-security]]: 클라우드에서는 VPC/보안 그룹/서브넷이 네트워크 보안의 구현 접점.

경계 예시: HTTPS 강제·방화벽은 network, AES 모드 선택은 crypto, 들어온 요청의 SQLi는 AppSec다. 시작점은 노출 표면 점검 + TLS 전수 적용 + 세그멘테이션 설계다.

## 5. 관련

- [[../security|↑ security]]
- [[../cryptography/cryptography|암호학 응용]]
- [[../cloud-security/cloud-security|클라우드 보안]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
- 외부: NIST SP 800-207(Zero Trust Architecture), NIST SP 800-41(방화벽), IETF RFC 8446(TLS 1.3), MITRE ATT&CK(Lateral Movement), OWASP Transport Layer Protection Cheat Sheet
