---
title: "AWS Device Farm"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:26:00+09:00
tags:
  - aws
  - frontend
  - mobile
  - testing
---

# AWS Device Farm

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Device Farm hub |

**[[../frontend|↑ Frontend]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

실제 모바일 디바이스 / 브라우저 **테스트 팜**. iOS / Android 다양한 모델 + 버전에서 자동 테스트.

## 2. 왜 / 언제

- 앱 / 모바일 웹 모든 디바이스 호환성 검증
- Appium / Espresso / XCUITest CI 통합
- 원격 실 디바이스로 수동 디버그
- 응용 게시 전 회귀 테스트

## 3. 함정

- 디바이스 분 단위 비용 — CI 마다 매번 돌리면 누적
- 대안: BrowserStack / Sauce Labs (SaaS) 가 UI / 통합 더 친숙
- 신규 React Native / Flutter 워크플로우와 친화도 검토

## 4. 관련

- [[../frontend|↑ Frontend]]
- [[../../dev-tools/codepipeline/codepipeline]]
