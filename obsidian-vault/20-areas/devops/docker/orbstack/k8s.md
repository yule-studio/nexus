---
title: "OrbStack — Kubernetes 통합"
kind: knowledge
status: current
created_at: 2026-05-22T12:38:00+09:00
tags: [devops, docker, orbstack, kubernetes, k3s]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[../desktop/kubernetes]]"
  - "[[../../kubernetes/kubernetes]]"
---

# OrbStack — Kubernetes

> OrbStack 는 K3s 기반 single-node Kubernetes 통합. Docker Desktop K8s 와 동등하지만 **더 가볍고
> 시작 빠름**.

## 1. 활성화

### GUI
Dashboard 좌측 **Kubernetes** → **Start** → 자동 K3s cluster.

### CLI
```bash
# 첫 시작
orb start k8s
# 또는 동등
docker context use orbstack && kubectl version --client
```

## 2. 구성

| 컴포넌트 | 값 |
| --- | --- |
| Distro | **K3s** (lightweight, < 100MB) |
| Control plane | embedded (api-server / etcd / scheduler / controller-manager) |
| CNI | flannel (default) |
| Runtime | containerd (Docker engine 과 같은 image store 공유) |
| LoadBalancer | klipper-lb (auto Mac port forward) |
| Ingress | Traefik (default, K3s 번들) |
| DNS | CoreDNS |

> Desktop K8s 가 kubeadm / kubelet 직접 사용 — OrbStack 는 K3s 임베드. 의미상 동일 K8s API.

## 3. 첫 명령

```bash
kubectl config current-context
# orbstack

kubectl get nodes
# NAME       STATUS   ROLES                  AGE   VERSION
# orbstack   Ready    control-plane,master   1m    v1.31.x+k3s1

kubectl get pods -n kube-system
# coredns / traefik / metrics-server / local-path-provisioner
```

## 4. image 공유 — 핵심

```bash
# OrbStack docker 로 빌드
docker build -t myapp:dev .

# 즉시 K8s 에서 사용 (image pull 없이)
kubectl run myapp --image=myapp:dev --image-pull-policy=Never -- port 8080

# 또는 manifest
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: myapp }
spec:
  containers:
  - name: app
    image: myapp:dev
    imagePullPolicy: Never
    ports: [{ containerPort: 8080 }]
EOF
```

## 5. Service / LoadBalancer / Ingress

### LoadBalancer
```yaml
apiVersion: v1
kind: Service
metadata: { name: myapp }
spec:
  type: LoadBalancer
  ports: [{ port: 80, targetPort: 8080 }]
  selector: { app: myapp }
```

K3s + klipper-lb 가 자동 Mac port 매핑 → `curl http://localhost:80`.

### orb.local 도메인 (자동)
```bash
# K8s service 도 자동 orb.local
curl http://myapp.k8s.orb.local
# 또는
curl http://myapp.default.svc.k8s.orb.local
```

### Traefik Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: myapp }
spec:
  rules:
  - host: myapp.orb.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: myapp, port: { number: 80 } }
```

## 6. Helm

```bash
brew install helm

# K3s context 에서 자동
helm install bitnami/postgresql --generate-name
kubectl get pods
```

## 7. PersistentVolume

K3s 기본 storageClass `local-path` — `/var/lib/rancher/k3s/storage/` 에 데이터:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
```

자동 PV provisioning — bound 즉시.

## 8. dashboard

### k9s (terminal UI)
```bash
brew install k9s
k9s
```

### Lens (full GUI)
```bash
brew install --cask lens
# OrbStack context 자동 추가
```

## 9. 다중 cluster (Linux Machine 안)

OrbStack 의 기본 K8s 외에 Linux machine 안에 별도 K3s/kind 설치 가능:

```bash
orb create ubuntu k8s-2
orb -m k8s-2 -- curl -sfL https://get.k3s.io | sh -
orb -m k8s-2 -- cat /etc/rancher/k3s/k3s.yaml > ~/.kube/k8s-2.yaml
# Mac 의 KUBECONFIG 에 추가
export KUBECONFIG=~/.kube/config:~/.kube/k8s-2.yaml
kubectl config get-contexts
```

복수 K8s cluster 동시 → multi-cluster 학습 / 실험.

## 10. Desktop K8s 와 비교

| 항목 | OrbStack | Desktop K8s |
| --- | --- | --- |
| 시작 속도 | ★★★★★ ~10초 | ★★★ ~60초 |
| 디스트로 | K3s | kubeadm 기반 |
| RAM 사용 (idle) | ~300MB | ~1GB |
| LoadBalancer | klipper-lb | none — 직접 service port 사용 |
| Ingress | Traefik (default) | none |
| dashboard | k9s/Lens (외부) | Lens extension |
| Helm | 외부 설치 | 외부 설치 |
| image 공유 | ✓ | ✓ |

## 11. troubleshooting

| 증상 | 해결 |
| --- | --- |
| `kubectl` 명령 timeout | OrbStack 가 K8s 시작 중 — `orb status` 확인 + 10초 대기 |
| Pod ImagePullBackOff | `imagePullPolicy: Never` 누락 |
| LoadBalancer External-IP `<pending>` | klipper-lb pod 상태 확인 — `kubectl get pods -n kube-system` |
| K8s reset | Dashboard > Kubernetes > Reset (또는 `orb reset k8s`) |

## 12. 다음

- ▶ Docker engine: [[docker-engine]]
- ▶ networking: [[networking]]
- ▶ Desktop K8s 비교: [[../desktop/kubernetes]]
- ▶ K8s 일반: [[../../kubernetes/kubernetes]]

## 참고

- OrbStack K8s: https://docs.orbstack.dev/kubernetes/
- K3s: https://k3s.io/
- klipper-lb: https://github.com/k3s-io/klipper-lb
