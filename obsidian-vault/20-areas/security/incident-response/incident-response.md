---
title: "Incident Response"
kind: knowledge
project: security
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - security
  - incident-response
---

# Incident Response

> 정의: 보안 사고를 탐지·억제·근절·복구하고 사후 교훈을 체계화하는 절차로, NIST SP 800-61 IR 생애주기를 따른다.

**[[../security|↑ security]]**

## 1. 무엇인가

Incident Response(IR, 침해사고 대응)는 "사고는 결국 일어난다"를 전제로, 발생 시 피해를 최소화하고 정상 운영으로 빠르게 복귀하기 위한 사전 준비된 절차와 조직 체계다.

NIST SP 800-61의 IR 생애주기:
1. 준비(Preparation) — IR 플랜, 도구, 연락망, 권한, 로그 수집 체계.
2. 탐지·분석(Detection & Analysis) — 이상 징후 식별, 사고 등급 분류(triage).
3. 억제·근절·복구(Containment, Eradication, Recovery) — 확산 차단, 원인 제거, 서비스 복원.
4. 사후 활동(Post-Incident Activity) — 포스트모템, 교훈 반영.

핵심 구성요소:
- IR 플랜/플레이북 — 시나리오별 대응 절차.
- CSIRT — 사고 대응 전담 조직과 역할(R&R).
- 로그/원격측정 — SIEM, EDR, 네트워크 플로우.
- 포렌식 — 증거 보존, 타임라인 재구성.
- 커뮤니케이션 — 내부 보고, 규제 신고, 고객 고지.

## 2. 왜 / 언제 필요한가

(1) 왜 필요한가 — 예방이 100%일 수 없다. 사고 발생 시 우왕좌왕하면 피해가 기하급수로 커진다. 미리 정의된 절차가 있어야 새벽 3시에도 "누가 무엇을 먼저 하는가"가 명확하다. 또한 다수 규제(GDPR 72시간, 국내 개인정보 유출 통지)가 신고 기한을 법으로 강제한다.

(2) 안 하면 무슨 문제가 생기나 — 구체적 사고 예시:
- 탐지 지연: 평균 침해 탐지(dwell time)가 수개월에 달하면 공격자가 그 사이 데이터를 전부 빼낸다.
- 잘못된 억제: 패닉으로 서버를 즉시 끄면 휘발성 메모리 증거가 사라져 원인 파악이 불가능해진다(포렌식 실패).
- 신고 지연: GDPR 72시간을 넘기면 과징금이 가중된다.
- 백도어 잔존: 근절 없이 복구만 하면 공격자가 남긴 백도어로 재침해된다.

(3) 대안/다른 접근과 비교 — 외주 MDR/IR 리테이너 vs 내부 CSIRT 구축. 내부는 맥락 이해가 빠르지만 24/7 대응 인력 부담이 크고, 외주는 즉시 전문성을 쓰지만 시스템 이해에 시간이 걸린다. 자동 격리(SOAR)는 빠르지만 오탐 시 정상 서비스를 차단할 위험이 있다.

(4) 트레이드오프 — 억제 속도 vs 증거 보존: 빨리 끊으면 확산은 막지만 증거가 날아간다. 격리 자동화 vs 오탐 영향. 투명한 고객 고지 vs 평판 리스크. 사고 와중에 이 균형을 즉석에서 판단하지 않도록 사전에 플레이북으로 정해 둔다.

## 3. 핵심 개념 / 어떻게

| 단계 | 핵심 활동 | 도구 |
| --- | --- | --- |
| 탐지 | 알림 수집·상관분석 | SIEM(Splunk), EDR(CrowdStrike) |
| 분석 | 스코프 산정, IOC 추출 | 로그 분석, 위협 인텔 |
| 억제 | 네트워크 격리, 계정 차단 | EDR isolate, 방화벽 |
| 근절 | 멀웨어/백도어 제거, 패치 | 포렌식 클린업 |
| 복구 | 깨끗한 백업에서 복원, 모니터링 강화 | IaC 재배포 |
| 사후 | 포스트모템(blameless) | 타임라인 문서화 |

핵심 개념:
- IOC(Indicator of Compromise): 악성 IP/해시/도메인 등 침해 흔적.
- 증거 보존 순서(Order of Volatility): 휘발성 높은 것(메모리, 네트워크 연결)부터 수집 후 디스크.
- Chain of Custody(증거 연속성): 법적 효력을 위한 증거 취급 기록.
- MITRE ATT&CK: 공격자 전술·기법(TTP) 매핑으로 탐지/대응 갭 분석.
- Blameless Postmortem: 개인 비난이 아니라 시스템/프로세스 개선에 집중.
- MTTD/MTTR: 탐지/복구 평균 시간 — IR 성숙도 지표.

실무: 테이블탑 훈련(모의 시나리오)으로 플레이북을 정기 검증한다. SOAR로 반복 대응을 자동화하되 파괴적 조치는 사람 승인 게이트를 둔다.

## 4. 적용 맥락 / 경계

- incident-response: "사고가 난 뒤" 탐지·대응·복구·교훈. 운영적·시간 민감 활동.
- [[../application-security/application-security|application-security]] / [[../cloud-security/cloud-security|cloud-security]]: 사고를 애초에 줄이는 "사전 예방" 계층. IR은 이들이 실패했을 때의 안전망.
- [[../pentesting/pentesting|pentesting]]: 공격자 관점으로 사전 약점 발견. IR은 실제 사고의 사후 관점.
- [[../compliance/compliance|compliance]]: 신고 의무·증거 보관 기한 등을 IR 절차에 요구.

경계 예시: 취약점을 미리 찾으면 pentesting, 그 취약점이 실제 악용된 사건을 처리하면 IR이다. 시작점은 IR 플랜 작성 + 로그 수집 체계 확립 + 연락망 정비다.

## 5. 관련

- [[../security|↑ security]]
- [[../pentesting/pentesting|침투 테스트]]
- [[../compliance/compliance|컴플라이언스]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
- 외부: NIST SP 800-61(Computer Security Incident Handling Guide), MITRE ATT&CK, SANS Incident Handler's Handbook, ENISA IR 가이드, KISA 침해사고 대응 가이드
