---
title: "EKS — Elastic Kubernetes Service ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:00:00+09:00
tags: [devops, cloud-aws, eks, kubernetes]
---

# EKS — Elastic Kubernetes Service ★

**[[cloud-aws|↑ cloud-aws]]**

---

## 1. 무엇

```
AWS 가 관리하는 Kubernetes:
  - control plane (apiserver / etcd / scheduler) 를 AWS 가 운영
  - worker node 는 우리가 (EC2 / Fargate)
  - vanilla k8s (CNCF certified)
  - AWS service 와 깊이 통합 (IAM / VPC / ELB / EBS)

비용:
  - control plane: $0.10/h = $73/mo per cluster
  - worker = EC2 또는 Fargate (별도)
```

→ "k8s 운영 부담 ↓, AWS 통합 강력" 의 선택지.

---

## 2. 대안

| | 무엇 |
| --- | --- |
| **EKS** | 표준 / 풍부 |
| **ECS** | AWS 전용 / 단순 |
| **Fargate** | serverless container (ECS/EKS 위) |
| **App Runner** | 단순 web service |
| **self-managed k8s** (EC2) | full control / 운영 부담 |
| **k3s on EC2** | 가벼움 |

→ 대규모 / multi-cloud = EKS. AWS only + 단순 = ECS.

---

## 3. 구조 (★)

```
[Control Plane]               ← AWS 가 관리 (multi-AZ 자동)
   - apiserver, etcd, scheduler
   - VPC 안 (cross-account ENI)

[Worker Node]                 ← 사용자 관리
   - EC2 (Managed / Self-managed Node Group)
   - 또는 Fargate (serverless)
   - kubelet, kube-proxy, CNI (AWS VPC CNI)

[Add-ons]
   - VPC CNI (네트워크)
   - CoreDNS (DNS)
   - kube-proxy (service)
   - EBS CSI Driver (storage)
   - EFS CSI Driver
   - Karpenter (autoscale)
```

---

## 4. cluster 생성 (eksctl)

```bash
brew install eksctl

# 빠른 cluster (수십 분 소요)
eksctl create cluster \
    --name prod \
    --region ap-northeast-2 \
    --version 1.30 \
    --nodegroup-name workers \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 1 \
    --nodes-max 10 \
    --managed
```

→ 더 권장: ClusterConfig YAML 으로 declarative.

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: prod
  region: ap-northeast-2
  version: "1.30"

iam:
  withOIDC: true        # ★ IRSA 활성

vpc:
  cidr: 10.30.0.0/16
  nat: {gateway: HighlyAvailable}    # AZ 별 NAT (★ HA)

managedNodeGroups:
  - name: workers-on-demand
    instanceType: m6i.large
    desiredCapacity: 3
    minSize: 1
    maxSize: 10
    privateNetworking: true
    iam:
      withAddonPolicies:
        autoScaler: true
        ebs: true
        efs: true
        cloudWatch: true

  - name: workers-spot
    instanceTypes: [m6i.large, m5.large, m5a.large]
    spot: true
    desiredCapacity: 5
    minSize: 0
    maxSize: 50
    labels: {capacity-type: spot}
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver

cloudWatch:
  clusterLogging:
    enableTypes: ["audit", "authenticator", "controllerManager", "scheduler"]
```

```bash
eksctl create cluster -f cluster.yaml
```

---

## 5. kubeconfig 설정

```bash
# kubeconfig 자동 update
aws eks update-kubeconfig \
    --region ap-northeast-2 \
    --name prod \
    --alias prod-eks

kubectl get nodes
kubectl get pods -A
```

---

## 6. node 종류 (★)

### A. Managed Node Group (★ 권장)

```
EC2 Auto Scaling Group:
  - AWS 가 AMI / patch / update
  - eksctl / Terraform 으로 관리
  - graceful drain 자동

설정:
  - instance type
  - capacity (min/max/desired)
  - Spot 가능
  - labels / taints
  - subnet / SG
```

### B. Self-managed Node Group

```
직접 EC2 띄움:
  - 더 control (custom AMI, ulimit 등)
  - 운영 부담 ↑
  - Karpenter / Bottlerocket 사용 시
```

### C. Fargate (★ serverless)

```
node 없음, pod 만:
  - per-pod 과금
  - 빠른 시작
  - cold start 약간
  - DaemonSet 제약
  - GPU / privileged 안 됨

용도:
  - 작은 / sporadic workload
  - dev / staging
  - CI / batch
```

```yaml
# FargateProfile
apiVersion: eksctl.io/v1alpha5
fargateProfiles:
  - name: dev
    selectors:
      - namespace: dev
      - namespace: kube-system
        labels: {k8s-app: kube-dns}     # CoreDNS 도 Fargate 가능
```

---

## 7. IRSA (★ IAM Roles for Service Accounts)

```
일반 EC2:
  Pod → instance profile → IAM Role (전체 node)
  → 한 node 의 모든 pod 가 같은 권한 (위험)

IRSA:
  Pod 의 ServiceAccount 별 IAM Role
  → least privilege
  → AWS SDK 가 자동 인식

구조:
  Pod → ServiceAccount (annotation: eks.amazonaws.com/role-arn)
        → OIDC token → STS AssumeRoleWithWebIdentity
        → IAM Role의 temporary credential
```

```yaml
# 1. OIDC provider (cluster 별 1번)
eksctl utils associate-iam-oidc-provider --cluster prod --approve

# 2. ServiceAccount + IAM Role (eksctl 자동 양쪽 생성)
eksctl create iamserviceaccount \
    --cluster prod \
    --namespace app \
    --name s3-reader \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
    --approve

# 3. Pod 의 ServiceAccount 지정
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/eksctl-prod-addon-iamserviceaccount-app-s3-reader-Role1-XXX

---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: s3-reader
  containers:
    - name: app
      # SDK 가 자동 IAM Role 의 credential 사용
```

→ access key 코드에 hardcode 절대 X.

---

## 8. EKS Pod Identity (★ 2023+ 새 방식)

```
IRSA 의 더 단순한 대안:
  - OIDC provider 설정 안 필요
  - Pod Identity Agent (DaemonSet)
  - 더 빠른 token refresh

사용:
  aws eks create-pod-identity-association \
      --cluster-name prod \
      --namespace app \
      --service-account s3-reader \
      --role-arn arn:aws:iam::123:role/S3ReaderRole
```

→ AWS 권장 (IRSA 보다).

---

## 9. VPC CNI (★)

```
AWS native CNI:
  - pod IP = VPC subnet 의 IP (★ flat)
  - 외부 network 에서 직접 도달
  - SG per pod 가능
  
한계:
  - pod 수 = ENI 의 IP 수 (instance type 별)
    예: m5.large = 30 pod max
  - subnet IP 부족 가능

해결:
  - prefix delegation (★ /28 prefix 할당)
    예: m5.large = 110 pod
  - custom networking (secondary CIDR)
  - 작은 instance type 더 많이
```

```bash
# prefix delegation 활성
kubectl set env daemonset aws-node \
    -n kube-system \
    ENABLE_PREFIX_DELEGATION=true \
    WARM_PREFIX_TARGET=1
```

---

## 10. cluster autoscaling

### A. Cluster Autoscaler (전통)

```yaml
# IAM role
eksctl create iamserviceaccount \
    --cluster=prod \
    --namespace=kube-system \
    --name=cluster-autoscaler \
    --attach-policy-arn=... \
    --approve

# helm install
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
    --namespace kube-system \
    --set autoDiscovery.clusterName=prod \
    --set awsRegion=ap-northeast-2
```

→ ASG 기반. 단순하지만 느림.

### B. Karpenter (★ 권장)

```bash
# Karpenter 설치
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
    --namespace kube-system \
    --set settings.clusterName=prod \
    --set settings.interruptionQueue=karpenter-prod-queue
```

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata: {name: default}
spec:
  template:
    spec:
      requirements:
        - {key: karpenter.k8s.aws/instance-family, operator: In, values: [m, c, r]}
        - {key: karpenter.k8s.aws/instance-size, operator: NotIn, values: [nano, micro, small]}
        - {key: karpenter.sh/capacity-type, operator: In, values: [spot, on-demand]}
        - {key: kubernetes.io/arch, operator: In, values: [amd64, arm64]}
      nodeClassRef: {name: default}
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 720h
  limits:
    cpu: 1000
```

→ ASG 안 거치고 직접 EC2. 30s 단위 scale. Spot 우수.

---

## 11. ALB / NLB 통합

```bash
# AWS Load Balancer Controller 설치
eksctl create iamserviceaccount \
    --cluster=prod \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::123:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=prod \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
```

```yaml
# Ingress (ALB)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip          # ★ pod 직접
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443},{"HTTP":80}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health
    alb.ingress.kubernetes.io/group.name: shared-alb    # 여러 Ingress 가 같은 ALB
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: {name: api, port: {number: 80}}
```

```yaml
# Service (NLB)
apiVersion: v1
kind: Service
metadata:
  name: tcp-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector: {app: tcp-app}
  ports:
    - {port: 5432, targetPort: 5432, protocol: TCP}
```

---

## 12. EBS / EFS storage

```yaml
# EBS gp3 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations: {storageclass.kubernetes.io/is-default-class: "true"}
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# EFS (RWX 가능)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: efs-sc}
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0abc123
  directoryPerms: "700"
```

---

## 13. observability

```bash
# Container Insights (CloudWatch)
helm install aws-cloudwatch-metrics aws-cloudwatch-metrics/aws-cloudwatch-metrics \
    -n amazon-cloudwatch --create-namespace

# Fluent Bit (log → CloudWatch)
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/...

# Prometheus / Grafana (self-host)
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# 또는 AMP (managed Prometheus) + AMG (Grafana)
aws amp create-workspace --alias prod
```

---

## 14. upgrade

```bash
# 1. control plane
eksctl upgrade cluster --name prod --version 1.31 --approve

# 2. add-on
eksctl update addon --name vpc-cni --cluster prod
eksctl update addon --name coredns --cluster prod

# 3. managed node group (rolling)
eksctl upgrade nodegroup --cluster prod --name workers

# 4. self-managed = manual

# 순서: control plane → add-on → worker
# minor 하나씩 (1.29 → 1.30 → 1.31)
```

---

## 15. cost 절감 (★)

```
1. Karpenter + Spot
   → 50-70% node cost ↓

2. Graviton (ARM64)
   → 20% 더 저렴

3. right-size
   → VPA recommendation
   → Goldilocks

4. Fargate for batch / sporadic
   → idle cost 0

5. cluster autoscaler / Karpenter consolidation
   → underutilization 회수

6. EBS gp3 (gp2 보다 cheaper / 더 빠름)

7. dev / staging 끄기 (CronJob scale 0)

8. observability cost
   → AMP + AMG (self-host 보다 운영 X)
   → log = Loki

9. control plane $73/mo 의 부담
   → 작은 환경 = ECS Fargate 또는 App Runner
```

---

## 16. 보안 best practice

```
☐ IRSA / Pod Identity (★ 필수)
☐ private endpoint (control plane 의 public access off)
☐ Network Policy (Calico / Cilium)
☐ Pod Security Admission (restricted profile)
☐ image scan (ECR + Trivy)
☐ cosign signed image only
☐ secrets external (Secrets Manager + ESO)
☐ encryption at rest (EBS / S3 / EFS)
☐ etcd encryption (KMS)
☐ audit log → CloudWatch + Athena
☐ GuardDuty for EKS
☐ Kyverno / Gatekeeper
```

---

## 17. troubleshoot

```bash
# cluster health
aws eks describe-cluster --name prod

# add-on health
aws eks describe-addon --cluster-name prod --addon-name vpc-cni

# node 의 IAM role
kubectl get node -o yaml | grep -i iam

# pod 의 IRSA
kubectl describe pod <name> | grep -A5 "ServiceAccount"
kubectl exec <pod> -- aws sts get-caller-identity

# control plane log (CloudWatch)
aws logs filter-log-events \
    --log-group-name /aws/eks/prod/cluster \
    --filter-pattern "ERROR"

# VPC CNI debug
kubectl logs -n kube-system -l k8s-app=aws-node
```

---

## 18. 함정

1. **control plane public + IP 제한 없음** — internet 노출.
2. **NAT GW 1 AZ** — AZ fail 시 outbound 끊김. HA NAT.
3. **VPC CNI IP 부족** — prefix delegation 켜기.
4. **Spot + critical workload** — capacity fail 시 영향.
5. **IRSA vs Pod Identity 혼용** — 한쪽으로.
6. **kubectl alias 잘못 (다른 cluster)** — `kubectx` 사용.
7. **add-on 자동 update 안 함** — 정기 update.
8. **single node group / single AZ** — HA 부재.
9. **control plane log off** — incident 추적 불가.
10. **Karpenter 미설정 limits** — 무한 node 폭주.

---

## 19. 관련

- [[cloud-aws|↑ cloud-aws]]
- [[compute]]
- [[kubernetes|↗ k8s]]
- [[migration|↗ k3s ↔ EKS]]
