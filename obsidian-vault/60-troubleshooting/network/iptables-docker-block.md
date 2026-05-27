---
title: "troubleshooting · iptables / Docker traffic drop"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, iptables, docker, firewall, ufw]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[connection-refused]]"]
---

# iptables / nftables 가 traffic drop

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

- Docker container 의 port mapping (`-p 8080:8080`) 가 외부에서 접근 안 됨
- ufw 가 enable 인데 docker port 우회됨 (위험)
- iptables 룰 잘못 변경 후 traffic block

## 원인

Docker 가 iptables `DOCKER` chain 자동 관리 → 직접 변경 시 충돌. 또한 Docker 가 ufw 우회 (FORWARD chain 직접 조작).

## 해결

### Docker iptables 작동

```bash
sudo iptables -L -n -v
# Chains:
# INPUT, OUTPUT, FORWARD, DOCKER, DOCKER-USER, DOCKER-INGRESS, DOCKER-ISOLATION-*
```

`DOCKER-USER` chain — 사용자가 룰 추가 가능 (Docker 가 안 만짐):

```bash
# 특정 IP 만 container 접근 허용
sudo iptables -I DOCKER-USER -i eth0 ! -s 1.2.3.4 -j DROP

# 또는 ufw 호환
sudo ufw allow from 1.2.3.4 to any port 8080
```

### ufw + Docker 안전 설정

```
# /etc/ufw/after.rules 끝에 추가
*filter
:DOCKER-USER - [0:0]
:ufw-user-input - [0:0]

-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

-A DOCKER-USER -j ufw-user-input

-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

COMMIT
```

```bash
sudo systemctl restart ufw
sudo systemctl restart docker
```

### Docker iptables disable (manual)

```json
// /etc/docker/daemon.json
{
  "iptables": false
}
```

→ Docker 가 iptables 안 만짐. NAT / FORWARD 룰 직접 작성 필요. 안 권장.

### Container 가 외부 못 나감

```bash
# host 의 ip_forward
sysctl net.ipv4.ip_forward
# 1 이어야 함 (Docker daemon 가 자동 활성화)

# 수동
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

### iptables 룰 backup / restore

```bash
# Backup
sudo iptables-save > /tmp/iptables.bak

# 실수 후 복구
sudo iptables-restore < /tmp/iptables.bak
```

```bash
# 전체 정리 (긴급 — 모든 룰 제거)
sudo iptables -F
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo systemctl restart docker   # Docker 룰 재생성
```

## fail2ban 충돌

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
# Banned IP 확인

sudo fail2ban-client set sshd unbanip 1.2.3.4
```

## 진단

```bash
# 룰 시각화
sudo iptables -L -n -v --line-numbers
sudo iptables -t nat -L -n -v

# 어느 룰이 hit
sudo iptables -L INPUT -v -n | sort -k1 -n -r | head

# 패킷 추적 (어디서 drop)
sudo iptables -t raw -A PREROUTING -p tcp --dport 8080 -j TRACE
sudo dmesg | tail -50

# 또는 tcpdump 양쪽 (호스트, container)
sudo tcpdump -i any -n port 8080
sudo nsenter -t <container_pid> -n tcpdump -i any port 8080
```

## 재발 방지

- Docker 와 ufw 같이 쓸 때 `DOCKER-USER` chain 표준
- iptables 직접 변경 금지 (Docker 가 관리)
- 변경 전 backup
- 알람: docker port 접근 실패 (health check)

## 관련

- [[network]] / [[connection-refused]] / [[vpc-public-internet]]
