---
title: "troubleshooting · K8s CrashLoopBackOff"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, kubernetes, pod, probe, restart]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[k8s-oomkilled]]"]
---

# K8s — `CrashLoopBackOff`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```bash
kubectl get pods
NAME              READY   STATUS             RESTARTS   AGE
myapp-7d8b-x1z2   0/1     CrashLoopBackOff   12         15m
```

## 원인 (단계별)

1. Container 시작 실패 → exit
2. K8s 가 restart
3. exponentially backoff (10s, 20s, 40s, ... max 5min)
4. `Back-off restarting failed container` 상태

근본 원인:
- 앱 startup error (config / secret 누락)
- 잘못된 CMD / ENTRYPOINT
- Liveness probe 가 startup 시간 안 기다림
- OOM (다른 issue — [[k8s-oomkilled]])
- Image 가 옛 버전

## 해결

### 첫 단계 — 로그 확인
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # 이전 instance log (현재 죽음)
kubectl describe pod <pod-name>      # events 섹션
```

자주 발견:
- `secret not found`
- `Connection refused: localhost:5432`
- `Cannot bind to port 80: permission denied` (non-root user)
- `OOMKilled` (exit code 137)
- `Configuration error` / `panic:`

### Startup probe (긴 init 시간 보호)
```yaml
startupProbe:
  httpGet: { path: /health, port: 8080 }
  periodSeconds: 5
  failureThreshold: 60    # 5 * 60 = 5분 까지 startup 허용

livenessProbe:
  httpGet: { path: /health/live, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3      # 30초 unhealthy 후 restart

readinessProbe:
  httpGet: { path: /health/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 3
```

→ startup 30초+ Spring Boot 는 startupProbe 필수.

### Init container 로 dependency 확인
```yaml
initContainers:
  - name: wait-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting; sleep 2; done']
```

### Resource 부족 — limits 명시
```yaml
resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: 1000m
```

→ requests 가 너무 작으면 scheduling 후 throttle. limits 가 작으면 OOMKilled.

### Crash 직후 shell 접근 — debug image
```yaml
spec:
  containers:
    - name: main
      command: ['sleep', '3600']   # 임시 — 정상 명령 대신 sleep
```

또는 ephemeral container:
```bash
kubectl debug -it <pod> --image=busybox --target=main -- sh
```

## 진단

```bash
# 가장 흔한 패턴
kubectl describe pod <pod>
# → Events 섹션:
#    Liveness probe failed: HTTP probe failed with statuscode: 500
#    Back-off restarting failed container

kubectl logs <pod> -c <container> --previous

# 이전 N restart 의 reason
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState}'

# Cluster-wide 비정상 pod
kubectl get pods -A | grep -E '(CrashLoop|Error|Pending)'
```

## 재발 방지

- 모든 deployment 에 startup / liveness / readiness probe (3개 분리)
- ConfigMap / Secret 변경 시 rolling restart 검증
- e2e: helm install 후 `kubectl wait --for=condition=ready` 검증
- 알람:
  ```yaml
  - alert: PodCrashLoop
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0.1
  ```

## 관련

- [[devops]] / [[k8s-oomkilled]] / [[k8s-imagepullbackoff]] / [[ecs-task-failed-to-start]]
