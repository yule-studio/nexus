---
title: "EKS 실습 1 — Cluster + Fargate Profile (eksctl)"
kind: practice
status: current
created_at: 2026-05-22T16:38:00+09:00
tags: [aws, eks, practice, eksctl, fargate]
home_hub: eks
---

# 실습 1 — EKS Cluster + Fargate

> 목표: eksctl 로 cluster 생성 + Fargate Profile + kubectl 첫 pod. 30분.

## 사전 준비

```bash
# eksctl
brew install eksctl

# kubectl
brew install kubectl

# helm
brew install helm
```

## 1. Cluster 생성 (eksctl)

```bash
eksctl create cluster --name lab-eks \
  --region ap-northeast-2 --version 1.30 \
  --without-nodegroup \
  --fargate \
  --tags Environment=lab
```

→ control plane + Fargate Profile (`default` + `kube-system` namespace) 자동.

10-15분 소요.

## 2. kubectl context

```bash
aws eks update-kubeconfig --name lab-eks --region ap-northeast-2

kubectl get nodes
# NAME                                          STATUS   ROLES    AGE   VERSION
# fargate-ip-10-0-x-x.ap-northeast-2.compute.internal   Ready    <none>   1m    v1.30.x-eks
# fargate-ip-10-0-x-y.ap-northeast-2.compute.internal   Ready    <none>   1m    v1.30.x-eks

kubectl get pods -A
# kube-system   coredns-xxx   1/1     Running
```

## 3. 추가 Fargate Profile

```bash
eksctl create fargateprofile --cluster lab-eks \
  --name app --namespace app
```

또는 YAML:
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata: { name: lab-eks, region: ap-northeast-2 }
fargateProfiles:
  - name: app
    selectors:
      - namespace: app
        labels: { tier: web }
```

## 4. Application 배포

```bash
kubectl create namespace app

kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: nginx, namespace: app, labels: { tier: web } }
spec:
  replicas: 3
  selector: { matchLabels: { app: nginx, tier: web } }
  template:
    metadata: { labels: { app: nginx, tier: web } }
    spec:
      containers:
        - name: nginx
          image: nginxdemos/hello
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "500m", memory: "512Mi" }
---
apiVersion: v1
kind: Service
metadata: { name: nginx, namespace: app }
spec:
  type: ClusterIP
  selector: { app: nginx, tier: web }
  ports: [{ port: 80, targetPort: 80 }]
EOF

kubectl get pod -n app -w
# nginx-xxx   0/1   Pending → 0/1   ContainerCreating → 1/1   Running (Fargate microVM 30~60초)
```

## 5. Port Forward (test)

```bash
kubectl port-forward -n app svc/nginx 8080:80

curl http://localhost:8080
# > <h1>NGINX...</h1>
```

## 6. ALB Ingress (옵션)

```bash
# IRSA — ALB Controller
eksctl create iamserviceaccount --cluster lab-eks \
  --namespace kube-system --name aws-load-balancer-controller \
  --role-name AWSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::aws:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts --approve

# Helm 설치
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=lab-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-northeast-2 \
  --set vpcId=$(aws eks describe-cluster --name lab-eks --query 'cluster.resourcesVpcConfig.vpcId' --output text)
```

```yaml
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: app
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: nginx, port: { number: 80 } } }
```

```bash
kubectl get ingress -n app
# ADDRESS: lab-eks-xxx.elb.ap-northeast-2.amazonaws.com

curl http://lab-eks-xxx.elb.ap-northeast-2.amazonaws.com/
```

## 7. Logs (CloudWatch via Fluent Bit on Fargate)

Fargate Profile 에는 Fluent Bit 가 컨테이너 안에 sidecar — log 자동 CloudWatch:

```yaml
# kube-system 의 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata: { name: aws-logging, namespace: aws-observability }
data:
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match *
        region ap-northeast-2
        log_group_name /aws/eks/lab-eks/fargate
        log_stream_prefix from-fluent-bit-
        auto_create_group true
```

```bash
kubectl create namespace aws-observability
kubectl apply -f -  # 위 ConfigMap

aws logs tail /aws/eks/lab-eks/fargate --follow
```

## 8. K8s Dashboard (옵션)

```bash
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace --namespace kubernetes-dashboard

kubectl -n kubernetes-dashboard create token kubernetes-dashboard-kong-proxy
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

## 9. Clean up

```bash
eksctl delete cluster --name lab-eks --wait
```

→ control plane + Fargate + IAM role + VPC + subnet 모두 자동 삭제.

## 10. 학습 포인트

- eksctl 로 cluster 생성 (15분)
- Fargate Profile + namespace selector
- Pod 가 Fargate microVM 에서 30-60초 후 Running
- ALB Ingress = AWS Load Balancer Controller + IRSA
- Fluent Bit 가 Fargate 의 log → CloudWatch 자동
- 청구: cluster $73 (한 달, 30분만 사용해도 시간 단위 $0.10)

## 11. 다음

- ▶ [[02-karpenter-and-alb-ingress]]
- ▶ [[03-irsa-app-end-to-end]]
- ▶ [[../05-security/01-irsa-pod-identity]]

## 12. 참고

- [[../eks]]
- [[../01-fundamentals/02-cluster-and-versioning]]
- [[../02-workload/02-service-and-ingress]]
