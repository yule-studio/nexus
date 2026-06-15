---
title: "AWS Systems Manager (SSM)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:46:00+09:00
tags:
  - aws
  - management
  - operations
  - ec2
---

# AWS Systems Manager (SSM)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | SSM hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

EC2 / 온프레 서버 / 컨테이너의 **운영 도구 묶음**. **SSH 없이 접속 + 패치 + 설정 배포 + 인벤토리 + 자동화 + 설정 저장**.

---

## 2. 왜 / 언제

- SSH key / 22 port / Bastion 없이 EC2 접속 (보안 / 감사)
- 패치 / 명령 실행 / 설정 배포를 fleet 단위로
- secret / 설정값 통합 ([[../../security/parameter-store/parameter-store|Parameter Store]])
- 응용 / OS 인벤토리 자동 수집
- 운영 작업의 IAM 권한 + CloudTrail 감사

---

## 3. SSM 의 주요 capability

| capability | 의미 |
| --- | --- |
| **Session Manager** | 브라우저 / CLI 로 EC2 접속 (SSH 대체) |
| **Run Command** | 여러 인스턴스에 명령 실행 (`aws ssm send-command`) |
| **Patch Manager** | OS / 응용 패치 스케줄링 |
| **State Manager** | 원하는 상태 (config) 강제 |
| **Parameter Store** | 설정 / secret KV ([[../../security/parameter-store/parameter-store]]) |
| **Inventory** | 설치된 패키지 / 응용 자동 수집 |
| **Automation** | Runbook 실행 (예: AMI 만들기) |
| **Maintenance Window** | 운영 작업 시간 윈도우 |
| **Distributor** | 패키지 배포 |
| **Application Manager / OpsCenter** | 운영 view / 이슈 추적 |

→ 가장 자주 쓰는 것: **Session Manager + Parameter Store + Run Command + Patch Manager**.

---

## 4. SSM Agent

EC2 안에 설치된 daemon. Amazon Linux / Ubuntu / Windows AMI 는 **사전 설치됨**.

- 인스턴스가 SSM 과 통신하려면 IAM role 필요: `AmazonSSMManagedInstanceCore` (managed policy)
- VPC 안에서 인터넷 없이 쓰려면 **VPC Endpoint** (`ssm`, `ssmmessages`, `ec2messages`)

---

## 5. Session Manager — SSH 대체

```bash
# EC2 접속 (key X, 22 port X)
aws ssm start-session --target i-0abc1234...

# Port forwarding (RDS 접근 등)
aws ssm start-session \
  --target i-0abc1234... \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.xxx.rds.amazonaws.com"],"portNumber":["3306"],"localPortNumber":["13306"]}'
```

장점:
- **22 port 닫음** — 공격면 ↓
- **IAM 권한 + MFA** 로 접근 제어
- **CloudTrail + S3** 로 세션 로그 (입력/출력) 저장
- Bastion 불필요

---

## 6. 표준 패턴

| 작업 | 도구 |
| --- | --- |
| EC2 접속 | Session Manager |
| 응용 설정 / secret | Parameter Store + IAM |
| OS 패치 | Patch Manager + Maintenance Window |
| 대량 명령 | Run Command + tag 기반 필터 |
| AMI 자동 빌드 | Automation Runbook |
| fleet 인벤토리 | Inventory + Resource Data Sync → S3 |

---

## 7. 함정 / 비용

- **무료** (대부분). 다만 Patch / Automation 의 SSM Advanced tier 는 일부 과금.
- **IAM role 누락** — Agent 가 통신 못함 → 콘솔에서 "instance not visible".
- **VPC Endpoint 누락** — private subnet 에서 통신 X (NAT 거치게 되거나 fail).
- **Agent 버전 오래됨** — Patch Manager 가 Inventory 자동 갱신.
- **Session 로그** — S3 / CloudWatch Logs 로 흘리려면 사전 설정.

---

## 8. 관련

- [[../management|↑ Management]]
- [[../../security/parameter-store/parameter-store]]
- [[../../security/secrets-manager/secrets-manager]]
- [[../../compute/ec2/ec2]]
- [[../../iam/iam]]
