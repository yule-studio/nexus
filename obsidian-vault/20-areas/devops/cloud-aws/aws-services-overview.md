---
title: "AWS Services Overview — 전체 서비스 인덱스"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:00:00+09:00
tags:
  - aws
  - cloud
  - devops
  - index
  - overview
related:
  - "[[cloud-aws]]"
home_hub: "[[cloud-aws]]"
---

# AWS Services Overview

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | AWS 전체 서비스 카테고리별 인덱스 (기존 deep 노트 cross-link) |

**[[cloud-aws|↑ AWS Cloud Hub]]**

---

## 1. 목적

이 문서는 AWS 의 주요 서비스를 카테고리별로 정리하여, 백엔드/DevOps 개발자가 각 서비스의 역할과 사용 맥락을 빠르게 이해할 수 있도록 돕는 **전체 서비스 인덱스**다.

AWS 는 200개 이상의 클라우드 서비스를 제공하며, 세부 제품과 기능까지 포함하면 약 240개 이상의 서비스로 볼 수 있다.

이 문서는 기존 AWS 관련 문서를 대체하지 않는다. 이미 작성된 문서는 그대로 유지하고, 이 문서는 전체 AWS 서비스의 빠른 탐색용 인덱스 역할만 한다.

---

## 2. 기존 문서 확인 결과

문서 작성 전에 vault 내부에서 기존 AWS 관련 문서를 검색한 결과를 정리한다.

| 카테고리 hub                        | 기존 deep 문서                             |                                                      |                                         |                                         |                                          |                                             |                |
| ------------------------------- | -------------------------------------- | ---------------------------------------------------- | --------------------------------------- | --------------------------------------- | ---------------------------------------- | ------------------------------------------- | -------------- |
| [[cloud-aws]] (전체 hub)          | 전체 카테고리 분류 + 시작 / CLI / Region / 비용 함정 |                                                      |                                         |                                         |                                          |                                             |                |
| [[compute/compute]]             | [[compute/ec2/ec2                      | EC2]] · [[compute/ecs/ecs                            | ECS]] · [[compute/eks/eks               | EKS]] · [[compute/lambda/lambda         | Lambda]] · [[compute/ecr/ecr             | ECR]] · [[compute/auto-scaling/auto-scaling | Auto Scaling]] |
| [[storage/storage]]             | [[storage/s3/s3                        | S3]] · [[storage/ebs/ebs                             | EBS]] · [[storage/efs/efs               | EFS]]                                   |                                          |                                             |                |
| [[database/database]]           | [[database/rds/rds                     | RDS]] · [[database/aurora/aurora                     | Aurora]] · [[database/dynamodb/dynamodb | DynamoDB]]                              |                                          |                                             |                |
| [[network/network]]             | [[network/vpc/vpc                      | VPC]] · [[network/alb/alb                            | ALB]] · [[network/nlb/nlb               | NLB]] · [[network/cloudfront/cloudfront | CloudFront]] · [[network/route53/route53 | Route 53]]                                  |                |
| [[iam/iam]]                     | IAM (별도 top-level 카테고리)                |                                                      |                                         |                                         |                                          |                                             |                |
| [[security/security]]           | [[security/kms/kms                     | KMS]] · [[security/secrets-manager/secrets-manager   | Secrets Manager]]                       |                                         |                                          |                                             |                |
| [[messaging/messaging]]         | [[messaging/sqs/sqs                    | SQS]] · [[messaging/sns/sns                          | SNS]]                                   |                                         |                                          |                                             |                |
| [[observability/observability]] | [[observability/cloudwatch/cloudwatch  | CloudWatch]] · [[observability/cloudtrail/cloudtrail | CloudTrail]]                            |                                         |                                          |                                             |                |
|                                 |                                        |                                                      |                                         |                                         |                                          |                                             |                |

> 이 문서에서는 위 서비스들의 상세 설명은 반복하지 않고, 한 줄 설명 + 사용 사례 + 기존 문서 링크만 제공한다. 위 표에 없는 AWS 서비스는 이 문서에서 보완한다.

---

## 3. 읽는 방법

각 서비스는 다음 기준으로 정리한다.

| 항목 | 설명 |
| --- | --- |
| 서비스명 | AWS 서비스 이름 |
| 설명 | 서비스가 해결하는 문제 |
| 주 사용 사례 | 언제 사용하는지 |
| 중요도 | 백엔드/DevOps 기준 중요도 |
| 기존 문서 | 기존 문서가 있으면 링크, 없으면 `-` |

중요도 기준:

| 중요도 | 의미 |
| --- | --- |
| ★★★ | 실무에서 자주 사용하거나 필수로 알아야 하는 서비스 |
| ★★☆ | 상황에 따라 자주 사용하는 서비스 |
| ★☆☆ | 특정 도메인이나 특수 목적에서 사용하는 서비스 |

---

## 4. Compute

가상 서버 / 컨테이너 / 함수 등 워크로드 실행 자원.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon EC2 | 가상 머신 (VM) | 백엔드 서버, 배치 서버, 레거시 시스템 운영 | ★★★ | [[compute/ec2/ec2]] |
| AWS Lambda | Serverless 함수 실행 | 이벤트 처리, API 백엔드, 짧은 작업 | ★★★ | [[compute/lambda/lambda]] |
| AWS Elastic Beanstalk | 응용 코드 업로드만으로 EC2/ALB/Auto Scaling 자동 프로비저닝 | 인프라 설정 없이 빠른 배포 (PaaS 성격) | ★☆☆ | [[compute/elastic-beanstalk/elastic-beanstalk]] |
| AWS Batch | 대량 배치 작업 스케줄링 + 컴퓨팅 자동 확장 | HPC, ML 학습 전처리, 야간 ETL | ★☆☆ | [[compute/batch/batch]] |
| AWS App Runner | 컨테이너 이미지나 Git 저장소를 받아 자동 빌드/배포/스케일 | 작은 웹서비스 빠른 배포 (Fargate 위의 PaaS) | ★☆☆ | [[compute/app-runner/app-runner]] |
| Amazon Lightsail | EC2/RDS/LB 를 정액제로 묶은 단순화 패키지 | 개인 프로젝트, 소규모 사이트 | ★☆☆ | [[compute/lightsail/lightsail]] |
| AWS Outposts | 온프레미스 환경에 AWS 인프라를 그대로 설치 | 데이터 주권/저지연 요구, 하이브리드 | ★☆☆ | [[compute/outposts/outposts]] |
| AWS Local Zones | 대도시 가까이 둔 미니 region | 5G/게임/실시간 응용의 저지연 | ★☆☆ | [[compute/local-zones/local-zones]] |
| AWS Wavelength | 통신사 5G 망 안에 AWS 컴퓨팅 배치 | 모바일 엣지 컴퓨팅 | ★☆☆ | [[compute/wavelength/wavelength]] |

---

## 5. Containers

컨테이너 오케스트레이션 + 이미지 저장.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon ECS | AWS native 컨테이너 오케스트레이션 | AWS 종속 OK, 빠르게 컨테이너 운영 | ★★★ | [[compute/ecs/ecs]] |
| Amazon EKS | 매니지드 Kubernetes | 표준 K8s 필요, 멀티클라우드 이식성 | ★★★ | [[compute/eks/eks]] |
| AWS Fargate | ECS/EKS 의 서버리스 실행 모드 (노드 관리 없음) | 컨테이너만 띄우고 인프라는 잊기 | ★★★ | [[compute/ecs/ecs]] (Fargate launch type 부분 참고) |
| Amazon ECR | 컨테이너 이미지 레지스트리 (Private/Public) | Docker 이미지 저장 + IAM 으로 권한 | ★★★ | [[compute/ecr/ecr]] |
| AWS App Mesh | 서비스 메시 (Envoy 기반 트래픽 관리) | 마이크로서비스 간 트래픽 라우팅/모니터링 | ★☆☆ | [[compute/app-mesh/app-mesh]] |
| AWS Copilot | ECS/Fargate/App Runner 배포 CLI | 컨테이너 배포 워크플로우 자동화 | ★☆☆ | [[compute/copilot/copilot]] |

---

## 6. Storage

객체 / 블록 / 파일 / 백업 / 데이터 이동.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon S3 | 객체 스토리지 | 이미지/PDF/정적 파일 저장, 데이터 레이크 | ★★★ | [[storage/s3/s3]] |
| Amazon EBS | EC2 의 블록 디스크 | OS 디스크, DB 데이터 파일 | ★★★ | [[storage/ebs/ebs]] |
| Amazon EFS | NFS 기반 파일 공유 (Linux) | 여러 EC2 가 동시에 마운트 | ★★☆ | [[storage/efs/efs]] |
| Amazon FSx | Windows / Lustre / NetApp ONTAP / OpenZFS 파일 시스템 | Windows 파일 공유, HPC 병렬 스토리지 | ★☆☆ | [[storage/fsx/fsx]] |
| Amazon S3 Glacier | 장기 보관용 저비용 객체 스토리지 (Glacier / Deep Archive) | 법적 보관, 백업 아카이브 | ★★☆ | [[storage/s3/s3]] (storage class 부분 참고) |
| AWS Backup | EBS / RDS / DynamoDB 등 다양한 자원의 통합 백업 | 백업 정책 일원화, 규정 준수 | ★★☆ | [[storage/backup/backup]] |
| AWS Storage Gateway | 온프레미스 ↔ AWS 스토리지 게이트웨이 (File/Volume/Tape) | 온프레 환경에서 AWS 스토리지를 로컬처럼 | ★☆☆ | [[storage/storage-gateway/storage-gateway]] |
| AWS DataSync | 온프레 / 다른 클라우드 ↔ AWS 데이터 이전 | 대용량 데이터 마이그레이션 | ★☆☆ | [[storage/datasync/datasync]] |

---

## 7. Database

관계형 / NoSQL / 캐시 / 데이터 웨어하우스.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon RDS | 매니지드 RDB (Postgres / MySQL / MariaDB / Oracle / SQL Server) | 일반 OLTP, 이식성 중요 | ★★★ | [[database/rds/rds]] |
| Amazon Aurora | AWS 자체 분산 RDB (Postgres / MySQL 호환) | 거대 워크로드, 빠른 read replica | ★★★ | [[database/aurora/aurora]] |
| Amazon DynamoDB | 매니지드 KV / Document NoSQL | 무한 확장 필요, single-digit ms latency | ★★★ | [[database/dynamodb/dynamodb]] |
| Amazon ElastiCache | 매니지드 Redis / Memcached | 캐시, 세션, 분산 락 | ★★★ | [[database/elasticache/elasticache]] |
| Amazon MemoryDB for Redis | Redis 호환 + 멀티 AZ 영속성 | 캐시 겸 primary store | ★☆☆ | [[database/memorydb/memorydb]] |
| Amazon Redshift | 매니지드 데이터 웨어하우스 (컬럼 기반) | 대규모 분석 쿼리, BI | ★★☆ | [[database/redshift/redshift]] |
| Amazon DocumentDB | MongoDB 호환 NoSQL | 기존 MongoDB 워크로드 이전 | ★☆☆ | [[database/documentdb/documentdb]] |
| Amazon Neptune | 그래프 DB (Property Graph / RDF) | 추천, 사기 탐지, 지식 그래프 | ★☆☆ | [[database/neptune/neptune]] |
| Amazon Keyspaces | Cassandra 호환 (CQL) | 기존 Cassandra 워크로드 매니지드로 | ★☆☆ | [[database/keyspaces/keyspaces]] |
| Amazon Timestream | 시계열 DB | IoT, 메트릭, 모니터링 데이터 | ★☆☆ | [[database/timestream/timestream]] |
| Amazon QLDB | 변경 불가 원장 (ledger) DB | 감사 로그 (2025-07 종료) | ★☆☆ | [[database/qldb/qldb]] |
| AWS Database Migration Service (DMS) | DB → DB 마이그레이션 (다른 엔진 가능) | 온프레/타 클라우드 DB → AWS 이전 | ★★☆ | [[database/dms/dms]] |

---

## 8. Networking & Content Delivery

VPC / LB / DNS / CDN / 사설 연결.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon VPC | 가상 사설 네트워크 | 모든 AWS 자원의 네트워크 base | ★★★ | [[network/vpc/vpc]] |
| Elastic Load Balancing (ALB / NLB / GLB) | L7 / L4 / L3 로드밸런서 | HTTP/HTTPS (ALB), TCP/UDP 고성능 (NLB), 보안 어플라이언스 체이닝 (GLB) | ★★★ | [[network/alb/alb]] · [[network/nlb/nlb]] |
| Amazon Route 53 | DNS + 헬스체크 + 라우팅 정책 | 도메인 관리, 지역별 트래픽 분산 | ★★★ | [[network/route53/route53]] |
| Amazon CloudFront | CDN (전 세계 edge cache) | 정적 파일 캐싱, 글로벌 콘텐츠 전송 | ★★★ | [[network/cloudfront/cloudfront]] |
| Amazon API Gateway | HTTP / REST / WebSocket API 게이트웨이 | Lambda 백엔드 노출, API 인증/throttling | ★★★ | [[network/api-gateway/api-gateway]] |
| AWS Direct Connect | 데이터센터 ↔ AWS 전용선 | 대역폭/지연 보장이 필요한 하이브리드 | ★☆☆ | [[network/direct-connect/direct-connect]] |
| AWS Global Accelerator | Anycast 기반 글로벌 트래픽 가속 | 비-HTTP 응용의 글로벌 latency 감소 | ★☆☆ | [[network/global-accelerator/global-accelerator]] |
| AWS Transit Gateway | 여러 VPC / 온프레를 허브로 묶는 라우터 | 멀티 VPC 토폴로지 단순화 | ★★☆ | [[network/transit-gateway/transit-gateway]] |
| AWS PrivateLink | 다른 VPC / 서비스에 사설 endpoint 로 접근 | NAT 안 거치고 AWS / SaaS 서비스 접근 | ★★☆ | [[network/privatelink/privatelink]] |
| AWS Cloud Map | 서비스 디스커버리 (DNS / API 기반) | 마이크로서비스 endpoint 등록/조회 | ★☆☆ | [[network/cloud-map/cloud-map]] |
| AWS VPN | Site-to-Site / Client VPN | 온프레 ↔ VPC 안전 연결 | ★☆☆ | [[network/vpn/vpn]] |

---

## 9. Security, Identity & Compliance

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS IAM | 인증 + 인가의 토대 | 모든 AWS 리소스 접근 제어 | ★★★ | [[iam/iam]] |
| AWS IAM Identity Center (구 SSO) | 여러 account / role 통합 로그인 | 조직 단위 SSO, Okta/Entra 연동 | ★★★ | [[iam/identity-center/identity-center]] |
| Amazon Cognito | 응용 사용자 인증 (User Pool + Identity Pool) | 모바일/웹 응용의 로그인/OAuth | ★★☆ | [[security/cognito/cognito]] |
| AWS KMS | 매니지드 암호화 키 (CMK) | S3/EBS/RDS 암호화 키 관리 | ★★★ | [[security/kms/kms]] |
| AWS Secrets Manager | DB password / API token 보관 + 자동 rotation | 응용 secret 관리, RDS 비밀번호 회전 | ★★★ | [[security/secrets-manager/secrets-manager]] |
| AWS Systems Manager Parameter Store | 설정값 / secret 저장 (KMS 통합) | secret 가벼운 대안 (회전 없음, 무료 tier) | ★★★ | [[security/parameter-store/parameter-store]] |
| AWS Certificate Manager (ACM) | TLS 인증서 무료 발급 + 자동 갱신 | ALB / CloudFront 의 HTTPS | ★★★ | [[security/acm/acm]] |
| AWS WAF | Web Application Firewall (L7 룰) | SQL injection / XSS / rate limit | ★★☆ | [[security/waf/waf]] |
| AWS Shield | DDoS 방어 (Standard 무료, Advanced 유료) | L3/L4/L7 DDoS 완화 | ★★☆ | [[security/shield/shield]] |
| Amazon GuardDuty | 계정 이상 행동 / 멀웨어 탐지 | 의심 IAM 사용, 침해 흔적 감지 | ★★☆ | [[security/guardduty/guardduty]] |
| Amazon Inspector | EC2 / Lambda / 컨테이너 이미지 취약점 스캔 | CVE 패치 우선순위, 컨테이너 보안 | ★★☆ | [[security/inspector/inspector]] |
| AWS Security Hub | 여러 보안 서비스의 findings 통합 | 보안 자세 (posture) 대시보드 | ★★☆ | [[security/security-hub/security-hub]] |
| Amazon Macie | S3 안의 민감 데이터 (PII / 신용카드) 자동 검출 | 개인정보 컴플라이언스 | ★☆☆ | [[security/macie/macie]] |
| AWS CloudHSM | 전용 HSM (FIPS 140-2 Level 3) | 규제상 키를 KMS 보다 강한 격리로 보관 | ★☆☆ | [[security/cloudhsm/cloudhsm]] |
| AWS Directory Service | 매니지드 Active Directory | Windows 워크로드, AD 연동 | ★☆☆ | [[security/directory-service/directory-service]] |
| AWS Artifact | AWS 컴플라이언스 보고서 (SOC / ISO 등) 다운로드 | 감사 대응 | ★☆☆ | [[security/artifact/artifact]] |
| AWS Firewall Manager | Organizations 단위 WAF / Shield / SG 룰 일괄 관리 | 멀티 account 보안 정책 | ★☆☆ | [[security/firewall-manager/firewall-manager]] |

---

## 10. Management & Governance

운영 / 감사 / 거버넌스 / 멀티 account.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon CloudWatch | Metric / Log / Alarm / Dashboard | 모든 AWS 자원 모니터링 | ★★★ | [[observability/cloudwatch/cloudwatch]] |
| AWS CloudTrail | 모든 AWS API 호출 audit log | 감사, 침해 조사, 변경 추적 | ★★★ | [[observability/cloudtrail/cloudtrail]] |
| AWS Config | 자원 설정 변경 이력 + 규정 준수 룰 평가 | 컴플라이언스, drift 감지 | ★★☆ | [[management/config/config]] |
| AWS Organizations | 여러 AWS account 를 조직 구조로 관리 + SCP | multi-account 거버넌스 | ★★★ | [[management/organizations/organizations]] |
| AWS Control Tower | Organizations + 보안/감사 베스트 프랙티스 자동화 | 신규 account 표준화된 landing zone | ★★☆ | [[management/control-tower/control-tower]] |
| AWS Systems Manager (SSM) | Run Command / Session Manager / Patch Manager / Parameter Store | EC2 운영 (SSH 없이 접속, 패치, 설정 배포) | ★★★ | [[management/ssm/ssm]] |
| AWS Trusted Advisor | 비용 / 성능 / 보안 / 한도 자동 점검 | 운영 헬스체크 | ★★☆ | [[management/trusted-advisor/trusted-advisor]] |
| AWS Health Dashboard | AWS 서비스 장애 / 이벤트 알림 | 장애 대응 | ★★☆ | [[management/health/health]] |
| AWS Service Catalog | 승인된 IaC 템플릿 / 제품 카탈로그 | 사용자가 표준 자원만 셀프 프로비저닝 | ★☆☆ | [[management/service-catalog/service-catalog]] |
| AWS License Manager | 소프트웨어 라이선스 추적 | Windows / Oracle 라이선스 관리 | ★☆☆ | [[management/license-manager/license-manager]] |
| AWS Well-Architected Tool | 6 pillar 점검 워크북 | 아키텍처 리뷰, 개선 액션 도출 | ★☆☆ | [[management/well-architected-tool/well-architected-tool]] |
| AWS Resource Groups / Tag Editor | 태그 기반 자원 그룹화 | 비용/운영 분류 | ★★☆ | [[management/resource-groups/resource-groups]] |
| AWS Resource Access Manager (RAM) | 자원 (Subnet / TGW 등) cross-account 공유 | 멀티 account 자원 공유 | ★☆☆ | [[management/ram/ram]] |

---

## 11. Developer Tools

CI/CD / IaC / 디버그 / 코드 저장소.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS CodeCommit | 매니지드 Git 저장소 | 레거시 AWS Git (신규는 GitHub 권장, 신규 가입 닫힘) | ★☆☆ | [[dev-tools/codecommit/codecommit]] |
| AWS CodeBuild | 매니지드 빌드 서비스 | CI 빌드 잡 실행 | ★★☆ | [[dev-tools/codebuild/codebuild]] |
| AWS CodeDeploy | EC2 / ECS / Lambda 배포 자동화 | Blue/Green, Canary 배포 | ★★☆ | [[dev-tools/codedeploy/codedeploy]] |
| AWS CodePipeline | CI/CD 파이프라인 오케스트레이션 | Source → Build → Deploy 묶기 | ★★☆ | [[dev-tools/codepipeline/codepipeline]] |
| AWS CodeArtifact | npm / Maven / PyPI / NuGet 패키지 저장소 | 사내 패키지 / 의존성 캐시 | ★☆☆ | [[dev-tools/codeartifact/codeartifact]] |
| AWS CloudFormation | AWS 공식 IaC (JSON / YAML) | 모든 AWS 자원 선언적 정의 | ★★★ | [[dev-tools/cloudformation/cloudformation]] |
| AWS CDK | TypeScript / Python / Java 로 IaC 작성 → CloudFormation 으로 변환 | 코드로 인프라 (프로그래밍 가능한 IaC) | ★★★ | [[dev-tools/cdk/cdk]] |
| AWS SAM | Lambda + API Gateway + DynamoDB 등 serverless 전용 IaC | Lambda 응용 배포 | ★★☆ | [[dev-tools/sam/sam]] |
| AWS X-Ray | 분산 추적 (서비스 간 호출 시각화) | 마이크로서비스 latency 디버그 | ★★☆ | [[dev-tools/x-ray/x-ray]] |
| AWS Cloud9 | 브라우저 IDE | 페어 프로그래밍, EC2 위 개발 (신규 가입 닫힘) | ★☆☆ | [[dev-tools/cloud9/cloud9]] |
| Amazon CodeCatalyst | 통합 SDLC (이슈/리포지토리/CI/CD/환경) | AWS 친화적 개발자 포털 | ★☆☆ | [[dev-tools/codecatalyst/codecatalyst]] |
| Amazon Q Developer (구 CodeWhisperer) | LLM 기반 코드 자동완성 + Q&A | IDE / 콘솔에서 AI 도우미 | ★★☆ | [[dev-tools/q-developer/q-developer]] |

---

## 12. Application Integration

비동기 메시징 / 이벤트 / 워크플로우.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon SQS | Queue (1:1, at-least-once) | 작은 작업 큐, 백프레셔 | ★★★ | [[messaging/sqs/sqs]] |
| Amazon SNS | Pub/Sub (1:N fan-out) | 이벤트 broadcast, push 알림 | ★★★ | [[messaging/sns/sns]] |
| Amazon EventBridge | 이벤트 버스 + 룰 기반 라우팅 (SaaS 통합 포함) | 서비스 간 이벤트 라우팅, schedule 대체 | ★★★ | [[app-integration/eventbridge/eventbridge]] |
| AWS Step Functions | 상태 머신 기반 워크플로우 (state machine) | 복잡한 비동기 작업 흐름, Lambda 오케스트레이션 | ★★☆ | [[app-integration/step-functions/step-functions]] |
| Amazon MQ | 매니지드 RabbitMQ / ActiveMQ | 기존 AMQP/JMS 응용을 옮길 때 | ★☆☆ | [[app-integration/mq/mq]] |
| AWS AppSync | 매니지드 GraphQL 서버 (subscription 지원) | 모바일/웹 GraphQL 백엔드 | ★☆☆ | [[app-integration/appsync/appsync]] |
| Amazon API Gateway | HTTP / REST / WebSocket API 게이트웨이 | (Networking 카테고리에도 등장, integration 측면) | ★★★ | [[network/api-gateway/api-gateway]] |
| Amazon Managed Workflows for Apache Airflow (MWAA) | 매니지드 Airflow | 데이터 파이프라인 오케스트레이션 | ★☆☆ | [[app-integration/mwaa/mwaa]] |

---

## 13. Analytics

데이터 처리 / 분석 / 스트리밍.

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon Athena | S3 위 SQL 쿼리 (Presto / Trino 기반) | 데이터 레이크 ad-hoc 분석 | ★★☆ | [[analytics/athena/athena]] |
| AWS Glue | 매니지드 ETL + 데이터 카탈로그 | 스키마 발견, 배치 ETL | ★★☆ | [[analytics/glue/glue]] |
| Amazon EMR | 매니지드 Hadoop / Spark / Hive / Presto | 빅데이터 분산 처리 | ★☆☆ | [[analytics/emr/emr]] |
| Amazon Kinesis Data Streams | 실시간 스트림 (shard 기반) | 로그/이벤트 스트림 처리 | ★★☆ | [[analytics/kinesis/kinesis]] |
| Amazon Data Firehose (구 Kinesis Firehose) | 스트림 → S3 / Redshift / OpenSearch 적재 | 스트림 손쉽게 저장소로 | ★★☆ | [[analytics/firehose/firehose]] |
| Amazon OpenSearch Service | 매니지드 OpenSearch / Elasticsearch | 로그 검색, 전문 검색 엔진 | ★★☆ | [[analytics/opensearch/opensearch]] |
| Amazon QuickSight | BI 대시보드 | 사내 대시보드, 임원 보고 | ★☆☆ | [[analytics/quicksight/quicksight]] |
| AWS Lake Formation | 데이터 레이크 권한 / 카탈로그 관리 | S3 데이터 레이크 거버넌스 | ★☆☆ | [[analytics/lake-formation/lake-formation]] |
| Amazon MSK | 매니지드 Apache Kafka | Kafka 호환 스트림, 거대 throughput | ★★☆ | [[analytics/msk/msk]] |
| AWS DataZone | 데이터 카탈로그 + 공유 / 거버넌스 포털 | 조직 단위 데이터 검색 | ★☆☆ | [[analytics/datazone/datazone]] |
| Amazon Managed Service for Apache Flink (구 Kinesis Data Analytics) | 매니지드 Flink (SQL / Java) | 실시간 스트림 처리 | ★☆☆ | [[analytics/flink/flink]] |

---

## 14. Machine Learning & AI

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon Bedrock | 파운데이션 모델 (Claude / Llama / Titan 등) API | LLM 응용 (RAG, 챗봇, 요약) | ★★★ | [[ml/bedrock/bedrock]] |
| Amazon SageMaker | 매니지드 ML 학습 / 배포 플랫폼 | 자체 모델 학습, MLOps | ★★☆ | [[ml/sagemaker/sagemaker]] |
| Amazon Comprehend | NLP (감정 / 개체 / 분류) | 텍스트 분류, 감정 분석 | ★☆☆ | [[ml/comprehend/comprehend]] |
| Amazon Textract | OCR + 표/양식 추출 | PDF/이미지에서 구조화 데이터 추출 | ★★☆ | [[ml/textract/textract]] |
| Amazon Rekognition | 이미지/영상 인식 (얼굴, 객체, 모더레이션) | 콘텐츠 모더레이션, 신원 확인 | ★☆☆ | [[ml/rekognition/rekognition]] |
| Amazon Transcribe | 음성 → 텍스트 (STT) | 콜센터 녹취, 자막 생성 | ★☆☆ | [[ml/transcribe/transcribe]] |
| Amazon Polly | 텍스트 → 음성 (TTS) | IVR, 안내 음성 | ★☆☆ | [[ml/polly/polly]] |
| Amazon Translate | 기계 번역 | 다국어 콘텐츠 자동 번역 | ★☆☆ | [[ml/translate/translate]] |
| Amazon Lex | 챗봇 / 음성 봇 (Alexa 엔진) | IVR, 응용 챗봇 | ★☆☆ | [[ml/lex/lex]] |
| Amazon Kendra | 기업용 의미 검색 | 사내 문서 검색 | ★☆☆ | [[ml/kendra/kendra]] |
| Amazon Personalize | 추천 시스템 | 콘텐츠 / 상품 추천 | ★☆☆ | [[ml/personalize/personalize]] |
| Amazon Forecast | 시계열 예측 | 수요 예측, 매출 예측 (신규 가입 닫힘) | ★☆☆ | [[ml/forecast/forecast]] |
| Amazon Fraud Detector | 사기 거래 탐지 ML | 결제 사기, 가짜 계정 탐지 | ★☆☆ | [[ml/fraud-detector/fraud-detector]] |

---

## 15. Migration & Transfer

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS Migration Hub | 마이그레이션 진행 상황 통합 대시보드 | 대규모 이전 트래킹 | ★☆☆ | [[migration/migration-hub/migration-hub]] |
| AWS Application Migration Service (MGN) | VM / 물리 서버 → EC2 lift-and-shift | 데이터센터 → AWS rehost | ★☆☆ | [[migration/mgn/mgn]] |
| AWS Database Migration Service (DMS) | DB → DB 마이그레이션 | (Database 카테고리에도 등장) | ★★☆ | [[database/dms/dms]] |
| AWS DataSync | 대용량 파일 데이터 이전 | (Storage 카테고리에도 등장) | ★☆☆ | [[storage/datasync/datasync]] |
| AWS Transfer Family | SFTP / FTPS / FTP / AS2 → S3/EFS | 외부 파트너의 SFTP 업로드 수용 | ★☆☆ | [[migration/transfer-family/transfer-family]] |
| AWS Snowball / Snowmobile | 물리 디바이스로 페타바이트급 데이터 운반 | 네트워크로 못 옮기는 대용량 이전 | ★☆☆ | [[migration/snowball/snowball]] |
| AWS Application Discovery Service | 온프레 인벤토리 자동 수집 | 마이그레이션 사전 조사 | ★☆☆ | [[migration/application-discovery/application-discovery]] |

---

## 16. Frontend Web & Mobile

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS Amplify | 프론트엔드 호스팅 + 백엔드 (인증/API/스토리지) 풀스택 | SPA / SSR 빠른 배포 | ★★☆ | [[frontend/amplify/amplify]] |
| AWS AppSync | 매니지드 GraphQL | (Application Integration 에도 등장) | ★☆☆ | [[app-integration/appsync/appsync]] |
| Amazon Cognito | 응용 사용자 인증 | (Security 카테고리에도 등장) | ★★☆ | [[security/cognito/cognito]] |
| Amazon Pinpoint | 모바일 푸시 / 이메일 / SMS 캠페인 | 마케팅 알림, 사용자 세그먼트 푸시 | ★☆☆ | [[frontend/pinpoint/pinpoint]] |
| AWS Device Farm | 실제 모바일 디바이스 테스트 팜 | iOS/Android 다양한 디바이스 자동 테스트 | ★☆☆ | [[frontend/device-farm/device-farm]] |
| Amazon Location Service | 지도 / 지오코딩 / 경로 / 지오펜스 | 위치 기반 응용 (Google Maps 대안) | ★☆☆ | [[frontend/location-service/location-service]] |

---

## 17. Business Applications

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon WorkSpaces | 매니지드 가상 데스크톱 (DaaS) | 원격 근무자에게 표준 데스크톱 제공 | ★☆☆ | [[business-apps/workspaces/workspaces]] |
| Amazon WorkDocs | 매니지드 문서 공유 (서비스 종료 2025-04) | 레거시 문서 협업 | ★☆☆ | [[business-apps/workdocs/workdocs]] |
| Amazon WorkMail | 매니지드 이메일 / 캘린더 | 사내 이메일 서버 대체 | ★☆☆ | [[business-apps/workmail/workmail]] |
| Amazon Chime | 화상회의 / 메신저 | 사내 커뮤니케이션 (Zoom 대안) | ★☆☆ | [[business-apps/chime/chime]] |
| Amazon Connect | 매니지드 콜센터 (옴니채널) | 클라우드 콜센터 운영 | ★☆☆ | [[business-apps/connect/connect]] |
| Amazon SES | 트랜잭션 / 마케팅 이메일 발송 | 회원가입 인증 메일, 비밀번호 재설정 | ★★★ | [[messaging/ses/ses]] |
| Amazon Pinpoint | 모바일/이메일/SMS 캠페인 | (Frontend 카테고리에도 등장) | ★☆☆ | [[frontend/pinpoint/pinpoint]] |

---

## 18. End User Computing

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon WorkSpaces | 가상 데스크톱 | (Business Applications 에도 등장) | ★☆☆ | [[business-apps/workspaces/workspaces]] |
| Amazon AppStream 2.0 | 데스크톱 응용 스트리밍 | Windows 응용을 브라우저로 제공 | ★☆☆ | [[business-apps/appstream/appstream]] |
| Amazon WorkSpaces Web | 브라우저 기반 격리된 웹 세션 | 보안 격리된 인터넷 접근 | ★☆☆ | [[business-apps/workspaces-web/workspaces-web]] |
| Amazon WorkSpaces Thin Client | WorkSpaces 전용 씬 클라이언트 하드웨어 | BYOD 대체용 표준 디바이스 | ★☆☆ | [[business-apps/thin-client/thin-client]] |

---

## 19. Internet of Things

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS IoT Core | MQTT 브로커 + 디바이스 레지스트리 | IoT 디바이스 클라우드 연결 | ★☆☆ | [[iot/iot-core/iot-core]] |
| AWS IoT Greengrass | 엣지 컴퓨팅 (디바이스에서 Lambda 실행) | 오프라인 동작, 엣지 추론 | ★☆☆ | [[iot/greengrass/greengrass]] |
| AWS IoT Device Management | 디바이스 프로비저닝 / OTA / 모니터링 | 대규모 디바이스 운영 | ★☆☆ | [[iot/device-management/device-management]] |
| AWS IoT Analytics | IoT 데이터 분석 파이프라인 | IoT 메트릭 시계열 분석 (2025-12 종료) | ★☆☆ | [[iot/iot-analytics/iot-analytics]] |
| AWS IoT SiteWise | 산업 장비 데이터 수집/모델링 | 공장 OPC UA / Modbus 수집 | ★☆☆ | [[iot/sitewise/sitewise]] |
| AWS IoT TwinMaker | 디지털 트윈 | 시설 / 장비의 3D 트윈 | ★☆☆ | [[iot/twinmaker/twinmaker]] |
| AWS IoT Events | 디바이스 상태 머신 | 임계값 기반 알림 / 액션 (2025-12 종료) | ★☆☆ | [[iot/events/events]] |

---

## 20. Media Services

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS Elemental MediaConvert | 파일 기반 트랜스코딩 | VOD 트랜스코딩 | ★☆☆ | [[media/mediaconvert/mediaconvert]] |
| AWS Elemental MediaLive | 라이브 인코딩 | 라이브 스트리밍 송출 | ★☆☆ | [[media/medialive/medialive]] |
| AWS Elemental MediaPackage | 라이브/VOD 패키징 (HLS/DASH/CMAF) | 멀티 디바이스 스트리밍 | ★☆☆ | [[media/mediapackage/mediapackage]] |
| AWS Elemental MediaStore | 저지연 미디어 객체 스토리지 (2025-11 종료) | 라이브 origin | ★☆☆ | [[media/mediastore/mediastore]] |
| AWS Elemental MediaTailor | 동적 광고 삽입 (SSAI) | 광고 기반 스트리밍 | ★☆☆ | [[media/mediatailor/mediatailor]] |
| Amazon IVS | 매니지드 저지연 라이브 스트리밍 | 인터랙티브 라이브 (방송, 게임) | ★☆☆ | [[media/ivs/ivs]] |

---

## 21. Game Tech

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| Amazon GameLift | 매니지드 게임 서버 호스팅 + 매치메이킹 | 멀티플레이어 세션 서버 | ★☆☆ | [[game/gamelift/gamelift]] |
| Open 3D Engine (O3DE) | 오픈소스 게임 엔진 (Linux Foundation) | AAA 게임 개발 (사용 빈도 낮음) | ★☆☆ | [[game/o3de/o3de]] |
| AWS GameKit | Unreal / Unity 용 백엔드 키트 | 인증/저장/리더보드 백엔드 빠르게 | ★☆☆ | [[game/gamekit/gamekit]] |

---

## 22. Robotics / Satellite / Quantum

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS RoboMaker | 로봇 시뮬레이션 + 플릿 관리 (2025-09 종료) | ROS 기반 로봇 시뮬레이션 | ★☆☆ | [[specialty/robomaker/robomaker]] |
| AWS Ground Station | 위성 안테나 as a Service | 위성 데이터 수신 | ★☆☆ | [[specialty/ground-station/ground-station]] |
| Amazon Braket | 양자 컴퓨팅 시뮬레이션 / 실제 양자 하드웨어 액세스 | 양자 알고리즘 실험 | ★☆☆ | [[specialty/braket/braket]] |

---

## 23. Cost Management

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS Billing | 청구 / 결제 콘솔 | 청구서 확인, 결제 수단 관리 | ★★★ | [[cost/billing/billing]] |
| AWS Cost Explorer | 비용 시계열 분석 + 예측 | 비용 추세, 서비스별 분포 | ★★★ | [[cost/cost-explorer/cost-explorer]] |
| AWS Budgets | 예산 + 알림 (cost / usage / RI) | 월 예산 초과 알림 | ★★★ | [[cost/budgets/budgets]] |
| AWS Cost and Usage Report (CUR) | 시간 단위 비용 raw 데이터 → S3 | Athena/QuickSight 로 정밀 분석 | ★★☆ | [[cost/cur/cur]] |
| AWS Savings Plans | 시간당 약정 (1년 / 3년, 컴퓨팅) | EC2 / Fargate / Lambda 비용 30-70% 절감 | ★★★ | [[cost/savings-plans/savings-plans]] |
| AWS Compute Optimizer | EC2 / Lambda / EBS rightsizing 추천 | over/underprovisioning 자동 발견 | ★★☆ | [[cost/compute-optimizer/compute-optimizer]] |

---

## 24. Customer Enablement / Miscellaneous

| 서비스명 | 설명 | 주 사용 사례 | 중요도 | 기존 문서 |
| --- | --- | --- | --- | --- |
| AWS Support | 기술 지원 플랜 (Basic / Developer / Business / Enterprise) | 장애/구성 질문 티켓 | ★★☆ | [[customer-enablement/support/support]] |
| AWS Professional Services | 컨설팅 서비스 | 대형 마이그레이션 컨설팅 | ★☆☆ | [[customer-enablement/professional-services/professional-services]] |
| AWS IQ | AWS 인증 전문가 매칭 마켓플레이스 | 단발성 컨설팅 | ★☆☆ | [[customer-enablement/iq/iq]] |
| AWS Marketplace | 3rd party 소프트웨어 카탈로그 | 라이선스 / SaaS / AMI 구매 | ★★☆ | [[customer-enablement/marketplace/marketplace]] |
| AWS Activate | 스타트업 크레딧 프로그램 | 초기 스타트업 비용 절감 | ★☆☆ | [[customer-enablement/activate/activate]] |

---

## 25. 백엔드/DevOps 우선 학습 서비스

처음 AWS 를 시작하는 백엔드 / DevOps 개발자가 **순서대로** 익혀야 할 핵심 서비스. 위에서 ★★★ 인 서비스 중에서 추렸다.

| 우선순위 | 서비스 | 이유 | 기존 문서 |
| --- | --- | --- | --- |
| 1 | IAM | 모든 AWS 리소스 접근 제어의 기본 | [[iam/iam]] |
| 2 | VPC | 네트워크 설계의 기본 (Subnet / SG / NACL / NAT / IGW) | [[network/vpc/vpc]] |
| 3 | EC2 | 가상 서버 운영의 기본 (AMI / instance type / EBS / userdata) | [[compute/ec2/ec2]] |
| 4 | S3 | 파일 / 객체 저장의 기본 (버킷 / 권한 / lifecycle) | [[storage/s3/s3]] |
| 5 | RDS | 관계형 DB 운영의 기본 (Multi-AZ / backup / parameter group) | [[database/rds/rds]] |
| 6 | CloudWatch | 모니터링의 기본 (Metric / Log / Alarm / Dashboard) | [[observability/cloudwatch/cloudwatch]] |
| 7 | CloudTrail | 감사 / 보안 추적의 기본 | [[observability/cloudtrail/cloudtrail]] |
| 8 | Route 53 | DNS / 헬스체크 / 라우팅 | [[network/route53/route53]] |
| 9 | CloudFront | CDN 의 기본 + S3 정적 사이트 | [[network/cloudfront/cloudfront]] |
| 10 | ACM | 무료 TLS 인증서 — HTTPS 의 기본 | [[security/acm/acm]] |
| 11 | ALB / ELB | L7 로드밸런서 + listener / target group | [[network/alb/alb]] |
| 12 | ECR | 컨테이너 이미지 저장의 기본 | [[compute/ecr/ecr]] |
| 13 | ECS | AWS 컨테이너 오케스트레이션 | [[compute/ecs/ecs]] |
| 14 | Fargate | ECS / EKS 의 서버리스 실행 | [[compute/ecs/ecs]] |
| 15 | Lambda | Serverless 함수 | [[compute/lambda/lambda]] |
| 16 | Secrets Manager | DB 비밀번호 / API 키 보관 + rotation | [[security/secrets-manager/secrets-manager]] |
| 17 | Systems Manager (SSM) | EC2 운영 (Session Manager / Patch / Parameter Store) | [[management/ssm/ssm]] |
| 18 | SQS | 비동기 작업 큐 | [[messaging/sqs/sqs]] |
| 19 | SNS | Pub/Sub fan-out | [[messaging/sns/sns]] |
| 20 | EventBridge | 이벤트 라우팅 + cron 대체 | [[app-integration/eventbridge/eventbridge]] |
| 21 | Budgets | 예산 알림 (비용 사고 방지 첫 단계) | [[cost/budgets/budgets]] |
| 22 | Cost Explorer | 비용 분석 | [[cost/cost-explorer/cost-explorer]] |

---

## 26. 백엔드/DevOps 표준 서비스 조합

자주 등장하는 실무 조합. "이 문제는 이 조합" 매칭용.

| 목적 | 서비스 조합 | 설명 |
| --- | --- | --- |
| 정적 프론트엔드 배포 | S3 + CloudFront + Route 53 + ACM | React / Vite 빌드 파일을 S3 에 올리고 CloudFront 로 전 세계 캐싱, Route 53 도메인 + ACM 인증서 |
| SSR / SPA 풀스택 빠르게 | Amplify Hosting + Cognito + AppSync | 초기 프로토타입 / 사내 도구 |
| 컨테이너 백엔드 배포 | ECR + ECS/Fargate + ALB + CloudWatch | Docker 이미지를 ECR 에, ECS/Fargate 로 실행, ALB 로 트래픽, CloudWatch 로 모니터링 |
| 표준 K8s 컨테이너 배포 | EKS + ECR + ALB Ingress + IRSA + Karpenter | 멀티클라우드 이식성 필요, K8s native 도구 사용 |
| 관계형 DB 운영 | RDS + Secrets Manager + CloudWatch + Parameter Store | RDS 매니지드, 비밀번호 rotation, slow query 알림 |
| KV / 무한 확장 DB | DynamoDB + DAX + Streams + Lambda | latency 민감, 확장성 우선 |
| 비동기 작업 처리 | SQS + Lambda (또는 ECS Worker) + Dead Letter Queue | 오래 걸리는 작업 분리, 실패 트래픽 격리 |
| 이벤트 기반 아키텍처 | EventBridge + Lambda + SQS + Step Functions | 서비스 간 결합도를 낮춘 이벤트 처리 + 복잡 워크플로우 |
| 실시간 스트림 처리 | Kinesis Data Streams / MSK + Lambda / Flink + S3 (Firehose) | 로그 / 이벤트 스트림 수집 + 분석 + 적재 |
| 데이터 레이크 / 분석 | S3 + Glue + Athena + QuickSight | ad-hoc 분석, BI |
| 운영 감사 / 보안 | CloudTrail + Config + GuardDuty + Security Hub | 계정 활동 / 보안 이벤트 / 컴플라이언스 모니터링 |
| Multi-account 거버넌스 | Organizations + Control Tower + SCP + IAM Identity Center | 조직 단위 account 표준화 + SSO |
| CI/CD (AWS 자체) | CodeCommit (또는 GitHub) + CodeBuild + CodeDeploy + CodePipeline | AWS 안에서 완결 |
| CI/CD (GitHub Actions) | GitHub Actions + OIDC + IAM Role + ECR + ECS/EKS | 외부 SaaS + IAM Role 임시 자격 |
| IaC | CloudFormation 또는 CDK 또는 Terraform | 인프라 코드화. CDK = 프로그래밍, Terraform = 멀티클라우드 |
| LLM 응용 백엔드 | Bedrock + Lambda + DynamoDB + OpenSearch (RAG) + API Gateway | Claude 등 API + 벡터 검색 + 응용 백엔드 |
| 비용 관리 | Budgets + Cost Explorer + Compute Optimizer + Savings Plans + CUR | 예산 알림 + 비용 분석 + 약정 절감 + raw 데이터 |
| 트랜잭션 이메일 | SES + Lambda + SNS (bounce / complaint) | 회원가입 / 비밀번호 재설정 메일 |
| 사내 표준 데스크톱 | WorkSpaces + Directory Service + AppStream 2.0 | 원격 근무자 / 외주 표준 환경 |

---

## 27. 작성 / 갱신 규칙

이 인덱스 문서는 **얇게 유지**한다. 깊이는 카테고리 hub 와 서비스별 deep 노트에 위임한다.

- 새 서비스가 vault 에 deep 노트로 추가되면 → 본 문서의 해당 행 `기존 문서` 칸에 wikilink 추가.
- AWS 가 새 서비스를 출시하면 → 적절한 카테고리에 한 줄 추가. 깊이는 별도 deep 노트로.
- 서비스가 종료되거나 다른 서비스로 통합되면 → "사용 빈도 낮음" / "기존 문서 확인 필요" 마킹. 행은 삭제하지 않음 (탐색 흔적 보존).
- 본 문서 안에서 deep 설명을 길게 쓰지 않는다 — 길이가 늘어나면 deep 노트로 분리한다.

---

## 28. 관련

- [[cloud-aws|↑ AWS Cloud Hub]] — 카테고리 분류 / 시작 / 비용 함정
- [[../devops|↑ devops]]
- [[../iac/iac]] — IaC 도구 (CloudFormation / CDK / Terraform / Pulumi)
- [[../kubernetes/kubernetes]] — EKS 의 K8s 측면
- [[../cloud-gcp/cloud-gcp]] / [[../cloud-azure/cloud-azure]] / [[../cloud-oci/cloud-oci]] — 비교용
