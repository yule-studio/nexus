---
title: "ECS 03-4 — Service Discovery (Cloud Map) / Service Connect (mesh)"
kind: knowledge
status: current
created_at: 2026-05-22T13:56:00+09:00
tags: [aws, ecs, service, service-discovery, service-connect, cloud-map]
home_hub: ecs
---

# Service Discovery / Service Connect

> ECS service 간 통신 2 방법: **Cloud Map** (DNS) / **ECS Service Connect** (Envoy sidecar — modern, lite service mesh).

## 1. 비교

| 항목 | Cloud Map | Service Connect |
| --- | --- | --- |
| 동작 | DNS A/SRV + Route 53 private | Envoy proxy sidecar + DNS |
| metric / trace | ✗ | ✓ (CloudWatch) |
| retry / timeout / circuit breaker | ✗ | ✓ |
| traffic encryption | ✗ | ✓ (옵션, mTLS) |
| 권장 | 단순 internal | service mesh / metric |

> **신규: Service Connect 권장**.

## 2. Cloud Map (Service Discovery)

```bash
# Namespace
aws servicediscovery create-private-dns-namespace --name bkurs.internal --vpc vpc-xxx

# Cloud Map service
aws servicediscovery create-service --name api \
  --dns-config "NamespaceId=ns-xxx,DnsRecords=[{Type=A,TTL=10}]"

# ECS Service 연결
aws ecs create-service \
  --service-registries "registryArn=arn:aws:servicediscovery:...:service/srv-yyy" \
  ...
```

→ task 시작/종료 시 자동 register/deregister. `api.bkurs.internal` 로 호출.

> DNS round-robin — caching 으로 분포 불균등 가능.

## 3. ECS Service Connect (2022+)

### Cluster default namespace
```bash
aws ecs update-cluster --cluster bkurs-prod \
  --service-connect-defaults "namespace=arn:aws:servicediscovery:...:namespace/ns-xxx"
```

### Service activation
```bash
aws ecs create-service --service-name api \
  --service-connect-configuration "enabled=true,namespace=bkurs.internal,services=[{portName=http,clientAliases=[{port=8080,dnsName=api}],discoveryName=api}],logConfiguration={logDriver=awslogs,options={awslogs-group=/ecs/bkurs-be/sc,awslogs-region=ap-northeast-2,awslogs-stream-prefix=envoy}}"
```

### Task Definition — portMappings name
```json
{
  "containerDefinitions": [{
    "portMappings": [{
      "name": "http",
      "containerPort": 8080,
      "appProtocol": "http"
    }]
  }]
}
```

### 호출
다른 service: `http://api:8080/foo` 또는 `http://api.bkurs.internal:8080`.

## 4. Service Connect 의 기능

| 기능 | 효과 |
| --- | --- |
| 자동 retry / timeout | Envoy 가 5xx / timeout 시 재시도 |
| circuit breaker | 연속 실패 시 upstream 차단 |
| outlier detection | bad task 잠시 traffic 우회 |
| metric (CloudWatch) | RequestCount / 2xxCount / 5xxCount / TargetResponseTime / ProcessedBytes |
| access log | Envoy log → CloudWatch |
| mTLS (2024+) | task 간 자동 TLS |

## 5. mTLS

```bash
--service-connect-configuration "enabled=true,namespace=...,services=[...],tlsTermination={role=ARN of ACM private CA,serverAlternativeNames=[api.bkurs.internal]}"
```

## 6. App Mesh vs Service Connect

| 항목 | App Mesh | Service Connect |
| --- | --- | --- |
| 출시 | 2018 | 2022 |
| 복잡도 | 매우 높음 | 낮음 |
| 권장 | EOL 2026-09 | ★ |

## 7. ALB 와 함께

| 사용 | 누가 |
| --- | --- |
| 외부 → service | ALB (public) |
| service → service | Service Connect / Cloud Map |

## 8. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Cloud Map TTL 길음 | 5-10s |
| Envoy OOM | memoryReservation 64MB+ |
| log driver 누락 | logConfiguration 명시 |
| App Mesh + Service Connect 혼합 | Service Connect 만 |
| HTTP/2 / gRPC 안 됨 | appProtocol: http2 / grpc |

## 9. CDK

```typescript
const ns = new servicediscovery.PrivateDnsNamespace(this, 'NS', { name: 'bkurs.internal', vpc });

new ecs.FargateService(this, 'Api', {
  cluster, taskDefinition,
  serviceConnectConfiguration: {
    namespace: 'bkurs.internal',
    services: [{ portMappingName: 'http', port: 8080, dnsName: 'api' }],
    logDriver: ecs.LogDrivers.awsLogs({ streamPrefix: 'envoy' }),
  },
});
```

## 10. 다음

- ▶ [[05-load-balancer-integration]]
- ▶ [[../04-operations/02-container-insights]]
- ▶ [[../05-security/02-network-isolation]]

## 11. 영상 / 자료

| 자료 |
| --- |
| [ECS Service Connect — AWS](https://www.youtube.com/watch?v=Bz5ZWWiQUKw) |
| [공식 Service Connect](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html) |

## 12. 참고

- [[../ecs]]
- [[../../network/route53/route53]]
