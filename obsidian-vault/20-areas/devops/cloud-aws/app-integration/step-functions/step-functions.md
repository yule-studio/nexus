---
title: "AWS Step Functions"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:40:00+09:00
tags:
  - aws
  - integration
  - workflow
  - state-machine
---

# AWS Step Functions

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Step Functions hub |

**[[../app-integration|↑ Application Integration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**상태 머신 (state machine) 기반 워크플로우 오케스트레이션**. Lambda / ECS / DynamoDB / SQS / SNS / Bedrock 등 250+ AWS 서비스를 코드 없이 묶고 재시도 / 분기 / 병렬 / 대기 / 오류 처리.

## 2. 왜 / 언제

- 여러 Lambda 를 순서/조건/재시도/병렬로 묶기
- 사람 승인이 끼는 워크플로우 (`waitForTaskToken`)
- 긴 작업 (15분 넘는 Lambda 한계 회피)
- 사가 (Saga) 패턴 — 보상 트랜잭션
- LLM 다단계 호출 (Bedrock + tool use)

## 3. 두 타입

| | Standard | Express |
| --- | --- | --- |
| 지속 시간 | 최대 **1년** | 최대 **5분** |
| 실행 모델 | exactly-once | at-least-once |
| 비용 | 상태 전환 당 | 실행 시간 + 메모리 |
| 이력 | 풍부 (콘솔에서 모든 단계) | CloudWatch Logs 만 |
| 용도 | **사람 승인 / 긴 워크플로우** | **고빈도 단순 흐름** (API 백엔드) |

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **State** | Task / Pass / Choice / Wait / Parallel / Map / Succeed / Fail |
| **ASL** | Amazon States Language (JSON) |
| **Task Token** | 비동기 / 사람 승인 시 토큰을 외부 시스템에 |
| **Retry / Catch** | 자동 재시도 + 오류 분기 |
| **Map state** | 배열 입력에 대해 병렬 처리 (분산 Map 으로 수천 병렬) |
| **Parallel state** | 정의된 N 갈래를 동시에 |
| **Direct integrations** | Lambda 호출 없이 SDK 직접 호출 (250+ 서비스) |

## 5. 표준 예시 — 주문 처리

```
StartAt: ValidateOrder (Lambda)
   ├─ catch → CancelOrder
   ↓
ChargePayment (Lambda)
   ├─ catch → RefundAndCancel  (사가 보상)
   ↓
Parallel:
   ├─ ReserveInventory (DynamoDB UpdateItem)
   └─ NotifyCustomer (SNS)
   ↓
FulfillOrder (ECS RunTask)
   ↓
WaitForApproval (waitForTaskToken — 사람 승인)
   ↓
End
```

→ 시각화된 상태 머신 + 각 단계 재시도 / catch / 실행 이력.

## 6. 비용 / 함정

- Standard $25 / million state transitions — 비용 많이 안 든다고 생각하지만 큰 워크플로우는 누적
- Express는 실행 시간 + 메모리 (Lambda 와 유사)
- **direct integration** 으로 Lambda 호출 비용 줄이기 (예: DynamoDB PutItem 직접)
- 단순한 한 줄 호출에 Step Functions 쓰면 과함 — 그냥 Lambda 1 개
- 분산 Map 으로 수만 병렬 처리 가능 (S3 객체 일괄 처리)

## 7. 관련

- [[../app-integration|↑ Application Integration]]
- [[../eventbridge/eventbridge]]
- [[../../compute/lambda/lambda]]
- [[../../ml/bedrock/bedrock]]
