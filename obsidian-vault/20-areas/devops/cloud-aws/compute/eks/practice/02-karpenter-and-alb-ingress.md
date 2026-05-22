---
title: "EKS 실습 2 — Karpenter + ALB Ingress"
kind: practice
status: current
created_at: 2026-05-22T16:40:00+09:00
tags: [aws, eks, practice, karpenter, alb, autoscaling]
home_hub: eks
---

# 실습 2 — Karpenter + ALB Ingress

> 목표: Karpenter 설치 + NodePool + 부하 → just-in-time provisioning + ALB Ingress. 60분.

> 실습 1 의 lab-eks cluster 위에서 진행 (Fargate Profile 그대로).

## 1. Karpenter IAM

```bash
TEMPOUT="$(mktemp)"
curl -s "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v1.0.0/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml" > "$TEMPOUT"

aws cloudformation deploy \
  --stack-name Karpenter-lab-eks \
  --template-file "$TEMPOUT" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ClusterName=lab-eks
```

→ KarpenterControllerRole + KarpenterNodeRole + KarpenterInterruptionQueue (SQS).

## 2. EKS Pod Identity Association

```bash
aws eks create-pod-identity-association --cluster-name lab-eks \
  --namespace karpenter \
  --service-account karpenter \
  --role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/KarpenterControllerRole-lab-eks

# Node Role 의 EKS cluster 등록
eksctl create iamidentitymapping --cluster lab-eks \
  --arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/KarpenterNodeRole-lab-eks \
  --username system:node:{{EC2PrivateDNSName}} \
  --group system:bootstrappers \
  --group system:nodes
```

## 3. Subnet / SG tag (Karpenter discovery)

```bash
SUBNETS=$(aws eks describe-cluster --name lab-eks \
  --query 'cluster.resourcesVpcConfig.subnetIds[]' --output text)

for s in $SUBNETS; do
  aws ec2 create-tags --resources $s --tags Key=karpenter.sh/discovery,Value=lab-eks
done

CLUSTER_SG=$(aws eks describe-cluster --name lab-eks \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)
aws ec2 create-tags --resources $CLUSTER_SG --tags Key=karpenter.sh/discovery,Value=lab-eks
```

## 4. Helm 설치

```bash
helm registry logout public.ecr.aws 2>/dev/null

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version 1.0.0 \
  --namespace karpenter --create-namespace \
  --set "settings.clusterName=lab-eks" \
  --set "settings.interruptionQueue=Karpenter-lab-eks" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait

kubectl get pod -n karpenter
# karpenter-xxx   1/1   Running
```

## 5. NodePool + EC2NodeClass

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: { name: default }
spec:
  template:
    metadata:
      labels: { workload-type: general }
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["arm64"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["m", "c"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["6"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h
  limits: { cpu: "100", memory: "100Gi" }
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata: { name: default }
spec:
  amiFamily: AL2023
  amiSelectorTerms:
    - alias: al2023@latest
  role: KarpenterNodeRole-lab-eks
  subnetSelectorTerms:
    - tags: { karpenter.sh/discovery: lab-eks }
  securityGroupSelectorTerms:
    - tags: { karpenter.sh/discovery: lab-eks }
  metadataOptions:
    httpTokens: required
    httpPutResponseHopLimit: 2
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs: { volumeType: gp3, volumeSize: 50Gi, encrypted: true, deleteOnTermination: true }
  tags:
    karpenter.sh/discovery: lab-eks
    environment: lab
```

```bash
kubectl apply -f nodepool.yaml
```

## 6. 부하 → Karpenter 동작 관찰

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: load-test, namespace: default }
spec:
  replicas: 0
  selector: { matchLabels: { app: load } }
  template:
    metadata: { labels: { app: load } }
    spec:
      nodeSelector: { workload-type: general }
      containers:
        - name: pause
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests: { cpu: "1", memory: "1Gi" }
```

```bash
kubectl apply -f load.yaml

# 5 → 20 → 50 replica
kubectl scale deployment load-test --replicas=20

# Karpenter 가 node 추가 관찰
watch -n 5 "kubectl get nodes -L node.kubernetes.io/instance-type,karpenter.sh/capacity-type"

# Karpenter logs
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter -f
```

→ pending pod 가 ~30초 안에 새 node 에서 시작.

## 7. Consolidation 관찰

```bash
kubectl scale deployment load-test --replicas=2

# 1~2분 후
kubectl get nodes
# Karpenter 가 빈 node 종료
```

## 8. ALB Ingress (실습 1 의 ALB Controller 재사용)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, namespace: default }
spec:
  replicas: 3
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      nodeSelector: { workload-type: general }
      containers:
        - name: api
          image: nginxdemos/hello
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: "200m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  type: ClusterIP
  selector: { app: api }
  ports: [{ port: 80, targetPort: 80 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
```

```bash
kubectl apply -f api.yaml
sleep 90
ADDR=$(kubectl get ingress api -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$ADDR
```

## 9. HPA + Karpenter 콤보

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 50 } }
```

```bash
# 부하 시뮬
kubectl run -i --tty load --rm --image=busybox --restart=Never -- /bin/sh
$ while true; do wget -q -O- http://api/; done
```

→ HPA → 더 많은 pod → Karpenter → 새 node.

## 10. Clean up

```bash
kubectl delete -f nodepool.yaml
kubectl delete deployment load-test api
kubectl delete svc api
kubectl delete ingress api

helm uninstall karpenter -n karpenter
aws cloudformation delete-stack --stack-name Karpenter-lab-eks
```

## 11. 학습 포인트

- Karpenter Controller 가 Fargate 에서 실행 (lab-eks 의 Fargate Profile 활용)
- NodePool requirements 의 의미
- just-in-time provisioning 30~60초
- Consolidation 자동 종료
- HPA + Karpenter 콤보 — pod 가 늘면 node 도 즉시

## 12. 다음

- ▶ [[03-irsa-app-end-to-end]]
- ▶ [[../04-operations/01-autoscaling-cas-karpenter]]

## 13. 참고

- [[../eks]]
- [[../01-fundamentals/03-node-types]]
- [[../04-operations/01-autoscaling-cas-karpenter]]
