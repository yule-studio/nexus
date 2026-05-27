---
title: "troubleshooting · SSH key permission too open"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, ssh, security, key, permission]
home_hub: "60-troubleshooting"
related: ["[[devops]]"]
---

# SSH `Permissions 0644 for id_rsa are too open`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0644 for '/Users/foo/.ssh/id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
```

```
Load key "/home/runner/.ssh/id_rsa": bad permissions
Permission denied (publickey).
```

## 원인

SSH client 가 private key 파일의 permission 검증. group / other 가 read 권한 있으면 거절 (보안).

## 해결

### 권한 수정
```bash
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/known_hosts
```

### Owner 확인
```bash
ls -la ~/.ssh/
# -rw-------  1 myuser mygroup  3243 Jan  1 12:00 id_rsa
# 만약 root 소유면:
sudo chown $USER:$USER ~/.ssh/id_rsa
```

### CI 환경 (GitHub Actions / GitLab)
```yaml
- name: Setup SSH
  run: |
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
    chmod 600 ~/.ssh/id_rsa
    ssh-keyscan -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts
```

→ secret 으로 받은 key 는 항상 chmod.

### `webfactory/ssh-agent` action (권장)
```yaml
- uses: webfactory/ssh-agent@v0.9.0
  with:
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

→ permission 자동 처리. agent forwarding.

### Windows (WSL) 함정
```bash
# Windows file system 마운트의 permission 인식 다름
# 해결: WSL2 native filesystem 으로 이동
cp /mnt/c/.ssh/id_rsa ~/.ssh/
chmod 600 ~/.ssh/id_rsa
```

### Modern key — Ed25519 권장
```bash
ssh-keygen -t ed25519 -C "you@example.com"
# 더 짧고 빠름. RSA 보다 권장.

# RSA 사용 시 최소 4096
ssh-keygen -t rsa -b 4096
```

## SSH config 활용
```
# ~/.ssh/config
Host prod-bastion
    HostName bastion.example.com
    User ec2-user
    IdentityFile ~/.ssh/prod_key
    Port 22
    IdentitiesOnly yes
    ForwardAgent no

Host prod-internal
    HostName 10.0.1.10
    User ec2-user
    ProxyJump prod-bastion
```

```bash
ssh prod-internal   # bastion 거쳐 자동
```

## 진단

```bash
# Verbose mode — 어디서 실패
ssh -vvv user@host

# 사용 중 key
ssh-add -L

# Key fingerprint
ssh-keygen -lf ~/.ssh/id_rsa.pub
```

## 재발 방지

- ssh-agent 사용 (manual chmod 회피)
- SSH key 는 git repo / chat 에 절대 commit X (`.gitignore` 의무)
- 1Password / Bitwarden SSH agent 통합
- 모든 prod key 는 hardware key (YubiKey) 또는 IAM (SSM Session Manager) 로 대체

## SSM Session Manager — SSH 대체 (AWS)
```bash
aws ssm start-session --target i-1234567890
# SSH key 없이 IAM 권한으로 instance shell
```

→ key management 불필요. audit log 자동.

## 관련

- [[devops]] / [[github-actions-secret-undefined]]
