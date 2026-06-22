---
title: "클라우드 보안"
kind: knowledge
project: security
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - security
  - cloud-security
---

# 클라우드 보안

> 정의: AWS/GCP/Azure 등 클라우드 인프라의 신원·권한(IAM), 네트워크, 데이터, 설정을 책임공유모델에 따라 안전하게 운영하는 분야.

**[[../security|↑ security]]**

## 1. 무엇인가

클라우드 보안은 온프레미스와 달리 책임 공유 모델(Shared Responsibility Model) 위에서 작동한다. 클라우드 제공자는 "클라우드 자체(of the cloud)"의 보안(하드웨어, 하이퍼바이저)을 책임지고, 사용자는 "클라우드 안(in the cloud)"의 보안(데이터, IAM, 설정, 네트워크 구성)을 책임진다.

핵심 구성요소:
- IAM(Identity and Access Management) — 사용자/역할/정책으로 최소 권한 부여.
- 네트워크 격리 — VPC, 서브넷, 보안 그룹(Security Group), NACL.
- 데이터 보호 — 저장 시 암호화(KMS), 전송 시 암호화(TLS).
- CSPM(Cloud Security Posture Management) — 잘못된 설정 자동 탐지.
- 시크릿/키 관리 — Secrets Manager, KMS, HSM.
- 로깅/감사 — CloudTrail, Cloud Audit Logs.

## 2. 왜 / 언제 필요한가

(1) 왜 필요한가 — 클라우드는 API 한 번 호출로 리소스가 전 세계에 노출될 수 있다. 잘못된 설정 하나가 곧바로 인터넷 전체에 데이터를 공개한다. 클라우드 침해의 대다수는 해킹이 아니라 사용자 측 설정 실수에서 비롯된다.

(2) 안 하면 무슨 문제가 생기나 — 구체적 사고 예시:
- 공개 S3 버킷: 수많은 데이터 유출이 "public-read"로 잘못 설정된 S3에서 발생했다. 2019년 Capital One 사고는 잘못 구성된 WAF/IAM이 SSRF와 결합돼 1억 명 데이터가 유출됐다.
- 과대 권한 IAM: `AdministratorAccess`를 붙인 키가 유출되면 계정 전체가 탈취된다. 공격자는 신규 IAM 사용자를 만들고 흔적을 지운다.
- 노출된 메타데이터 서비스(IMDSv1): SSRF로 인스턴스 자격증명을 빼내 권한 상승.
- 비활성화된 로깅: CloudTrail이 꺼져 있으면 침해 사실조차 탐지/포렌식이 불가능하다.

(3) 대안/다른 접근과 비교 — 전통적 데이터센터(온프레미스)는 물리 경계가 명확하지만 확장성/탄력성이 떨어진다. 클라우드는 민첩하지만 경계가 "정책으로 정의"되므로 설정 오류 위험이 크다. 멀티클라우드는 벤더 락인을 줄이지만 보안 모델이 제각각이라 운영 복잡도가 급증한다.

(4) 트레이드오프 — 최소 권한을 엄격히 적용하면 개발 속도가 느려지고 "권한 부족" 장애가 잦다. 반대로 넓은 권한은 편하지만 폭발 반경(blast radius)이 커진다. CSPM 알림을 다 대응하려면 노이즈가 많고, 무시하면 진짜 위협을 놓친다.

## 3. 핵심 개념 / 어떻게

| 영역 | 위험 | 방어 |
| --- | --- | --- |
| IAM | 과대 권한, 장기 액세스 키 | 최소 권한, 역할 기반(STS 임시 자격), MFA |
| 네트워크 | 0.0.0.0/0 인바운드 개방 | 보안 그룹 최소 개방, private 서브넷 |
| 스토리지 | 공개 버킷, 미암호화 | Block Public Access, KMS 암호화 |
| 시크릿 | 하드코딩된 키 | Secrets Manager, 자동 회전 |
| 가시성 | 로그 부재 | CloudTrail/감사로그 상시 활성 |

주요 도구/기법:
- CSPM: Prowler, ScoutSuite, AWS Security Hub — 설정 드리프트 탐지.
- IaC 스캔: Checkov, tfsec, Terrascan — Terraform 배포 전 잘못된 설정 차단(shift-left).
- CWPP: 워크로드(컨테이너/VM) 런타임 보호.
- CIEM: 권한 과다 분석.

CIS Benchmarks(AWS/GCP/Azure)가 설정 기준선의 사실상 표준이다. 원칙: deny-by-default, 최소 권한, 모든 곳에 암호화, 변경 불가능한 인프라(immutable), 상시 로깅.

## 4. 적용 맥락 / 경계

- cloud-security: 인프라 설정·IAM·클라우드 네이티브 서비스. "리소스가 안전하게 구성됐나."
- [[../application-security/application-security|application-security]]: 그 인프라 위에서 도는 코드의 결함.
- [[../network-security/network-security|network-security]]: 전송 경로·트래픽 통제. 클라우드에서는 VPC/보안 그룹이 그 접점.
- [[../compliance/compliance|compliance]]: 클라우드 설정이 규정(PCI-DSS 등)을 충족하는지 증명.

경계 예시: 코드의 SQLi는 AppSec, S3 public 설정은 cloud-security, 두 VPC 간 트래픽 암호화는 network-security다. 시작점은 책임공유모델 이해 + CIS Benchmark 자가 점검 + CSPM 도입이다.

## 5. 관련

- [[../security|↑ security]]
- [[../network-security/network-security|네트워크 보안]]
- [[../compliance/compliance|컴플라이언스]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
- 외부: AWS Well-Architected Framework(Security Pillar), CIS Benchmarks, NIST SP 800-53, Cloud Security Alliance(CSA) CCM, MITRE ATT&CK for Cloud
