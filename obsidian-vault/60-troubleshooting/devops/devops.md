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

| #  | 증상 / 에러                                                                  | 한 줄 원인                              | 핵심 해결                                                                  |
| -- | ------------------------------------------------------------------------ | ---------------------------------- | ---------------------------------------------------------------------- |
| 1  | **Docker build cache miss — 매번 전체 layer 재빌드**                            | COPY 순서가 자주 바뀌는 파일 먼저              | dependency 파일 (`package.json` / `pom.xml`) 먼저 COPY + install, 그 다음 source |
| 2  | **`docker buildx build` exec format error**                              | host (amd64) 와 target (arm64) arch 불일치 | `--platform linux/arm64` 명시 / multi-arch manifest                     |
| 3  | **ECR push 권한 — `denied: User is not authorized`**                       | IAM 가 `ecr:BatchCheckLayerAvailability` 등 부재 | `AmazonEC2ContainerRegistryPowerUser` policy / `aws ecr get-login-password` |
| 4  | **ECS Task — `TaskFailedToStart` / `CannotPullContainerError`**           | VPC 가 ECR 도달 못함 (private subnet without NAT/VPC Endpoint) | ECR VPC Endpoint (Interface) + S3 Gateway / NAT Gateway              |
| 5  | **K8s `CrashLoopBackOff`**                                               | 컨테이너 시작 실패 + 재시작 반복                 | `kubectl logs --previous` / liveness probe initial delay / startup probe |
| 6  | **K8s `OOMKilled`** (exit code 137)                                      | 컨테이너 memory limit 초과               | `kubectl top pod` / limits/requests 조정 / heap 비율 조정 (`-XX:MaxRAMPercentage`) |
| 7  | **K8s `ImagePullBackOff`**                                               | private registry 인증 부재 / image tag 없음 | `imagePullSecrets` 설정 / `kubectl describe pod`                         |
| 8  | **CodeDeploy rollback — Blue/Green deployment failed**                   | Target Group health check 실패        | health check path/port 검증 / grace period 확대 / 새 task definition 로그 확인 |
| 9  | **Terraform state lock — `Error acquiring the state lock`**              | 이전 plan/apply 가 비정상 종료              | `terraform force-unlock <lock-id>` (확인 후) / S3+DynamoDB backend 권장    |
| 10 | **GitHub Actions secret undefined** — `${{ secrets.X }}` 가 빈 문자열         | fork PR / environment 차이 / secret 이름 오타 | repo settings → Secrets / pull_request 대신 pull_request_target (주의) / environment 분리 |
| 11 | **Disk full — `no space left on device`**                                | log / Docker overlay / journald 누적 | `docker system prune -af` / `journalctl --vacuum-time=7d` / log rotation |
| 12 | **TLS cert renewal 실패 — Let's Encrypt rate limit**                       | 5/주 같은 도메인 발급 한도                   | staging endpoint 로 테스트 / wildcard / certbot --dry-run                  |
| 13 | **cron / EventBridge — TZ 불일치로 실행 안 됨**                                  | container TZ = UTC, cron 은 local 기준 작성 | `TZ=Asia/Seoul` env / cron 을 UTC 로 통일 / EventBridge `cron(0 0 * * ? *)` 검증 |
| 14 | **SSH — `Permissions 0644 for 'id_rsa' are too open`**                   | private key permission 너무 열림        | `chmod 600 ~/.ssh/id_rsa`                                              |
| 15 | **GitHub Actions cache miss — 매 빌드 의존성 재다운**                             | cache key 가 매번 다름 / hashFiles 순서   | `key: ${{ runner.os }}-deps-${{ hashFiles('**/lockfile') }}` + `restore-keys` |

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
