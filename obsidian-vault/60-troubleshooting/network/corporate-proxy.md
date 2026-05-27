---
title: "troubleshooting · Corporate proxy / firewall 차단"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, proxy, firewall, corporate]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[tls-unknown-authority]]"]
---

# Corporate proxy — `npm install ECONNREFUSED` / `pip` 차단

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
npm ERR! network connect ECONNREFUSED
pip install ... : Connection timed out
git clone ... : Connection refused
docker pull : net/http: TLS handshake timeout
```

- 회사 network 에서만 발생
- 집 / mobile hotspot 에서는 정상
- HTTPS 요청만 차단 (HTTP 는 OK), 또는 반대

## 해결

### Proxy 환경 변수
```bash
export HTTP_PROXY=http://proxy.corp.com:8080
export HTTPS_PROXY=http://proxy.corp.com:8080
export NO_PROXY=localhost,127.0.0.1,*.corp.com,10.0.0.0/8

# 인증 필요 시
export HTTP_PROXY=http://user:password@proxy.corp.com:8080
```

`~/.zshrc` / `~/.bashrc` 에 추가.

### Tool 별 설정

**npm**:
```bash
npm config set proxy http://proxy.corp.com:8080
npm config set https-proxy http://proxy.corp.com:8080
npm config set registry https://registry.npmjs.org/   # mirror 있으면 변경
```

**pip**:
```bash
pip install --proxy http://proxy.corp.com:8080 package

# 또는 ~/.pip/pip.conf
[global]
proxy = http://proxy.corp.com:8080
index-url = https://pypi.corp.com/simple/   # internal mirror
```

**git**:
```bash
git config --global http.proxy http://proxy.corp.com:8080
git config --global https.proxy http://proxy.corp.com:8080

# 특정 host 만
git config --global http.https://github.com.proxy http://proxy.corp.com:8080
```

**Docker**:
```bash
# ~/.docker/config.json
{
  "proxies": {
    "default": {
      "httpProxy": "http://proxy.corp.com:8080",
      "httpsProxy": "http://proxy.corp.com:8080",
      "noProxy": "localhost,*.corp.com"
    }
  }
}

# Docker daemon (systemd)
sudo systemctl edit docker
[Service]
Environment="HTTP_PROXY=http://proxy.corp.com:8080"
Environment="HTTPS_PROXY=http://proxy.corp.com:8080"
Environment="NO_PROXY=localhost,*.corp.com"
```

```dockerfile
# Build 시 proxy 전달
ARG HTTP_PROXY
ARG HTTPS_PROXY
ENV HTTP_PROXY=$HTTP_PROXY HTTPS_PROXY=$HTTPS_PROXY
RUN npm install
```

```bash
docker build --build-arg HTTP_PROXY=http://proxy.corp.com:8080 -t myapp .
```

**Maven**:
```xml
<!-- ~/.m2/settings.xml -->
<settings>
  <proxies>
    <proxy>
      <id>corp</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>proxy.corp.com</host>
      <port>8080</port>
      <nonProxyHosts>localhost|*.corp.com</nonProxyHosts>
    </proxy>
  </proxies>
</settings>
```

**Gradle**:
```
# ~/.gradle/gradle.properties
systemProp.http.proxyHost=proxy.corp.com
systemProp.http.proxyPort=8080
systemProp.https.proxyHost=proxy.corp.com
systemProp.https.proxyPort=8080
systemProp.http.nonProxyHosts=localhost|*.corp.com
```

### TLS interception 대응

회사 proxy 가 TLS inspection 하면 corporate CA 도 truststore 에 — [[tls-unknown-authority]] 참조.

### Allowlist 협상

```
회사 IT 에 요청:
- registry.npmjs.org
- pypi.org
- github.com / raw.githubusercontent.com
- docker.io / registry-1.docker.io / gcr.io / quay.io
- *.googleapis.com (gcloud)
- *.amazonaws.com (aws)
```

→ block 되는 endpoint 발견 시 IT 와 협상.

## 진단

```bash
# Proxy 통과 확인
curl -v http://example.com   # CONNECT / 200 응답

# Tool 별 config
npm config get proxy
git config --global --get http.proxy
docker info | grep -i proxy
mvn help:effective-settings

# 직접 접근 test (proxy 없이)
HTTPS_PROXY= curl -v https://registry.npmjs.org/
```

## 재발 방지

- Onboarding script (`bootstrap.sh`) 가 proxy 환경 자동 설정
- corporate registry mirror (Nexus / Artifactory) 사용 — 외부 직접 접근 회피
- dev container (`.devcontainer/devcontainer.json`) 에 proxy ARG

## 관련

- [[network]] / [[tls-unknown-authority]] / [[../devops/github-actions-secret-undefined]]
