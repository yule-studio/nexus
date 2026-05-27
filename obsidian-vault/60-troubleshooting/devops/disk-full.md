---
title: "troubleshooting · Disk full — no space left on device"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, linux, disk, docker, log, journald]
home_hub: "60-troubleshooting"
related: ["[[devops]]"]
---

# `no space left on device`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
write /var/log/app.log: no space left on device
docker: failed to register layer: no space left on device
```

- 어떤 서비스도 write 실패 (DB, log, image pull)
- `df -h` 의 root / Docker volume 100%

## 원인 (4가지 흔함)

1. Docker overlay (`/var/lib/docker`) 누적
2. journald log (`/var/log/journal`) 무한 누적
3. App log (`/var/log/myapp.log`) rotation 안 됨
4. Tmp file (`/tmp`) 미정리

## 해결

### 즉시 — 공간 확보
```bash
# Docker 정리 (가장 효과적)
docker system prune -af --volumes
# stopped container + 사용 안 하는 image + volume 모두

# journald
journalctl --vacuum-size=500M
journalctl --vacuum-time=7d

# 큰 파일 찾기
du -h / 2>/dev/null | sort -hr | head -20
sudo find / -type f -size +500M 2>/dev/null

# tmp 정리
sudo find /tmp -type f -atime +7 -delete
```

### Log rotation (logrotate)
```
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

```bash
logrotate -d /etc/logrotate.d/myapp   # dry run
logrotate -f /etc/logrotate.d/myapp   # force
```

### Docker log driver — size limit
```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  }
}
```

```bash
systemctl restart docker
```

→ container 별 log 500MB 한도 (100MB × 5 file).

### journald 한도
```ini
# /etc/systemd/journald.conf
[Journal]
SystemMaxUse=2G
SystemMaxFileSize=200M
MaxRetentionSec=1week
```

```bash
systemctl restart systemd-journald
```

### K8s — node disk pressure
```yaml
# kubelet config
evictionHard:
  imagefs.available: "10%"
  nodefs.available: "10%"
```

→ disk 압박 시 pod eviction. ECS / EKS 는 적절한 EBS 크기 선택.

## CloudWatch / Prometheus 모니터링

```
- alert: DiskSpaceLow
  expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.15
  for: 10m

- alert: DiskInodeLow
  expr: (node_filesystem_files_free / node_filesystem_files) < 0.10
```

→ inode 별도 모니터 (작은 파일 많을 때 size OK 인데 inode 고갈).

## 진단

```bash
# Disk 사용량
df -h
df -i           # inode

# Mount point 별
du -sh /* 2>/dev/null | sort -hr
du -sh /var/lib/docker

# Docker 상세
docker system df
docker system df -v

# 가장 큰 directory
du -h --max-depth=2 /var | sort -hr | head

# 열린 파일이 잡고 있는 disk (deleted 인데 close 안 됨)
lsof | grep deleted | awk '{print $7,$NF}' | sort -k1 -n | tail
```

## 재발 방지

- Log rotation 표준 (logrotate.d / Docker daemon.json)
- journald 한도 설정
- 모니터링: disk available + inode 양쪽
- Container 의 log 는 stdout/stderr → docker / k8s 가 관리

## 관련

- [[devops]] / [[k8s-imagepullbackoff]] / [[../backend/file-descriptor-leak]]
