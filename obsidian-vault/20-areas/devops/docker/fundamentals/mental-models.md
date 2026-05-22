---
title: "Docker mental models — 컨테이너 = 프로세스 / layer / OCI"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T11:00:00+09:00
tags: [devops, docker, mental-models]
home_hub: devops
related:
  - "[[docker]]"
  - "[[concepts]]"
  - "[[buildkit-deep]]"
  - "[[production-patterns]]"
  - "[[../linux/linux-mental-models-for-devops]]"
---

# Docker mental models — 컨테이너 = 프로세스 / layer / OCI

**[[docker|↑ docker]]**

---

## 1. 목적

본 문서는 Docker 의 동작을 명령 사용법이 아니라 **운영체제 / 파일시스템 / 빌드 시스템 의 구성 요소** 로 정의한다.

본 문서가 정의하는 것:
- 컨테이너의 OS 레벨 구성 (process / namespace / cgroup / capability)
- image 의 저장 모델 (layer / OCI manifest / content-addressable store)
- build 의 단위 (layer cache 키 / context / multi-stage)
- runtime 인터페이스 계층 (Docker Engine / containerd / runc / OCI)
- 위 4 가지 모델에서 파생되는 운영 결정 (효율 / 보안 / 디버깅)

본 문서가 정의하지 않는 것:
- Dockerfile 문법 — [[concepts]], [[dockerfile-best-practices]]
- compose / swarm 운영 — [[compose]]
- registry / 배포 — [[registry]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| runtime | Linux container (containerd / runc / Docker Engine) |
| build | BuildKit (Docker 23+ default) 기반 build graph |
| 표준 | OCI Image Spec v1.1, OCI Runtime Spec v1.1, OCI Distribution Spec v1.1 |
| 제외 | Windows container, gVisor / Firecracker / Kata 등 sandboxed runtime ([[runtime-comparison]]) |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **container** | namespace + cgroup 으로 격리된 host 의 process tree. |
| **image** | container 의 root filesystem 과 실행 메타데이터를 layer 의 stack 으로 정의한 immutable artifact. |
| **layer** | filesystem 변경의 단위. content-addressable (SHA256) 하며 image 간 공유된다. |
| **OCI** | Open Container Initiative. image / runtime / distribution 의 3 표준 spec. |
| **runc** | OCI runtime spec 의 reference 구현. container process 를 실제로 fork/exec 한다. |
| **containerd** | runc 를 호출하는 container manager. image pull / snapshot / lifecycle 관리. |
| **build context** | `docker build` 가 daemon 으로 전송하는 디렉터리 tarball. `.dockerignore` 가 결정. |
| **BuildKit** | DAG 기반의 새로운 builder. cache mount / secret mount / multi-arch / parallel build 지원. |

---

## 4. 컨테이너의 정의 — process + namespace + cgroup

### 4.1 모델

container 는 host kernel 을 공유하는 process 이다. 격리는 다음 3 가지 커널 메커니즘으로 구성된다.

| 메커니즘 | 격리 대상 | 관련 파일 / 명령 |
| --- | --- | --- |
| **namespace** | view of system resources | `/proc/<pid>/ns/`, `unshare`, `nsenter`, `lsns` |
| **cgroup (v2)** | resource quota (cpu / mem / io / pids) | `/sys/fs/cgroup/`, `systemd-cgls` |
| **capabilities** | 권한 분해 (CAP_NET_BIND_SERVICE 등) | `/proc/<pid>/status` 의 Cap*, `getcap`, `setcap` |

### 4.2 namespace 7 종

| namespace | 격리 대상 |
| --- | --- |
| mnt | mount points |
| pid | process ID space (컨테이너 안 PID 1) |
| net | 인터페이스 / 라우팅 / iptables |
| uts | hostname / domainname |
| ipc | SysV IPC / POSIX message queue |
| user | UID/GID mapping (rootless container 의 핵심) |
| cgroup | cgroup root view |

→ container = 위 7 가지를 동시에 분리한 process. 일부만 분리도 가능 (`docker run --network host`).

### 4.3 cgroup v2 의 4 컨트롤러

| 컨트롤러 | 제한 항목 | docker 옵션 |
| --- | --- | --- |
| cpu | shares / weight / max | `--cpus`, `--cpu-shares` |
| memory | max / swap / low | `--memory`, `--memory-swap` |
| io | weight / max bps | `--device-read-bps` 등 |
| pids | max pid 수 | `--pids-limit` |

→ cgroup limit 이 없으면 container 가 host 전체 자원을 점유 가능. production 운영의 필수 조건.

### 4.4 검증

```bash
# container 의 PID 가 host 에서 별도 PID 로 보임을 확인
docker run -d --name web nginx
docker inspect --format '{{.State.Pid}}' web
ps -p <pid> -o pid,ppid,cmd
ls -la /proc/<pid>/ns/                # mnt/pid/net/uts/ipc/user/cgroup

# container 의 cgroup limit 확인
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/cpu.max
```

→ container 가 host 의 process 임을 확인하면 "왜 컨테이너가 가볍나" / "왜 kernel 패치는 모든 컨테이너에 영향을 주나" 가 자명.

상세: [[../linux/linux-mental-models-for-devops]].

---

## 5. Image 의 정의 — layer + content-addressable

### 5.1 모델

image 는 다음 3 가지 객체의 그래프이다.

| 객체 | 내용 | content addressing |
| --- | --- | --- |
| **manifest** | layer 와 config 의 참조 list | SHA256 |
| **config** | 실행 메타데이터 (env / cmd / entrypoint / labels) | SHA256 |
| **layer** | filesystem changeset (tar) | SHA256 |

```
manifest (sha256:abc...)
 ├─ config (sha256:def...)
 │   └─ history, env, cmd, ...
 ├─ layer 1 (sha256:111...)  ← base os
 ├─ layer 2 (sha256:222...)  ← apt install
 ├─ layer 3 (sha256:333...)  ← COPY app.jar
 └─ layer 4 (sha256:444...)  ← CMD
```

### 5.2 content-addressable 의 효과

| 효과 | 결과 |
| --- | --- |
| 동일 layer 1 회 저장 | 100 개 image 가 base 공유 시 디스크 1 회 |
| pull 시 부재 layer 만 다운로드 | 1GB image 의 갱신이 50MB layer 1 개일 수 있음 |
| layer 변조 시 hash 불일치 | tampering 검출 |
| build cache key 로 사용 | identical input → identical layer hash → cache hit |

### 5.3 union filesystem

container 실행 시 layer 가 union mount 로 합쳐진다.

| driver | 메커니즘 | 비고 |
| --- | --- | --- |
| overlay2 | Linux overlayfs (lowerdir + upperdir) | 현행 default |
| btrfs / zfs | filesystem snapshot | snapshot 능력 활용 |
| devicemapper | block-level | deprecated |
| fuse-overlayfs | userspace overlay | rootless |

container 의 writable 변경은 모두 가장 위의 `upperdir` 에 기록된다. container 삭제 시 `upperdir` 도 삭제 — 영속 데이터는 volume 필요.

### 5.4 OCI 표준의 분리

| spec | 정의 대상 |
| --- | --- |
| Image Spec | image layout (manifest / config / layer / index) |
| Runtime Spec | container 실행 contract (config.json + rootfs) |
| Distribution Spec | registry HTTP API |

→ 3 spec 이 분리되어 Docker 외 도구 (Podman / Buildah / containerd / Skopeo) 가 호환된다.

---

## 6. Build 의 정의 — context + layer cache + BuildKit graph

### 6.1 build context

`docker build <ctx>` 의 `<ctx>` 디렉터리 전체가 daemon 으로 전송된다.

| 사실 | 결과 |
| --- | --- |
| `.dockerignore` 미사용 시 `node_modules` / `.git` / `dist` 가 전부 전송 | build 시간 증가, context 수 GB |
| context 안 파일 1 byte 변경 시 hash 변경 → 후속 cache miss | 의도치 않은 rebuild |
| context 밖 파일 참조 불가 | monorepo 의 경우 root 에서 build 하거나 BuildKit `--build-context` 사용 |

### 6.2 layer cache 키

각 Dockerfile instruction 의 cache 키는 다음 4 가지 입력의 hash 이다.

| 입력 | 비고 |
| --- | --- |
| 이전 layer hash | 위 layer 가 바뀌면 아래는 모두 miss |
| instruction 문자열 | 공백 / 주석 변경도 hash 영향 |
| COPY/ADD 의 file content hash | content-addressable |
| build arg / ENV (참조된 경우만) | `ARG VERSION` 사용 시만 영향 |

→ 효율적 layer 순서의 원칙:

| 원칙 | 적용 |
| --- | --- |
| 자주 바뀌지 않는 것을 위로 | base image / OS 패키지 → dependency manifest → 소스 코드 → CMD |
| dependency manifest 와 source 분리 | `COPY package.json` → `RUN npm ci` → `COPY src` |
| build arg 는 사용 직전에 선언 | 상단의 ARG 변경이 아래 cache 무효화 |

### 6.3 BuildKit 의 변화

| 항목 | 기존 builder | BuildKit |
| --- | --- | --- |
| 빌드 모델 | 순차 layer | DAG (병렬 가능) |
| context 전송 | 전체 | 필요한 파일만 (streaming) |
| cache | local layer | local + registry + cache mount |
| secret | build arg (history 노출) | `--mount=type=secret` |
| multi-arch | buildx 별도 | 표준 |
| stage 선택 | 전체 빌드 | `--target` 으로 stage 까지만 |

### 6.4 multi-stage build

```dockerfile
# stage 1 — builder (최종 image 에 포함 안 됨)
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

# stage 2 — runtime
FROM gcr.io/distroless/static-debian12
COPY --from=builder /out/app /app
ENTRYPOINT ["/app"]
```

| 효과 | 결과 |
| --- | --- |
| 최종 image 에 toolchain 미포함 | 크기 / attack surface 감소 |
| stage 별 cache | builder 변경이 runtime stage cache 와 독립 |
| distroless / scratch 와 결합 | shell / package manager 없음 |

### 6.5 cache mount / secret mount

```dockerfile
# package manager cache 를 build 간 재사용
RUN --mount=type=cache,target=/root/.cache/go-build \
    go build ./...

# secret 을 image history 에 남기지 않고 build 중에만 노출
RUN --mount=type=secret,id=npm,target=/root/.npmrc \
    npm ci
```

→ `--mount=type=secret` 이전엔 `ARG TOKEN` → image history 영구 노출이 흔한 사고였음.

---

## 7. Runtime 인터페이스 계층

### 7.1 호출 그래프

```
docker CLI
   │
docker daemon (dockerd)
   │  (Docker API)
containerd                     ← CRI (k8s 도 동일)
   │  (containerd shim)
runc                            ← OCI runtime spec 실행
   │  (clone + exec)
Linux kernel (namespace / cgroup / capability)
```

### 7.2 layer 별 책임

| layer | 책임 |
| --- | --- |
| docker CLI | 사용자 명령 → API 호출 |
| dockerd | image build / network / volume orchestration |
| containerd | image pull / snapshot / lifecycle |
| runc | OCI runtime spec 으로 process fork/exec |
| kernel | namespace / cgroup / capability 강제 |

### 7.3 호환성 결과

| 사실 | 결과 |
| --- | --- |
| containerd 만 있어도 컨테이너 실행 가능 | Kubernetes 1.24+ 가 dockershim 제거 후에도 동작 |
| Podman 은 daemon 없이 runc 호출 | rootless / systemd integration 우수 |
| nerdctl 은 containerd 직결 CLI | Docker CLI 호환 명령 |

상세: [[runtime-comparison]].

---

## 8. 운영 결정의 근거

### 8.1 image 크기 최적화

| 기법 | 효과 | trade-off |
| --- | --- | --- |
| multi-stage + distroless / scratch | runtime image 50MB → 10MB 이하 | shell / debug tool 없음 |
| layer 통합 (`RUN ... && ...`) | layer 수 감소 | cache 단위 큼 |
| `.dockerignore` 정리 | context / image 크기 감소 | 누락 시 빌드 실패 |
| Alpine base | 5MB base | musl libc 호환성 (glibc 가정 라이브러리 실패) |
| `--squash` | 모든 layer 1 개로 압축 | 공유 base 의 이점 상실 |

### 8.2 보안

| 항목 | 권장 |
| --- | --- |
| user | `USER 1000:1000` — non-root |
| capability | `--cap-drop=ALL --cap-add=NET_BIND_SERVICE` 같이 명시 추가 |
| read-only rootfs | `--read-only` + 필요한 path 는 tmpfs / volume |
| seccomp / AppArmor | default profile 유지 |
| supply chain | image scan (trivy / grype) + SBOM (syft) + signing (cosign) |

상세: [[security]].

### 8.3 디버깅

| 증상 | 추적 경로 |
| --- | --- |
| 컨테이너 즉시 종료 | `docker logs <id>` → exit code → entrypoint 확인 |
| 메모리 OOM | `dmesg | grep -i oom` + cgroup memory.events |
| 네트워크 실패 | `docker network inspect` + `nsenter -t <pid> -n ip a` + iptables -L |
| layer mount 실패 | `docker info` 의 storage driver + `journalctl -u docker` |
| build cache miss | `docker build --progress=plain --no-cache` 비교 |

---

## 9. 흔한 실패 모드

| 실패 | 원인 | 모델로부터의 설명 |
| --- | --- | --- |
| `COPY . .` 직후 layer 항상 miss | context 안 파일 변경이 hash 변경 | §6.2 cache 키 = file content hash |
| build 가 갑자기 느려짐 | `.dockerignore` 미적용 + context 수 GB | §6.1 context 전체 전송 |
| `ARG TOKEN` 후 image history 에 token | build arg 가 history 에 남음 | §6.5 cache mount / secret mount 미사용 |
| Alpine 에서 glibc 라이브러리 실패 | musl libc 차이 | §8.1 Alpine base |
| `docker run` 의 container 가 host 100% CPU | cgroup limit 미설정 | §4.3 cgroup |
| host 의 process 가 container 안에서 보임 | `--pid=host` | §4.2 pid namespace 미분리 |
| `latest` 태그 갱신 후 deployment 가 옛 image | image pull policy + tag mutability | §5.1 manifest 기반 + tag 는 단순 alias |
| compose down 후 데이터 손실 | volume 미정의 (writable layer 가 사라짐) | §5.3 upperdir 삭제 |

---

## 10. 참고

- [[docker|↑ docker]]
- [[concepts]]
- [[dockerfile-best-practices]]
- [[buildkit-deep]]
- [[production-patterns]]
- [[runtime-comparison]]
- [[security]]
- [[pitfalls]]
- [[../linux/linux-mental-models-for-devops]]
- [[../../computer-science/network/topics/container-networking|↗ CS — container networking (veth / bridge / VXLAN)]] — Docker bridge 의 실제 구성
- [[../../computer-science/network/tools/iptables-netfilter|↗ CS — iptables / netfilter]] — Docker 의 NAT / DOCKER chain
- OCI Image Spec v1.1 — https://github.com/opencontainers/image-spec
- OCI Runtime Spec v1.1 — https://github.com/opencontainers/runtime-spec
