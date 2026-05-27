---
title: "troubleshooting · Connection refused"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, tcp, firewall, port, security-group]
home_hub: "60-troubleshooting"
related: ["[[network]]"]
---

# Connection refused

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
curl: (7) Failed to connect to host: Connection refused
java.net.ConnectException: Connection refused
Error: connect ECONNREFUSED 1.2.3.4:8080
```

(timeout 과 다름 — refused 는 즉시 RST 응답)

## 원인

1. Service 가 listen 안 함 (process down)
2. 다른 port (config typo)
3. localhost vs 0.0.0.0 binding
4. Firewall / iptables 가 port drop
5. AWS Security Group inbound 미허용
6. K8s Service ↔ Pod label mismatch

## 해결

### 가장 먼저 — service 가 떠 있나
```bash
# Listening port 확인
ss -tlnp | grep 8080
netstat -an | grep :8080 | grep LISTEN
lsof -i :8080

# Process 확인
ps aux | grep myapp
systemctl status myapp
```

### Binding address 확인

```
0.0.0.0:8080   # 모든 interface (외부 OK)
127.0.0.1:8080 # localhost only — 외부 접근 X
192.168.1.1:8080 # 특정 interface
```

Spring Boot:
```yaml
server:
  address: 0.0.0.0   # default 0.0.0.0 — 외부 OK
  port: 8080
```

Express:
```js
app.listen(8080, '0.0.0.0', () => ...);
// 또는 '127.0.0.1' (localhost only)
```

### Firewall

```bash
# iptables
sudo iptables -L INPUT -v -n

# ufw (Ubuntu)
sudo ufw status
sudo ufw allow 8080/tcp

# firewalld (RHEL)
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

### AWS Security Group

```bash
# Inbound rule 확인
aws ec2 describe-security-groups --group-ids sg-xxx \
  --query 'SecurityGroups[0].IpPermissions'

# 허용 추가
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 8080 \
  --cidr 10.0.0.0/8
```

### K8s Service ↔ Pod

```bash
# Service 가 어떤 pod 를 가리키는지
kubectl get endpoints myservice
# 비어있으면 → selector 가 pod label 매칭 안 됨

# Service spec
kubectl describe service myservice
# Selector: app=myapp

# Pod label
kubectl get pods --show-labels | grep myapp

# Pod 에 직접
kubectl port-forward pod/myapp-xxx 8080:8080
curl localhost:8080
```

### Docker

```bash
# Container 가 떠 있나
docker ps | grep myapp

# Port mapping
docker port myapp
# 0.0.0.0:8080->8080/tcp

# Container 내부 접근
docker exec -it myapp curl localhost:8080

# 호스트에서 container IP 로
docker inspect myapp | grep IPAddress
```

## 진단 순서

```
1. service listen 중? → ss/netstat/lsof
   안 떠 있으면 → process / log 확인

2. 같은 host 에서 가능? → curl localhost:8080
   되면 → bind address / firewall 의심

3. 다른 host 에서 → telnet/nc -zv host 8080
   안 되면 → network path (SG / firewall / route)

4. Tracetoute → 어디서 drop
```

```bash
# 표준 진단
nc -zv host 8080            # connection 시도 + close
telnet host 8080            # interactive
curl -v --connect-timeout 5 http://host:8080

# Network path
mtr host
traceroute -T -p 8080 host

# Packet capture (host 측)
sudo tcpdump -i any port 8080
# SYN 가 오는지, RST 응답하는지
```

## 재발 방지

- Health check endpoint (/healthz) + monitoring
- Service deployment 후 sanity check (curl)
- Security Group / NACL 변경 시 PR review
- 알람: TCP listen port 사라짐 (process down)

## 관련

- [[network]] / [[dns-resolve-fail]] / [[../devops/k8s-crashloopbackoff]]
