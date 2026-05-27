---
title: "ECS 02-2 — Image / CPU / Memory / PID / GPU 자원"
kind: knowledge
status: current
created_at: 2026-05-22T13:32:00+09:00
tags: [aws, ecs, task-definition, resources]
home_hub: ecs
---

# Task Definition — Image / 자원

> task 가 사용할 image 출처 (ECR / Docker Hub / GHCR) + cpu / memory / pids / ephemeral storage /
> GPU 할당. Fargate vs EC2 의 자원 모델 차이.

## 1. image 출처

```json
{
  "image": "123.dkr.ecr.ap-northeast-2.amazonaws.com/bkurs-be:v1.2.3"
}
```

지원되는 registry:
| Registry | URL 패턴 |
| --- | --- |
| ECR | `{account}.dkr.ecr.{region}.amazonaws.com/{repo}:{tag}` |
| ECR Public | `public.ecr.aws/{alias}/{repo}:{tag}` |
| Docker Hub | `library/nginx:alpine` 또는 `myuser/myapp:v1` |
| GHCR | `ghcr.io/{org}/{repo}:{tag}` |
| GCR | `gcr.io/{project}/{repo}:{tag}` |

### Private registry — repositoryCredentials

ECR 외 private registry (GHCR 등) 의 인증:

```json
{
  "image": "ghcr.io/org/private:v1",
  "repositoryCredentials": {
    "credentialsParameter": "arn:aws:secretsmanager:...:secret:ghcr-pat"
  }
}
```

Secrets Manager value 예:
```json
{ "username": "myuser", "password": "ghp_xxxxxxxx" }
```

Task Execution Role 에 `secretsmanager:GetSecretValue` 권한 필요.

> ECR (같은 AWS 계정) 는 IAM 만으로 — credentials parameter 불필요.

## 2. digest pinning (production 권장)

```json
{
  "image": "123.dkr.ecr.ap-northeast-2.amazonaws.com/bkurs-be@sha256:abc123..."
}
```

→ tag 변경에도 영원히 같은 image. supply chain attack 방어.

자세히 → [[../../../../docker/practice/10-image-tags-and-layers]]

## 3. CPU / Memory — Fargate

Fargate 는 **task 단위** + container 별 추가 가능. 정해진 조합만 허용:

| vCPU | RAM 선택지 |
| --- | --- |
| 0.25 | 0.5 / 1 / 2 GB |
| 0.5 | 1 / 2 / 3 / 4 GB |
| 1 | 2~8 GB (1GB step) |
| 2 | 4~16 GB |
| 4 | 8~30 GB |
| 8 | 16~60 GB |
| 16 | 32~120 GB |

```json
{
  "cpu": "1024",           // 1 vCPU (= 1024 단위)
  "memory": "2048",        // 2 GB
  "containerDefinitions": [
    { "cpu": 512, "memoryReservation": 1024, "memory": 1536 }
  ]
}
```

container 단위 한도:
- `cpu` — soft / share (relative weight)
- `memoryReservation` — soft limit
- `memory` — hard limit (초과 시 OOMKilled)

## 4. CPU / Memory — EC2 launch

EC2 의 경우 task 단위 cpu/memory 선택 자유 (어차피 EC2 instance 가 한도). bin-packing 효율 ↑.

```json
{
  "cpu": "256",            // 0.25 vCPU
  "memory": "512",
  "containerDefinitions": [
    {
      "cpu": 256,
      "memoryReservation": 256,
      "memory": 512
    }
  ]
}
```

EC2 의 `t3.medium` (2 vCPU / 4 GB) 에 위 task 8개 packing 가능.

## 5. ephemeral storage (Fargate)

```json
{
  "ephemeralStorage": {
    "sizeInGiB": 100        // 21~200, default 20
  }
}
```

비용: 추가 GB 마다 $0.000111/GB/h. 100GB 24/7 ≈ $8/월.

EC2 launch: 호스트의 EBS 자유롭게 사용 (별도 제한 없음).

영구 storage 는 EFS — [[04-volumes-and-efs]].

## 6. PID limit (EC2 only)

```json
{
  "pidsLimit": 200
}
```

→ fork bomb / 누수 방지. Fargate 는 자동 (변경 불가).

## 7. GPU (EC2 only)

```json
{
  "resourceRequirements": [
    { "type": "GPU", "value": "1" }
  ]
}
```

EC2 instance type 이 GPU 지원해야 (예: p3.2xlarge, g4dn.xlarge). Fargate 미지원.

자세히: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-gpu.html

## 8. ulimits

```json
{
  "ulimits": [
    { "name": "nofile",  "softLimit": 65535, "hardLimit": 65535 },
    { "name": "nproc",   "softLimit": 4096,  "hardLimit": 4096 }
  ]
}
```

JVM / Node / DB 에서 file descriptor / process 한도 — 위 default 권장.

## 9. environment 변수 (간단)

```json
{
  "environment": [
    { "name": "ENV", "value": "prod" },
    { "name": "JAVA_OPTS", "value": "-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError" }
  ]
}
```

자세히 → [[05-environment-and-secrets]]

## 10. JVM 메모리 in Fargate

`-XX:MaxRAMPercentage=75` 권장 (heap = 컨테이너 RAM × 0.75):

| Task RAM | MaxRAMPercentage 75 | non-heap 헤드룸 |
| --- | --- | --- |
| 1 GB | 768 MB heap | 256 MB |
| 2 GB | 1.5 GB | 0.5 GB |
| 4 GB | 3 GB | 1 GB |

> JVM 11+ 가 container 메모리 자동 인식 — Fargate / EC2 둘 다 OK.

## 11. 권장 시작 spec — Spring Boot

```json
{
  "cpu": "1024",                          // 1 vCPU
  "memory": "2048",                       // 2 GB
  "runtimePlatform": {
    "cpuArchitecture": "ARM64",            // Graviton -20%
    "operatingSystemFamily": "LINUX"
  },
  "ephemeralStorage": { "sizeInGiB": 30 },
  "containerDefinitions": [{
    "name": "api",
    "image": "...",
    "essential": true,
    "memoryReservation": 1536,
    "ulimits": [{ "name": "nofile", "softLimit": 65535, "hardLimit": 65535 }],
    "environment": [
      { "name": "JAVA_OPTS", "value": "-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError" }
    ]
  }]
}
```

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Fargate 의 cpu/memory 가 허용 조합 아님 | 위 §3 표 |
| JVM 이 컨테이너 메모리 안 봄 (Java 8u131 이전) | Java 11+ 사용 — 자동 |
| arm64 image 가 amd64 만 build | multi-arch build ([[../../../../docker/practice/06-multi-platform-build]]) |
| ECR cross-account pull 실패 | repository policy 추가 |
| Fargate `ephemeralStorage` 21~200 만 | 그 외 값 reject |
| pids limit Fargate 적용 안 됨 | EC2 only |

## 13. 다음

- ▶ [[03-network-mode]] — awsvpc / bridge / host / none
- ▶ [[04-volumes-and-efs]] — 영구 storage
- ▶ [[05-environment-and-secrets]] — env / secret
- ▶ [[../01-fundamentals/03-fargate-deep]] — Graviton / Fargate spec

## 14. 영상 / 자료

| 자료 |
| --- |
| [Right-sizing Containers — re:Invent](https://www.youtube.com/watch?v=qsHF1zaxNAg) |
| [JVM in containers — Aboullaite](https://www.youtube.com/watch?v=KvBakaTBV-c) |
| [Graviton 활용 — AWS](https://www.youtube.com/watch?v=mZ-ZGyqLMzg) |

## 15. 참고

- [[../ecs]]
- 공식 task size: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-cpu-memory-error.html
