---
title: "EKS 02-2 — Service / Ingress (ALB Controller / Gateway API)"
kind: knowledge
status: current
created_at: 2026-05-22T16:12:00+09:00
tags: [aws, eks, workload, service, ingress, alb, gateway-api]
home_hub: eks
---

# EKS Service / Ingress

> Service (L4) + Ingress (L7) + Gateway API (modern). AWS Load Balancer Controller 가 ALB / NLB 자동 생성.

## 1. Service 4 type

| Type | 동작 |
| --- | --- |
| **ClusterIP** (default) | cluster 안 가상 IP. pod-to-pod 접근. |
| **NodePort** | 모든 node 의 동일 port 에 open (30000-32767). |
| **LoadBalancer** | NLB / ALB 자동 생성 (Controller 필요). |
| **ExternalName** | DNS CNAME 만 (외부 서비스 alias). |

## 2. ClusterIP — 표준 내부 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  selector: { app: api }
  ports:
    - port: 80
      targetPort: 8080
      name: http
```

DNS 자동:
```
api.default.svc.cluster.local         (FQDN)
api                                    (same namespace)
api.default                            (cross-namespace)
```

Headless Service (StatefulSet 용):
```yaml
spec:
  clusterIP: None    # 직접 pod IP 반환
```

## 3. LoadBalancer — NLB (L4)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tcp-api
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-attributes: load_balancing.cross_zone.enabled=true
spec:
  type: LoadBalancer
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
```

→ AWS Load Balancer Controller 가 자동 NLB 생성, target group 등록.

## 4. AWS Load Balancer Controller 설치

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=bkurs-prod \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::...:role/ALBControllerRole
```

> IRSA → IAM Role + OIDC ([[../05-security/01-irsa-pod-identity]]).

## 5. Ingress — ALB (L7)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bkurs-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}, {"HTTP": 80}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...:certificate/abc
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/readiness
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type":"redirect","RedirectConfig":{"Protocol":"HTTPS","Port":"443","StatusCode":"HTTP_301"}}'
    alb.ingress.kubernetes.io/group.name: bkurs            # 여러 Ingress 가 1 ALB 공유
spec:
  ingressClassName: alb
  rules:
    - host: api.bkurs.com
      http:
        paths:
          - path: /api/v1/auth
            pathType: Prefix
            backend: { service: { name: auth-api,  port: { number: 80 } } }
          - path: /api/v1/study
            pathType: Prefix
            backend: { service: { name: study-api, port: { number: 80 } } }
    - host: admin.bkurs.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: admin, port: { number: 80 } } }
```

→ ALB Controller 가 자동:
- ALB 1개 (group.name=bkurs)
- listener 443 (HTTPS) + 80 (HTTP→HTTPS redirect)
- 4 target group (auth/study/admin/...)
- Route 53 (ExternalDNS controller 별도 시 자동)

## 6. ExternalDNS

```bash
helm install external-dns external-dns/external-dns \
  -n kube-system \
  --set provider=aws \
  --set txtOwnerId=bkurs-prod \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::...:role/ExternalDNSRole
```

→ Ingress hostname 을 Route 53 에 자동 등록.

## 7. NLB target type — IP vs Instance

| | IP | Instance |
| --- | --- | --- |
| Target | Pod IP | EC2 instance ID + NodePort |
| awsvpc 필수 | ✓ (자연스러움) | ✗ |
| Fargate | ✓ | ✗ |
| Karpenter | ✓ | ✓ |
| 권장 | ★ | legacy |

## 8. Gateway API (modern)

K8s 의 차세대 ingress 표준:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: bkurs }
spec:
  gatewayClassName: alb
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        certificateRefs: [{ name: bkurs-cert }]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: auth-route }
spec:
  parentRefs: [{ name: bkurs }]
  hostnames: ["api.bkurs.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /api/v1/auth } }]
      backendRefs: [{ name: auth-api, port: 80 }]
```

장점:
- Multi-protocol (HTTP / TCP / gRPC / WebSocket)
- Cross-namespace 권한 분리 (Gateway = network admin / Route = app team)
- 표준 (vendor 의존 ↓)

## 9. Service Mesh (Istio / Linkerd / Cilium)

east-west traffic 의 retry / timeout / mTLS / observability:

| | Istio | Linkerd | Cilium Service Mesh |
| --- | --- | --- | --- |
| Sidecar | Envoy | Linkerd2-proxy | eBPF (sidecar-less) |
| 학습 곡선 | ↑↑ | ↓ | 중 |
| 성능 overhead | 5~10% | 작음 | 거의 0 |
| 권장 | 큰 cluster | 단순 | modern |

> ECS 의 Service Connect 와 동일한 역할.

## 10. ALB share — group.name

```yaml
alb.ingress.kubernetes.io/group.name: bkurs
alb.ingress.kubernetes.io/group.order: '10'
```

→ 여러 Ingress 가 1 ALB. 비용 절감.

## 11. NLB share

```yaml
service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-aaa,eipalloc-bbb,eipalloc-ccc
```

→ Static IP 보장 (3 AZ).

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| LoadBalancer 마다 ALB / NLB 별도 (비용 ↑) | Ingress + group.name |
| ALB target type instance (Fargate 안 됨) | ip |
| ACM 인증서 region 불일치 | 같은 region |
| ExternalDNS 미설치 → DNS 수동 | ExternalDNS |
| Gateway API + ALB Controller 미지원 | Controller 최신 / Envoy Gateway 도 가능 |
| Healthcheck path 잘못 (404) | 정확한 path + 200 응답 |

## 13. 다음

- ▶ [[03-storage-csi]]
- ▶ [[../03-networking/01-vpc-cni-and-pod-network]]
- ▶ [[../05-security/01-irsa-pod-identity]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [AWS Load Balancer Controller 공식](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) |
| [Gateway API 공식](https://gateway-api.sigs.k8s.io/) |
| [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) |

## 15. 참고

- [[../eks]]
- [[../../network/alb/alb]]
- [[../../ecs/03-service/05-load-balancer-integration]]
