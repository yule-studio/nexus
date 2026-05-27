---
title: "troubleshooting · K8s ImagePullBackOff"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, kubernetes, image, registry, secret]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[ecr-push-denied]]"]
---

# K8s ImagePullBackOff / ErrImagePull

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```bash
kubectl get pods
NAME              READY   STATUS             AGE
myapp-x1z2        0/1     ImagePullBackOff   2m
```

## 원인

1. Image tag 없음 (typo)
2. Private registry 인증 부족 (imagePullSecrets 누락)
3. Network 차단 (private subnet without route)
4. Registry rate limit (Docker Hub 무료 100 pull/6h)

## 해결

### Image 존재 확인
```bash
docker pull myregistry/myapp:v1.2.3
# 또는
crane manifest myregistry/myapp:v1.2.3
```

### imagePullSecrets 등록
```bash
kubectl create secret docker-registry myregistry-cred \
  --docker-server=myregistry.io \
  --docker-username=$USER \
  --docker-password=$PASS \
  --docker-email=$EMAIL
```

```yaml
spec:
  imagePullSecrets:
    - name: myregistry-cred
  containers:
    - name: app
      image: myregistry.io/myapp:v1
```

### ECR — IAM Role for ServiceAccount (IRSA / EKS)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/ecr-pull-role
```

→ IAM role 가 ECR pull 권한 가지면 imagePullSecrets 불필요.

### Docker Hub rate limit
```yaml
imagePullSecrets:
  - name: dockerhub-cred   # 인증된 계정은 200/6h
```

→ 또는 ECR Pull-through Cache 사용 (Docker Hub mirror).

### Network — private subnet
- VPC Endpoint (ECR/S3)
- 또는 NAT Gateway
- [[ecs-task-failed-to-start]] 와 동일 원인

## 진단

```bash
kubectl describe pod <pod>
# Events:
#   Failed to pull image "...": rpc error: code = NotFound desc = ...
#   Failed to pull image "...": pull access denied
#   Failed to pull image "...": dial tcp ... i/o timeout

# Secret 내용 확인 (base64 decoded)
kubectl get secret myregistry-cred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# Image inspect
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].image}'
```

## 재발 방지

- CI 에서 image push 후 sanity pull 검증
- imagePullSecrets 는 default ServiceAccount 에 미리 부착
- IMMUTABLE tag (ECR) — `latest` 같은 mutable tag 금지
- 모니터링: `kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} > 0`

## 관련

- [[devops]] / [[k8s-crashloopbackoff]] / [[ecr-push-denied]]
