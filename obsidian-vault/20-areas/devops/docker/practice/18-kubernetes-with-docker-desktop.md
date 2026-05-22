---
title: "Docker 실습 18 — Kubernetes with Docker Desktop (Pod / Service / Ingress / Helm)"
kind: practice
status: current
created_at: 2026-05-22T14:40:00+09:00
tags: [devops, docker, practice, kubernetes, helm]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../desktop/kubernetes]]"
  - "[[../orbstack/k8s]]"
---

# 실습 18 — Kubernetes with Docker Desktop

> 시간: 45분. 학습 포인트: K8s 활성 / Pod / Service / Deployment / Ingress / Helm /
> ConfigMap / Secret / PVC / Liveness/Readiness / image 공유. OrbStack K3s 도 동등.

## 1. K8s 활성

### Docker Desktop
Settings > **Kubernetes** > **Enable Kubernetes** → Apply & restart.

### OrbStack
```bash
orb start k8s
```

또는 dashboard > Kubernetes > Start.

### 확인
```bash
kubectl config get-contexts
# docker-desktop  *  docker-desktop  docker-desktop
# 또는
# orbstack        *  orbstack        orbstack

kubectl get nodes
# NAME             STATUS   ROLES                  AGE   VERSION
# docker-desktop   Ready    control-plane,master   1m    v1.31.x
```

## 2. 첫 Pod

```bash
# CLI
kubectl run nginx --image=nginx:1.27-alpine --port=80
kubectl get pods
kubectl logs nginx
kubectl exec -it nginx -- sh
kubectl delete pod nginx
```

YAML 방식 (권장):
```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels: { app: nginx }
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports: [{ containerPort: 80 }]
```

```bash
kubectl apply -f nginx-pod.yaml
```

## 3. Service — Pod 노출

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: LoadBalancer            # Desktop/OrbStack 가 localhost 자동 매핑
  selector: { app: nginx }
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f nginx-svc.yaml
kubectl get svc
# nginx-svc  LoadBalancer  10.96.x.x  localhost  80:31234/TCP
curl http://localhost:80
```

Service type:
| type | 의미 |
| --- | --- |
| ClusterIP | cluster 내부만 (default) |
| NodePort | node 의 :30000~ 직접 |
| LoadBalancer | cluster 외부 (cloud LB or local Desktop) |
| ExternalName | DNS alias |

## 4. Deployment — 다중 replica + rolling

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: nginx }
spec:
  replicas: 3
  selector: { matchLabels: { app: nginx } }
  template:
    metadata: { labels: { app: nginx } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports: [{ containerPort: 80 }]
          resources:
            limits:   { cpu: "200m", memory: 128Mi }
            requests: { cpu: "50m",  memory: 32Mi }
          livenessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 2
            periodSeconds: 5
```

```bash
kubectl apply -f deploy.yaml
kubectl get deploy
kubectl get pods                            # 3 개

# scale
kubectl scale deploy/nginx --replicas=5

# rolling update
kubectl set image deploy/nginx nginx=nginx:1.27.2-alpine
kubectl rollout status deploy/nginx
kubectl rollout undo deploy/nginx           # 롤백
```

## 5. local image 공유

Desktop / OrbStack 의 K8s 는 docker engine 과 image store 공유:

```bash
# 빌드
docker build -t myapp:dev .

# K8s 에서 즉시 사용 — imagePullPolicy: Never 명시
kubectl run myapp --image=myapp:dev --image-pull-policy=Never
```

다른 K8s (minikube / kind / EKS) 는 별도 `image load` 또는 push 필요.

## 6. ConfigMap / Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  ENV: production
  LOG_LEVEL: info
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret }
type: Opaque
stringData:
  DB_PASSWORD: my-secret-pwd
  JWT_SECRET: my-jwt-key
```

Pod 에 inject:
```yaml
containers:
  - name: app
    image: myapp:dev
    envFrom:
      - configMapRef: { name: app-config }
      - secretRef:    { name: app-secret }
```

> Secret 은 base64 — 권장: external-secrets (SSM / Vault) 또는 sealed-secrets.

## 7. PVC — 영구 볼륨

Desktop K8s 기본 storageClass `hostpath`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: db-data }
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests: { storage: 1Gi }
---
apiVersion: v1
kind: Pod
metadata: { name: db }
spec:
  containers:
    - name: postgres
      image: postgres:16
      env: [{ name: POSTGRES_PASSWORD, value: secret }]
      volumeMounts:
        - { name: data, mountPath: /var/lib/postgresql/data }
  volumes:
    - name: data
      persistentVolumeClaim: { claimName: db-data }
```

```bash
kubectl apply -f db.yaml
kubectl get pvc
kubectl get pv
```

## 8. Ingress

### nginx-ingress 설치

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### Ingress resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: nginx-svc, port: { number: 80 } }
```

```bash
# /etc/hosts 에 추가
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
curl http://myapp.local
```

OrbStack 는 Traefik 기본 + `myapp.k8s.orb.local` 자동.

## 9. Helm

```bash
brew install helm

# Postgres 빠른 띄움
helm install pg bitnami/postgresql \
  --repo https://charts.bitnami.com/bitnami \
  --set auth.postgresPassword=secret

# 자체 chart
helm create mychart
helm install myapp ./mychart
helm upgrade --install myapp ./mychart --set image.tag=v2
helm rollback myapp 1
```

영상: [Helm tutorial — TechWorld with Nana](https://www.youtube.com/watch?v=-ykwb1d0DXU)

## 10. Compose → K8s (Compose Bridge)

Docker Desktop > Settings > Features in development > **Enable Compose Bridge**:

```bash
docker compose bridge convert
# 생성: ./out/*.yaml
kubectl apply -f ./out/
```

dev 의 compose 그대로 prod 같은 K8s 환경.

## 11. 실습 종합 — 3-tier app

`bkurs-stack.yaml`:
```yaml
# Postgres (StatefulSet)
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: db }
spec:
  serviceName: db
  replicas: 1
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - { name: POSTGRES_PASSWORD, value: secret }
            - { name: POSTGRES_DB, value: bkurs }
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 1Gi } }
---
apiVersion: v1
kind: Service
metadata: { name: db }
spec:
  clusterIP: None
  selector: { app: db }
  ports: [{ port: 5432 }]
---
# API
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 2
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: api
          image: ghcr.io/seum-running/bkurs-be:v1
          ports: [{ containerPort: 8080 }]
          env:
            - { name: DB_HOST, value: db }
            - { name: DB_PASSWORD, valueFrom: { secretKeyRef: { name: app-secret, key: DB_PASSWORD } } }
          livenessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
          readinessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
---
apiVersion: v1
kind: Service
metadata: { name: api-svc }
spec:
  type: LoadBalancer
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
```

```bash
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=secret
kubectl apply -f bkurs-stack.yaml
kubectl get all
curl http://localhost/actuator/health
```

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `imagePullPolicy: Always` 인데 local image | `Never` 또는 `IfNotPresent` |
| LoadBalancer External-IP `<pending>` | Desktop/OrbStack 자동 — 1-2분 대기 |
| Pod 가 `OOMKilled` | resources.limits.memory 늘림 |
| `kubectl exec` 가 hang | Pod 가 `Pending` 또는 image 미pull |
| PVC `Pending` | storageClass 확인 |

## 13. 영상 / 자료

| 자료 |
| --- |
| [Kubernetes Crash Course — TechWorld with Nana](https://www.youtube.com/watch?v=s_o8dEzUF34) |
| [Helm Tutorial — TechWorld with Nana](https://www.youtube.com/watch?v=-ykwb1d0DXU) |
| [Compose Bridge — Docker blog](https://www.docker.com/blog/compose-bridge/) |
| [k9s — terminal UI](https://k9scli.io/) |

## 14. 다음

- ▶ [[19-github-actions-build-push]] — CI 통합
- ▶ [[20-end-to-end-bkurs-style]] — 종합 프로젝트
- ▶ [[../desktop/kubernetes]] — Desktop K8s
- ▶ [[../orbstack/k8s]] — OrbStack K3s

## 15. 참고

- 공식 K8s docs: https://kubernetes.io/docs/
- Helm: https://helm.sh/
- bitnami charts: https://github.com/bitnami/charts
