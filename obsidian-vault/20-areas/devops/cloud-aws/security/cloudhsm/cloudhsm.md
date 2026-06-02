---
title: "AWS CloudHSM"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:28:00+09:00
tags:
  - aws
  - security
  - hsm
  - cloudhsm
---

# AWS CloudHSM

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CloudHSM hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**전용 HSM 어플라이언스 (FIPS 140-2 Level 3)**. KMS 보다 강한 키 격리 — AWS 도 키 자체에 접근 불가. 규제 산업에서.

## 2. 왜 / 언제

- 금융 / 의료 / 정부 규제로 single-tenant HSM 필수
- PKCS#11 / JCE / OpenSSL 으로 직접 키 사용
- 자체 키 (BYOK) 의 더 엄격한 보관
- 거래 서명 / 인증서 발급용

## 3. CloudHSM vs KMS

| | KMS | CloudHSM |
| --- | --- | --- |
| 모델 | 매니지드 멀티테넌트 | 단독 HSM |
| FIPS | 140-2 Level 2 | **Level 3** |
| 비용 | 키당 $1 / 월 + 호출 | 시간당 ~$1.45/HSM + 클러스터 |
| 운영 | 0 | 직접 관리 (사용자 / 백업) |
| 신규 | KMS 디폴트 | 규제 필요할 때만 |

## 4. 함정

- 비싸고 운영 부담 큼 — 진짜로 필요할 때만
- AWS 도 키 못 봄 = 키 분실 시 데이터 복구 불가
- 신규는 대부분 KMS 로 충분 (FIPS Level 3 endpoint 도 제공)

## 5. 관련

- [[../security|↑ Security]]
- [[../kms/kms]]
