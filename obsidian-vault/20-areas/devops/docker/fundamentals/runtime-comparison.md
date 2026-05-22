---
title: "Container runtime — runc / containerd / gVisor / kata"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:17:00+09:00
tags: [devops, docker, runtime, containerd]
---

# Container runtime — runc / containerd / gVisor / kata

**[[docker|↑ docker]]**

---

## 1. layer

```
[kubectl / docker CLI]
       ↓
[Container Runtime Interface (CRI)]
       ↓
[high-level runtime]    containerd / CRI-O
       ↓
[low-level runtime]     runc / crun / gVisor / kata
       ↓
[Linux kernel]          namespace / cgroup / seccomp / AppArmor
```

---

## 2. high-level runtime

| | 무엇 |
| --- | --- |
| **containerd** | CNCF, Docker 의 backbone, k8s default |
| **CRI-O** | k8s 전용, RedHat |
| **Docker Engine** | dockerd (legacy in k8s, 1.20+ deprecated) |

→ Kubernetes 의 default = **containerd**.

---

## 3. low-level runtime

| | 무엇 | trade-off |
| --- | --- | --- |
| **runc** | OCI 표준, 가장 흔함 | ★ default |
| **crun** | Rust + C, 빠름 | runc 호환 |
| **gVisor** | userland kernel (Google) | 보안 ↑ / 성능 ↓ |
| **Kata Containers** | VM 기반 (KVM) | 강력 격리 / 무거움 |
| **Firecracker** | AWS Lambda 의 VM | 보안 + 빠른 boot |
| **youki** | Rust runtime | runc 호환 |

---

## 4. runc (★ 표준)

```
- OCI 표준 (Open Container Initiative)
- C++ → Go rewrite
- Linux namespace + cgroup
- 가장 가벼움 (single binary)
- 거의 모든 container 의 backbone

장점: 빠름 / mature / 호환
단점: kernel 공유 (host 와 같은 kernel)
```

---

## 5. containerd (★ k8s)

```bash
# k8s 의 default
# Docker Desktop 도 내부에서 containerd 사용

# 직접 사용 (crictl - CRI client)
sudo crictl ps
sudo crictl images
sudo crictl logs <id>

# nerdctl (Docker CLI like)
sudo nerdctl run alpine echo hello
sudo nerdctl ps
sudo nerdctl pull nginx
```

→ Docker 의 대체로 사용 가능 (Lima / Rancher Desktop / colima).

---

## 6. nerdctl (★ Docker 대안)

```bash
# Docker 와 똑같은 CLI
nerdctl run -it alpine sh
nerdctl ps
nerdctl images
nerdctl build .
nerdctl compose up

# 차이:
nerdctl 은 containerd 직접
nerdctl 의 multi-platform / lazy-pull 등 추가 기능
```

```bash
# Lima (Mac 에서 Linux + containerd)
brew install lima
lima
lima nerdctl run alpine

# Rancher Desktop
brew install --cask rancher
```

---

## 7. podman (★ Docker 대안)

```bash
# rootless / daemonless
brew install podman
podman machine init
podman machine start

podman run -it alpine sh
podman ps
podman build .
podman-compose up
```

```
podman 의 강점:
  - daemonless (각 container 가 별도 process)
  - rootless (default)
  - systemd 친화 (각 container → unit)
  - Docker CLI 호환
  - 자동 sigterm
  - pod (k8s like grouping)

약점:
  - macOS 의 약간 어색
  - 일부 Docker 기능 차이
```

→ enterprise / 보안 환경.

---

## 8. gVisor (★ security)

```
sandbox = userland kernel:
  container → gVisor → host kernel (제한)
  
syscall 가 gVisor 에서 처리 (host 안 부름).
→ kernel exploit 격리.

설정:
  RuntimeClass: gvisor
  k8s 의 pod 가 RuntimeClass 명시.
```

```yaml
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: gvisor    # 일부 pod 만 sandbox
  containers:
    - name: untrusted
      image: untrusted/code:1.0
```

→ multi-tenant SaaS / user-submitted code.

→ 성능 X (syscall overhead 큼).

---

## 9. Kata Containers (★ strong isolation)

```
VM (KVM) + container 결합:
  container → light VM → host
  
강력한 격리 (kernel 분리).
시작 시간 ↑ (수백ms).

설정:
  RuntimeClass: kata-clh (cloud-hypervisor)
```

→ untrusted code / 강한 compliance / 금융.

---

## 10. Firecracker (★ AWS Lambda)

```
microVM:
  - 125ms 부팅
  - 5MB memory overhead
  - KVM
  - AWS Lambda / Fargate 의 backbone

OSS:
  https://firecracker-microvm.github.io/
```

→ FaaS 의 표준.

---

## 11. 보안 layer

```
container 의 격리:

namespace:
  - pid (process)
  - net (network)
  - mnt (filesystem)
  - uts (hostname)
  - ipc (System V IPC)
  - user (uid 매핑)
  - cgroup
  - time (Linux 5.6+)

cgroup:
  - cpu / memory / disk / network 제한

seccomp:
  - syscall whitelist / blacklist
  - default profile = 50+ syscall block

AppArmor / SELinux:
  - MAC (mandatory access control)
  - 강제 권한 정책

capabilities:
  - CAP_NET_ADMIN / CAP_SYS_ADMIN / ...
  - default = drop almost all, allow specific

→ 모든 layer 가 protective. 1개 fail 도 다른 layer 가 막아줌.
```

---

## 12. 결정 매트릭스

```
일반 production:
  containerd + runc + seccomp default + AppArmor + drop caps

multi-tenant SaaS (user code 실행):
  containerd + gVisor 또는 Kata

cloud-native FaaS:
  Firecracker

dev / homelab:
  Docker Desktop / Rancher / Podman / Lima
```

---

## 13. WASM (★ 2024+ 새 추세)

```
WebAssembly 의 server-side:
  container 보다 작음 (KB)
  매우 빠름 (us)
  강력한 격리
  language-agnostic

runtime:
  - WasmEdge
  - wasmtime
  - WAMR

k8s 통합:
  - containerd + WasmEdge / wasmtime shim
  - SpinKube
```

→ edge / serverless / 매우 빠른 boot 필요 시. 아직 ecosystem 발전 중.

---

## 14. monitoring (runtime 단)

```bash
# 모니터링 도구
- ctop (container top)
- crictl stats
- nerdctl stats
- Sysdig
- Falco (runtime security)
- Tracee (eBPF)
```

```yaml
# Falco rule (sandbox runtime detection)
- rule: Container escape
  desc: container 안에서 host file 접근
  condition: open_read and fd.name startswith /host
```

---

## 15. 함정

1. **dockerd 가 k8s 에서 deprecated** — containerd 로 migrate.
2. **Docker Desktop 사용** — 회사 license 정책 확인.
3. **rootless 의 일부 호환성** — port < 1024 등.
4. **gVisor 의 성능** — syscall-heavy workload 부적합.
5. **Kata 의 부팅 시간** — short-lived task 부적합.
6. **seccomp 의 default 끔** — escape 위험 ↑.
7. **CAP_SYS_ADMIN** — 거의 root.

---

## 16. 관련

- [[docker|↑ docker]]
- [[security]]
- [[../kubernetes/concepts|↑ k8s]]
- [[../security-ops/supply-chain-security|↗ supply chain]]
