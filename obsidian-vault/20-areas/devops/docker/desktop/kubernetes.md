---
title: "Docker Desktop — Kubernetes 통합"
kind: knowledge
status: current
created_at: 2026-05-22T11:38:00+09:00
tags: [devops, docker, docker-desktop, kubernetes]
home_hub: docker
related:
  - "[[settings]]"
  - "[[../../kubernetes/kubernetes]]"
---

# Docker Desktop — Kubernetes

> single-node Kubernetes cluster 를 GUI 토글로 활성. minikube/kind 대안. local 개발에 충분.

## 1. 활성화

Settings → **Kubernetes** → **Enable Kubernetes** 체크 → **Apply & restart**.

첫 활성: 컴포넌트 image pull (control-plane / kube-proxy / coredns 등) 약 1~3분.

확인:
```bash
kubectl config get-contexts
# docker-desktop  *  docker-desktop  docker-desktop

kubectl get nodes
# NAME             STATUS   ROLES           AGE   VERSION
# docker-desktop   Ready    control-plane   1m    v1.31.x
```

## 2. 구성 요소

- **Control plane**: API server / etcd / scheduler / controller-manager — 단일 노드에 통합
- **Networking**: kube-proxy + CoreDNS (default)
- **Storage**: hostpath-provisioner (PVC 자동 binding)
- **Runtime**: containerd (Docker Desktop 의 도커 엔진과 image store 공유 — 빌드한 image 즉시 사용)

## 3. image 공유 — 핵심 가치

```bash
# Desktop 에서 빌드
docker build -t myapp:dev .

# 즉시 k8s 에서 사용 가능 — imagePullPolicy: Never 만 명시
kubectl run myapp --image=myapp:dev --image-pull-policy=Never
```

minikube 는 `minikube image load` 필요 — Desktop K8s 는 image store 공유라 불필요.

## 4. context 전환

```bash
# 일반 docker
kubectl config use-context docker-desktop

# 다른 cluster (예: prod EKS) 와 빠른 전환
kubectl config use-context arn:aws:eks:ap-northeast-2:...
```

GUI 우상단의 context 표시 — 클릭하여 전환.

## 5. 리소스 한도

K8s 가 Docker 와 같은 VM 안. Settings > Resources 의 CPU/RAM 을 함께 공유. K8s 활성 시 권장:
- CPU 4+ cores
- RAM 6~8GB+

부족 시 K8s pods Pending 또는 OOMKilled.

## 6. 자주 쓰는 통합

### local Helm chart 테스트

```bash
helm install myapp ./chart --create-namespace --namespace dev
kubectl get pods -n dev
```

### Ingress (port forward 또는 service NodePort)

K8s default 는 nginx-ingress 없음. 설치:
```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

`localhost:80` 으로 접근.

### LoadBalancer service

K8s LoadBalancer type → Docker Desktop 가 자동 localhost port 매핑 (single-node).
```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
```
→ `curl http://localhost`

### PVC

기본 storageClass `hostpath` — `/var/lib/docker/k8s/` 에 데이터 저장. 컨테이너 재시작 후에도 보존.

## 7. 진단 / log

| 작업 | 명령 |
| --- | --- |
| 모든 node | `kubectl get nodes -o wide` |
| 시스템 pod | `kubectl get pods -n kube-system` |
| K8s component log | `kubectl logs -n kube-system <pod>` |
| docker 측 K8s 컨테이너 | `docker ps -a --filter "label=io.kubernetes.pod.namespace"` |
| K8s 자체 reset | Settings > Kubernetes > **Reset Kubernetes Cluster** |
| 디스크 정리 (image / volume) | `docker system prune -a --volumes` (K8s 사용 image 포함 — 주의) |

## 8. kind / minikube 와 비교

| 항목 | Desktop K8s | kind | minikube |
| --- | --- | --- | --- |
| 설치 | GUI 토글 | `brew install kind` + cluster create | `brew install minikube` + start |
| Multi-node | ✗ single | ✓ (config) | ✓ (`--nodes`) |
| image 공유 | ✓ 자동 | `kind load docker-image` | `minikube image load` |
| 자원 | Desktop VM 공유 | host docker 위 컨테이너 | 별도 VM (또는 컨테이너) |
| 적합 | 빠른 local 개발 | CI / multi-node | 다양한 driver / addon |

> 다중 노드 / specific K8s 버전 / 다양한 CNI 테스트는 **kind**.
> 단일 노드 빠른 dev 는 **Desktop K8s**.

## 9. troubleshooting

| 증상 | 해결 |
| --- | --- |
| 활성 중 멈춤 (Starting...) | RAM 부족 — Settings > Resources 늘림 + Reset Kubernetes Cluster |
| `kubectl` 명령 실패 (connection refused) | engine 정지 또는 context 잘못 — `kubectl config get-contexts` |
| Pod ImagePullBackOff (local image 인데) | `imagePullPolicy: Never` 누락 또는 image tag mismatch |
| LoadBalancer 가 `<pending>` 상태 | Desktop K8s 에서는 잠시 후 자동 localhost — 1-2분 대기 |
| 디스크 폭주 | image / etcd / log — `docker system prune` + Reset cluster |

## 10. 다음

- ▶ [[../../kubernetes/kubernetes]] — K8s 일반 hub
- ▶ Compose Bridge (compose → manifest 변환): [[settings#7]] features
- ▶ Extensions — Lens / k9s 통합: [[extensions]]
