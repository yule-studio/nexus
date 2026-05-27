---
title: "EKS 실습 3 — Spring Boot + IRSA + RDS + S3 end-to-end"
kind: practice
status: current
created_at: 2026-05-22T16:42:00+09:00
tags: [aws, eks, practice, irsa, spring-boot, rds, s3]
home_hub: eks
---

# 실습 3 — Spring Boot + IRSA + RDS + S3

> 목표: bkurs-be 와 유사 — Spring Boot, RDS MySQL, S3, IRSA, ALB Ingress, Karpenter node, ExternalDNS. 90분.

> 실습 1+2 의 lab-eks 위.

## 1. RDS MySQL (Aurora Serverless v2)

```bash
VPC=$(aws eks describe-cluster --name lab-eks --query 'cluster.resourcesVpcConfig.vpcId' --output text)
SUBNETS=($(aws eks describe-cluster --name lab-eks --query 'cluster.resourcesVpcConfig.subnetIds[]' --output text))

# DB subnet group
aws rds create-db-subnet-group --db-subnet-group-name lab-db \
  --db-subnet-group-description "lab" \
  --subnet-ids ${SUBNETS[@]}

# RDS SG
DB_SG=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name lab-rds --description "rds" --query GroupId --output text)
CLUSTER_SG=$(aws eks describe-cluster --name lab-eks --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 3306 --source-group $CLUSTER_SG

# Aurora Serverless v2
aws rds create-db-cluster --db-cluster-identifier lab-mysql \
  --engine aurora-mysql --engine-version 8.0.mysql_aurora.3.05.2 \
  --master-username admin --manage-master-user-password \
  --db-subnet-group-name lab-db --vpc-security-group-ids $DB_SG \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=2

aws rds create-db-instance --db-instance-identifier lab-mysql-1 \
  --db-cluster-identifier lab-mysql --engine aurora-mysql \
  --db-instance-class db.serverless
```

→ 비밀번호 = Secrets Manager 자동 (`rds!cluster-xxx`).

## 2. S3 bucket

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
aws s3 mb s3://lab-bkurs-$ACCT
aws s3api put-bucket-encryption --bucket lab-bkurs-$ACCT \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

## 3. IAM Role (IRSA)

```bash
# Application 권한
cat > policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject","s3:ListBucket"],
      "Resource": ["arn:aws:s3:::lab-bkurs-$ACCT","arn:aws:s3:::lab-bkurs-$ACCT/*"]
    },
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-northeast-2:$ACCT:secret:rds!cluster-*"
    }
  ]
}
EOF
aws iam create-policy --policy-name LabAppPolicy --policy-document file://policy.json

# eksctl 한 줄로 ServiceAccount + IAM Role
eksctl create iamserviceaccount --cluster lab-eks \
  --namespace app --name api-sa \
  --role-name LabAPIRole \
  --attach-policy-arn arn:aws:iam::$ACCT:policy/LabAppPolicy \
  --override-existing-serviceaccounts --approve
```

## 4. Pod Identity (대안)

```bash
aws eks create-addon --cluster-name lab-eks --addon-name eks-pod-identity-agent

aws eks create-pod-identity-association --cluster-name lab-eks \
  --namespace app --service-account api-sa \
  --role-arn arn:aws:iam::$ACCT:role/LabAPIRole
```

## 5. External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::$ACCT:role/LabAPIRole

# SecretStore
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata: { name: aws-sm, namespace: app }
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef: { name: api-sa }
EOF

# ExternalSecret (RDS password)
RDS_SECRET=$(aws rds describe-db-clusters --db-cluster-identifier lab-mysql --query 'DBClusters[0].MasterUserSecret.SecretArn' --output text | awk -F'secret:' '{print $2}' | awk -F'-' '{print $1"-"$2"-"$3"-"$4"-"$5}')

kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-credentials, namespace: app }
spec:
  secretStoreRef: { name: aws-sm, kind: SecretStore }
  target: { name: db-credentials }
  data:
    - secretKey: username
      remoteRef: { key: $RDS_SECRET, property: username }
    - secretKey: password
      remoteRef: { key: $RDS_SECRET, property: password }
EOF
```

## 6. Spring Boot Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, namespace: app }
spec:
  replicas: 3
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      serviceAccountName: api-sa
      nodeSelector: { workload-type: general }     # Karpenter node
      containers:
        - name: api
          image: 123.dkr.ecr.ap-northeast-2.amazonaws.com/bkurs-be:v1.0.0
          ports:
            - { containerPort: 8080, name: http, protocol: TCP }
          env:
            - { name: AWS_REGION, value: ap-northeast-2 }
            - { name: SPRING_PROFILES_ACTIVE, value: prod }
            - { name: DB_HOST, value: lab-mysql.cluster-xxx.ap-northeast-2.rds.amazonaws.com }
            - { name: DB_NAME, value: bkurs }
            - { name: S3_BUCKET, value: lab-bkurs-ACCT_ID }
            - { name: DB_USER, valueFrom: { secretKeyRef: { name: db-credentials, key: username } } }
            - { name: DB_PASSWORD, valueFrom: { secretKeyRef: { name: db-credentials, key: password } } }
          resources:
            requests: { cpu: "500m", memory: "1Gi" }
            limits:   { cpu: "1",    memory: "2Gi" }
          livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 }, initialDelaySeconds: 60 }
          readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }
          securityContext:
            runAsNonRoot: true
            runAsUser: 10001
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: [ALL] }
```

## 7. Service + Ingress

```yaml
apiVersion: v1
kind: Service
metadata: { name: api, namespace: app }
spec:
  type: ClusterIP
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: app
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/readiness
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
```

## 8. HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api, namespace: app }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
    - type: Resource
      resource: { name: memory, target: { type: Utilization, averageUtilization: 70 } }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 600 }
    scaleUp: { stabilizationWindowSeconds: 30 }
```

## 9. NetworkPolicy

```yaml
# deny-all default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: deny-all, namespace: app }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# api allow
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-policy, namespace: app }
spec:
  podSelector: { matchLabels: { app: api } }
  ingress:
    - ports: [{ port: 8080 }]   # ALB → pod (target-type ip 라 직접)
  egress:
    - to: [{ namespaceSelector: { matchLabels: { name: kube-system } }, podSelector: { matchLabels: { k8s-app: kube-dns } } }]
      ports: [{ port: 53, protocol: UDP }]
    - to: [{ ipBlock: { cidr: 10.0.0.0/16 } }]   # RDS / VPC
      ports: [{ port: 3306 }]
    - to: [{ ipBlock: { cidr: 0.0.0.0/0 } }]     # S3 / SM (HTTPS)
      ports: [{ port: 443 }]
```

## 10. ExternalDNS + Route 53 (옵션)

```bash
eksctl create iamserviceaccount --cluster lab-eks \
  --namespace external-dns --name external-dns \
  --role-name LabExternalDNSRole \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess \
  --override-existing-serviceaccounts --approve

helm install external-dns external-dns/external-dns \
  -n external-dns --create-namespace \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns \
  --set provider=aws \
  --set txtOwnerId=lab-eks \
  --set domainFilters[0]=bkurs.com
```

Ingress 에 hostname 추가:
```yaml
rules:
  - host: api.bkurs.com
    ...
```

→ Route 53 자동 등록.

## 11. 검증

```bash
sleep 180

ALB=$(kubectl get ingress api -n app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$ALB/actuator/health
# {"status":"UP","components":{...}}

curl http://$ALB/api/v1/test-s3
# {"message":"OK","bucket":"lab-bkurs-..."}
```

## 12. Observability (Container Insights)

```bash
aws eks create-addon --cluster-name lab-eks \
  --addon-name amazon-cloudwatch-observability \
  --service-account-role-arn arn:aws:iam::$ACCT:role/EKS-ContainerInsightsRole
```

CloudWatch Console → Container Insights → lab-eks.

## 13. Clean up

```bash
kubectl delete -n app deployment api
kubectl delete -n app service api
kubectl delete -n app ingress api
kubectl delete -n app hpa api
kubectl delete -n app networkpolicy --all
kubectl delete -n app externalsecret db-credentials

aws rds delete-db-instance --db-instance-identifier lab-mysql-1 --skip-final-snapshot
aws rds delete-db-cluster --db-cluster-identifier lab-mysql --skip-final-snapshot
aws s3 rb s3://lab-bkurs-$ACCT --force

eksctl delete iamserviceaccount --cluster lab-eks --namespace app --name api-sa
aws iam delete-policy --policy-arn arn:aws:iam::$ACCT:policy/LabAppPolicy

# 실습 1 의 cluster 도 정리
eksctl delete cluster --name lab-eks --wait
```

## 14. 학습 포인트

- IRSA / Pod Identity 의 ServiceAccount 매핑
- External Secrets — Secrets Manager → K8s Secret 자동 sync
- Aurora Serverless v2 의 master password Secrets Manager 통합
- PSA `restricted` 수준 의 securityContext
- NetworkPolicy 의 egress (DNS / DB / 외부 API 만)
- ALB Ingress + Karpenter node + HPA 결합

## 15. 다음

- ▶ [[../eks]]
- ▶ [[../../ecs/practice/04-spring-boot-end-to-end]]
- ▶ [[../05-security/01-irsa-pod-identity]]

## 16. 참고

- [[../eks]]
- [[01-cluster-with-fargate]]
- [[02-karpenter-and-alb-ingress]]
- 모든 04-operations / 05-security
