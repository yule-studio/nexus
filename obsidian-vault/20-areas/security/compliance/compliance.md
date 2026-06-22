---
title: "컴플라이언스"
kind: knowledge
project: security
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - security
  - compliance
---

# 컴플라이언스

> 정의: 법규·산업 표준·계약상 요구사항(ISO 27001, SOC 2, GDPR, PCI-DSS 등)을 조직의 통제로 구현하고, 이를 증거로 입증하는 보안 거버넌스 분야.

**[[../security|↑ security]]**

## 1. 무엇인가

컴플라이언스는 "우리가 안전하게 운영한다"를 외부가 검증 가능한 형태로 증명하는 활동이다. 보안이 기술적 통제라면, 컴플라이언스는 그 통제를 정책·절차·증거(audit trail)로 문서화하고 제3자 감사를 통과하는 것이다.

핵심 구성요소:
- 통제(Control) — 위험을 줄이는 구체적 조치(예: 접근 통제, 암호화, 로그 보관).
- 정책/절차 — 통제의 근거가 되는 문서화된 규칙.
- 증거(Evidence) — 통제가 실제 작동함을 보여주는 로그/스크린샷/설정.
- 감사(Audit) — 내부/외부 평가자가 준수 여부 확인.
- 위험 평가(Risk Assessment) — 무엇을 얼마나 보호할지 결정.

주요 프레임워크:
- ISO/IEC 27001 — 정보보안 경영시스템(ISMS) 국제 표준. 인증 가능.
- SOC 2 — AICPA Trust Services Criteria(보안/가용성/기밀성/처리무결성/프라이버시). SaaS 업계 사실상 표준.
- GDPR — EU 개인정보보호 규정. 위반 시 전 세계 매출 4% 또는 2천만 유로 과징금.
- PCI-DSS — 카드 결제 데이터 보호 표준.
- 국내: 개인정보보호법, ISMS-P(KISA), 정보통신망법.

## 2. 왜 / 언제 필요한가

(1) 왜 필요한가 — B2B 계약, 규제 산업(금융/의료), 글로벌 진출에서 컴플라이언스는 사업의 전제 조건이다. 엔터프라이즈 고객은 SOC 2 보고서 없이는 계약하지 않는다. 또한 표준은 검증된 통제 체크리스트라 "무엇을 빠뜨렸는지" 가늠자가 된다.

(2) 안 하면 무슨 문제가 생기나 — 구체적 피해 예시:
- 규제 과징금: GDPR 위반으로 Meta는 2023년 12억 유로 과징금을 받았다. 국내에서도 개인정보 유출 시 매출 기반 과징금이 부과된다.
- 계약 상실: SOC 2가 없으면 엔터프라이즈 영업이 막힌다.
- 침해 후 가중 책임: 적절한 통제 없이 데이터가 유출되면 손해배상과 신뢰 붕괴가 동시에 온다.
- 카드사 제재: PCI-DSS 미준수 시 가맹 자격 박탈, 거래 정지.

(3) 대안/다른 접근과 비교 — "보안만 잘하면 되지 컴플라이언스는 형식 아니냐"는 시각이 있으나, 증거 없는 보안은 외부가 신뢰할 수 없다. 반대로 컴플라이언스만 추구하면 "체크박스 보안"(문서는 완벽한데 실제는 취약)에 빠진다. 둘은 보완 관계다.

(4) 트레이드오프 — 인증 취득/유지에는 상당한 비용과 인력, 지속적 증거 수집 부담이 든다. 통제를 과하게 적용하면 개발 민첩성이 떨어진다. 반대로 최소한만 하면 감사를 통과해도 실질 보안은 약하다. 자동화(Drata/Vanta류 GRC 도구)로 증거 수집 부담을 줄이는 것이 추세다.

## 3. 핵심 개념 / 어떻게

| 프레임워크 | 대상 | 인증 형태 |
| --- | --- | --- |
| ISO 27001 | 정보보안 경영 전반 | 인증서(3년 주기) |
| SOC 2 Type I/II | 서비스 조직 통제 | 감사 보고서(Type II=기간 검증) |
| GDPR | EU 개인정보 처리 | 규정 준수(인증 아님) |
| PCI-DSS | 카드 결제 데이터 | 자가평가(SAQ) 또는 QSA 감사 |
| ISMS-P | 국내 정보보호·개인정보 | KISA 인증 |

핵심 개념:
- 통제 매핑(Control Mapping): 여러 프레임워크의 중복 통제를 한 번에 충족(예: ISO 27001 암호화 통제 = PCI-DSS 요구 4번).
- 적용 명세서(SoA, Statement of Applicability): ISO 27001에서 어떤 통제를 적용/제외했는지 명시.
- 위험 처리(Risk Treatment): 수용/완화/이전/회피.
- 지속적 모니터링: 일회성 통과가 아니라 상시 증거 유지.

실무: GRC 플랫폼(Vanta, Drata, OneTrust)으로 클라우드 설정/접근 로그를 자동 수집해 증거화한다. NIST CSF(Identify/Protect/Detect/Respond/Recover)는 통제 체계를 잡는 출발점으로 자주 쓰인다.

## 4. 적용 맥락 / 경계

- compliance: "통제가 존재하고 증거로 입증되는가" — 거버넌스/문서/감사 관점.
- [[../application-security/application-security|application-security]] / [[../cloud-security/cloud-security|cloud-security]]: 컴플라이언스가 요구하는 기술 통제의 실제 구현체.
- [[../incident-response/incident-response|incident-response]]: GDPR의 72시간 신고 의무 등 컴플라이언스가 IR 절차에 직접 요구사항을 부과.

경계 예시: TLS 적용은 network/crypto의 일이고, "TLS가 적용됐음을 감사 증거로 남기는 것"이 컴플라이언스다. 시작점은 사업이 요구하는 프레임워크 식별(SaaS면 SOC 2, EU 사용자면 GDPR) → 갭 분석이다.

## 5. 관련

- [[../security|↑ security]]
- [[../cloud-security/cloud-security|클라우드 보안]]
- [[../incident-response/incident-response|Incident Response]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
- 외부: ISO/IEC 27001, AICPA SOC 2(Trust Services Criteria), EU GDPR, PCI-DSS v4.0, NIST Cybersecurity Framework(CSF), KISA ISMS-P
