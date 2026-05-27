---
title: "60-troubleshooting/devops — DevOps 트러블 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-27
tags: [troubleshooting, devops, docker, k8s, ci, aws, hub]
home_hub: "60-troubleshooting"
---

# DevOps 트러블 카탈로그

> Docker / Kubernetes / GitHub Actions / AWS 배포에서 자주 만나는 대표 트러블.

**[[../troubleshooting|↑ 60-troubleshooting]]** · **[[../../20-areas/devops/devops|↑↑ 20-areas/devops]]**

---

## 카탈로그

| #  | 증상 / 에러                                                                  | 한 줄 원인                              | 핵심 해결                                                                  | 깊이 노트                                |
| -- | ------------------------------------------------------------------------ | ---------------------------------- | ---------------------------------------------------------------------- | ------------------------------------ |
| 1  | **Docker build cache miss**                                              | COPY 순서가 자주 바뀌는 파일 먼저              | deps 먼저 COPY / BuildKit cache mount / CI cache-from                  | [[docker-build-cache-miss]]          |
| 2  | **docker buildx exec format error**                                      | host arch ↔ target arch 불일치        | `--platform` 명시 / multi-arch buildx / runtimePlatform                 | [[docker-arch-mismatch]]             |
| 3  | **ECR push denied**                                                      | IAM 권한 / login token 만료            | ecr get-login-password / IAM policy / OIDC                            | [[ecr-push-denied]]                  |
| 4  | **ECS TaskFailedToStart / CannotPullContainerError**                     | private subnet without VPC Endpoint | VPC Endpoint (ECR/S3/Secrets) / execution role / runtimePlatform     | [[ecs-task-failed-to-start]]         |
| 5  | **K8s CrashLoopBackOff**                                                 | 컨테이너 시작 실패 + 재시작 반복                 | logs --previous / startup probe / init container                      | [[k8s-crashloopbackoff]]             |
| 6  | **K8s OOMKilled** (exit 137)                                             | memory limits 초과                    | limits 확대 / MaxRAMPercentage / VPA                                   | [[k8s-oomkilled]]                    |
| 7  | **K8s ImagePullBackOff**                                                 | private registry 인증 / network       | imagePullSecrets / IRSA / VPC Endpoint                                | [[k8s-imagepullbackoff]]             |
| 8  | **CodeDeploy Blue-Green rollback**                                       | Target Group health check 실패        | liveness/readiness 분리 / grace period / alarm 기반 rollback           | [[codedeploy-rollback]]              |
| 9  | **Terraform state lock**                                                 | 이전 plan/apply 가 비정상 종료              | force-unlock / DynamoDB delete / S3 versioning 복구                    | [[terraform-state-lock]]             |
| 10 | **GitHub Actions secret undefined**                                      | fork PR / environment 차이 / 오타     | env / OIDC 우선 / fork PR 은 secret 차단                                 | [[github-actions-secret-undefined]]  |
| 11 | **Disk full — no space left on device**                                  | log / Docker overlay / journald 누적 | docker prune / journald vacuum / logrotate / Docker log driver limit  | [[disk-full]]                        |
| 12 | **TLS cert renewal — Let's Encrypt rate limit**                          | 5/주 같은 도메인 발급 한도                   | certbot renew (재발급 X) / wildcard / cert-manager / ACM             | [[tls-cert-renewal]]                 |
| 13 | **cron / EventBridge TZ 불일치**                                            | container TZ = UTC, cron 한국 기준 작성 | container TZ 명시 / CRON_TZ / EventBridge UTC 변환 표                  | [[cron-timezone]]                    |
| 14 | **SSH permission too open**                                              | private key 권한 너무 열림              | chmod 600 / ssh-agent / SSM Session Manager (대체)                      | [[ssh-key-permission]]               |
| 15 | **GitHub Actions cache miss**                                            | cache key / hashFiles 잘못           | setup-* cache / restore-keys / hashFiles glob 정확                     | [[github-actions-cache-miss]]        |

## 자주 쓰는 진단 명령

```bash
# Docker
docker inspect <container>
docker stats
docker system df / prune

# K8s
kubectl describe pod <pod>
kubectl logs -f <pod> --previous
kubectl top pod / node
kubectl get events --sort-by='.lastTimestamp'

# AWS
aws ecs describe-tasks --cluster X --tasks Y
aws logs tail /aws/ecs/X --follow --since 30m
aws codedeploy get-deployment --deployment-id Z
```

## 관련

- [[../troubleshooting]]
- [[../../20-areas/devops/devops|20-areas/devops]]
- [[../network/network|network 트러블]] — VPC / SG / DNS 경계
