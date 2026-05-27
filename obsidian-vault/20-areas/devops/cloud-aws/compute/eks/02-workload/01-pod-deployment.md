---
title: "EKS 02-1 — Pod / Deployment / StatefulSet / DaemonSet / Job"
kind: knowledge
status: current
created_at: 2026-05-22T16:10:00+09:00
tags: [aws, eks, workload, pod, deployment, statefulset, daemonset, job]
home_hub: eks
---

# EKS Workload — Pod / Deployment / etc

> 5 workload kind + replica + 배포 전략. K8s 의 핵심.

## 1. 5 workload kind

| Kind | 용도 | replica 보장 | identity |
| --- | --- | --- | --- |
| **Pod** | 1+ container, 임시 | ✗ | ephemeral |
| **Deployment** | stateless replica | ✓ | random hash |
| **StatefulSet** | stateful replica | ✓ | stable name + PVC |
| **DaemonSet** | node 마다 1 pod | per-node | per-node |
| **Job / CronJob** | 일회성 / 스케줄 | ✓ (작업 완료 보장) | ephemeral |

## 2. Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  namespace: prod
spec:
  containers:
    - name: app
      image: 123.dkr.ecr.ap-northeast-2.amazonaws.com/bkurs-be:v1
      ports: [{ containerPort: 8080 }]
      env:
        - name: PROFILE
          value: prod
      resources:
        requests: { cpu: "500m", memory: "1Gi" }
        limits:   { cpu: "1",    memory: "2Gi" }
      livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 }, initialDelaySeconds: 60 }
      readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }
```

> 직접 Pod 만들면 죽으면 끝. 보통 Deployment 안에서 사용.

## 3. Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: app
          image: 123.dkr.ecr...:v1
          # 위와 동일
```

| 옵션 | 의미 |
| --- | --- |
| `replicas` | 유지할 pod 수 |
| `strategy.type` | RollingUpdate (default) / Recreate |
| `maxUnavailable` | 0 = 무중단 |
| `maxSurge` | 1 = 1개 추가까지 |
| `revisionHistoryLimit` | rollback 위한 이전 ReplicaSet 보관 |

## 4. Rolling Update

```bash
# Image 변경
kubectl set image deployment/api app=123.dkr.ecr...:v2

# 진행 확인
kubectl rollout status deployment/api

# 이전 revision 보기
kubectl rollout history deployment/api

# Rollback
kubectl rollout undo deployment/api --to-revision=3
```

## 5. Canary / Progressive — Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      canaryService: api-canary
      stableService: api-stable
      trafficRouting:
        alb:
          ingress: api-ingress
          servicePort: 80
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis:
            templates: [{ templateName: success-rate }]
        - setWeight: 50
        - pause: {}
```

ALB Controller + Argo Rollouts → ECS CodeDeploy Blue-Green 와 비슷.

## 6. StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
    spec:
      containers:
        - name: db
          image: postgres:16
          ports: [{ containerPort: 5432 }]
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: ebs-gp3
        resources: { requests: { storage: 100Gi } }
```

특징:
- Pod 이름: `db-0`, `db-1`, `db-2` (고정)
- PVC: pod 마다 별도 (db-data-db-0, db-data-db-1, ...)
- 순서 보장 (1→2→3 시작 / 3→2→1 종료)
- StableNetworkID — Headless Service + per-pod DNS

> production DB 는 보통 RDS / Aurora ([[../../../database]]). StatefulSet 는 Kafka / ZK / Cassandra / Elasticsearch.

## 7. DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  selector: { matchLabels: { app: fluent-bit } }
  template:
    metadata: { labels: { app: fluent-bit } }
    spec:
      tolerations:
        - operator: Exists
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.0
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
      volumes:
        - name: varlog
          hostPath: { path: /var/log }
```

활용: 로그 수집 / monitoring agent / kube-proxy / CNI.

> Fargate 는 DaemonSet 안 됨 — EC2 node 만.

## 8. Job / CronJob

```yaml
# 일회성
apiVersion: batch/v1
kind: Job
metadata: { name: migrate }
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: 123.dkr.ecr...:v1
          command: ["java","-jar","/app/migrate.jar"]
---
# 매일 03:00
apiVersion: batch/v1
kind: CronJob
metadata: { name: backup }
spec:
  schedule: "0 3 * * *"
  timeZone: "Asia/Seoul"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: 123.dkr.ecr.../backup:v1
```

## 9. HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 60 }
    - type: Resource
      resource:
        name: memory
        target: { type: Utilization, averageUtilization: 70 }
    - type: External
      external:
        metric: { name: alb_request_count_per_target }
        target: { type: AverageValue, averageValue: "500" }
```

→ Metric Server 필요. external metric 은 CloudWatch Adapter 또는 KEDA.

## 10. VPA / KEDA

| | 역할 |
| --- | --- |
| **VPA** | pod 의 resource request 자동 조정 (right-sizing) |
| **KEDA** | 30+ source (SQS / Kafka / Cron / Prometheus / CloudWatch) → HPA |

## 11. Pod Disruption Budget (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: api } }
```

→ node drain / cluster upgrade 시 동시 종료 한도. 무중단 보장.

## 12. Affinity / Anti-Affinity / Topology Spread

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector: { matchLabels: { app: api } }
        topologyKey: topology.kubernetes.io/zone

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector: { matchLabels: { app: api } }
```

→ AZ 별 분산, 같은 node 회피.

## 13. Init Container / Sidecar

```yaml
initContainers:
  - name: migrate
    image: flyway:10
    command: ["flyway","migrate"]
containers:
  - name: app
    image: bkurs-be:v1
  - name: envoy
    image: envoyproxy/envoy:v1.30
```

→ init 끝나야 main 시작. sidecar = 같은 lifecycle, network 공유.

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| 직접 Pod (controller 없음) | Deployment / StatefulSet |
| resources 미지정 → noisy neighbor | requests + limits |
| livenessProbe 없음 → 죽어도 안 재시작 | health endpoint |
| PDB 없음 → upgrade 시 down | minAvailable 설정 |
| HPA + Karpenter 미연동 | 둘 다 활성 |
| Fargate + DaemonSet | EC2 만 |

## 15. 다음

- ▶ [[02-service-and-ingress]]
- ▶ [[03-storage-csi]]
- ▶ [[../04-operations/01-autoscaling-cas-karpenter]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [K8s workloads 공식](https://kubernetes.io/docs/concepts/workloads/) |
| [Argo Rollouts](https://argo-rollouts.readthedocs.io/) |
| [KEDA](https://keda.sh) |

## 17. 참고

- [[../eks]]
- [[../01-fundamentals/03-node-types]]
- [[../../ecs/03-service/01-service-overview]]
