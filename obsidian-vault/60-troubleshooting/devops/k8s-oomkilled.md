---
title: "troubleshooting · K8s OOMKilled (exit 137)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, kubernetes, memory, jvm, oom]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[../backend/oom-heap]]"]
---

# K8s OOMKilled — exit code 137

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```bash
kubectl describe pod <pod>
# State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

## 원인

container memory 가 `limits.memory` 도달 → OS kernel 이 OOM killer 실행 → SIGKILL (137 = 128 + 9).

## 해결

### Limits 확대 (메모리 여유 시)
```yaml
resources:
  requests:
    memory: 1Gi
  limits:
    memory: 2Gi    # 보통 requests 의 2x
```

### JVM heap percentage (Java)
```dockerfile
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"
```

→ container 의 75% 만 heap 으로. 나머지는 metaspace / direct memory / stack.

기본 (`-Xmx` 명시 없음): JVM 이 cgroup limit 인식 (Java 10+) — 25% default. 명시 권장.

### Node.js
```dockerfile
ENV NODE_OPTIONS="--max-old-space-size=1536"   # MiB
```

container limit 의 75% 정도.

### 메모리 leak 추적
- [[../backend/oom-heap]] — heap dump 분석
- Pod 의 memory metric 시계열 (Prometheus)
- 점진적 증가 → leak. 즉시 spike → workload.

### Vertical Pod Autoscaler (VPA)
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: myapp }
  updatePolicy: { updateMode: Auto }
```

→ 사용 패턴 학습 후 limits 자동 조정.

## 진단

```bash
# Pod 의 메모리 사용
kubectl top pod <pod>

# 노드 전체 메모리
kubectl describe node <node>

# OOM event 확인
kubectl get events --sort-by='.lastTimestamp' | grep -i oom

# 컨테이너 안 메모리 (cgroup)
kubectl exec <pod> -- cat /sys/fs/cgroup/memory.max
kubectl exec <pod> -- cat /sys/fs/cgroup/memory.current
```

## 재발 방지

- 모든 container 에 limits 명시 (without limits = unbounded)
- requests:limits 비율 1:2 가이드
- JVM / Node 옵션이 cgroup 인식
- 모니터링: `container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.85` 알람

## 관련

- [[devops]] / [[k8s-crashloopbackoff]] / [[../backend/oom-heap]] / [[../backend/oom-metaspace]]
