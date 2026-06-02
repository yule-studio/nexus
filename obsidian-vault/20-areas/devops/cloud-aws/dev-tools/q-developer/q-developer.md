---
title: "Amazon Q Developer"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:35:00+09:00
tags:
  - aws
  - dev-tools
  - ai
  - q-developer
---

# Amazon Q Developer

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Q Developer hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

> 옛 이름: **Amazon CodeWhisperer** (2024 rebrand + 확장).

---

## 1. 한 줄

LLM 기반 **코드 자동완성 + 채팅 + IaC / 운영 Q&A**. IDE 안에서 코드 제안, AWS 콘솔 안에서 자연어로 자원 조회 / 명령. GitHub Copilot 의 AWS 버전.

## 2. 왜 / 언제

- IDE (VS Code / JetBrains) 에서 코드 자동완성
- AWS 콘솔 안에서 자연어 — "내 prod RDS 디스크 80% 이상인 것"
- CloudFormation / CDK / Terraform 스니펫 생성
- 코드 안의 보안 / 라이선스 / 취약점 스캔 (`Q Code Review`)
- 단위 테스트 자동 생성
- legacy 코드 마이그레이션 (Java 8 → 17, .NET Framework → .NET)

## 3. 두 가지 사용 표면

| 표면 | 용도 |
| --- | --- |
| **IDE 플러그인** | 코드 완성, /chat, /fix, /explain, /tests |
| **AWS Console** | 콘솔 우상단 채팅 — 자원 조회, 비용 분석, 트러블슈팅 |

## 4. 핵심 차이 — Bedrock vs Q

| | Q Developer | Bedrock |
| --- | --- | --- |
| 위치 | IDE / Console (응용 X) | 응용 백엔드 |
| 학습 데이터 | AWS 도큐 / 콘솔 컨텍스트 | 일반 모델 |
| 자원 접근 | IAM 권한 기반 (내 account 의 자원) | 응용이 명시적 전달 |
| 비용 | Free / Pro ($19/user/월) | 토큰 단위 |

## 5. 함정

- 코드 제안 = 라이선스 / 보안 검토 권장 (auto-suggest reference tracking)
- 큰 컨텍스트는 chat 윈도우보다 IDE Q chat 이 컨텍스트 잡기 더 정확
- private code 학습은 Pro 의 "Customizations" 기능 (사설 GitHub 리포 학습)
- IAM 권한이 없으면 Console Q 가 답을 못 함

## 6. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../../ml/bedrock/bedrock]]
- [[../../../ai-agent/ai-agent|↗ AI Agent]]
