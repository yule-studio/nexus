---
title: "SSRF — Server-Side Request Forgery (OWASP A10)"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, ssrf, cloud, metadata, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[broken-access-control]]", "[[security-misconfiguration]]"]
---

# SSRF — Server-Side Request Forgery (OWASP A10)

| 버전 | 날짜 | 작성 | 변경 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — 원리/Capital One/메타데이터/방어 |

**[[web-security|↑ web-security]]**

> OWASP A10. "서버가 공격자 제어 URL로 요청을 대신 보내게 만들어 내부 자원에 도달".

## 1. 개요

SSRF(Server-Side Request Forgery)는 서버가 **사용자가 제어하는 URL을 받아 그
URL로 요청을 대신 보내는** 기능에서 발생한다. 공격자는 외부에서 직접 도달할 수
없는 자원(내부망 서비스, 클라우드 메타데이터, 로컬 관리 포트)을 **서버의 신원과
네트워크 위치를 빌려** 간접적으로 호출한다.

전형적인 진입점:

- **이미지 프록시 / 썸네일 생성** — 외부 이미지 URL을 받아 서버가 fetch 후 리사이즈
- **웹훅(webhook) 등록** — 사용자가 콜백 URL을 입력하면 서버가 주기적으로 POST
- **PDF / HTML 렌더러** — 사용자 문서 안의 `<img src>` / 외부 리소스를 서버가 로드
- **URL 미리보기(link unfurl)** — 채팅/위키가 붙여넣은 링크의 OG 태그를 서버가 긁음
- **파일 import "from URL"** — CSV/이미지를 URL로 가져오는 기능

이 기능들의 공통점은 **목적지를 사용자가 정하고, 실제 네트워크 요청은 서버가
한다**는 것이다. 여기에서 신뢰 경계가 무너진다.

## 2. 원리

```
공격자 ──(POST /preview {url: "http://169.254.169.254/..."})──▶ 웹 서버
                                                                  │
                       서버가 그 URL을 그대로 fetch ◀────────────┘
                                                                  ▼
                               내부 자원 (외부에서 못 가는 곳)에 도달
```

공격자가 주입할 수 있는 대표 목적지:

- `http://169.254.169.254/latest/meta-data/...` — **클라우드 인스턴스 메타데이터**(AWS/GCP/Azure 공통 link-local 주소). IAM 자격증명, 사용자 데이터, 토큰이 나온다.
- `http://localhost:8080/admin`, `http://127.0.0.1:9200` — **로컬 관리/내부 포트**(Elasticsearch, Actuator, 관리 콘솔). 보통 인증이 약하거나 없다.
- `http://10.0.3.12/`, `http://192.168.x.x` — **내부망 사설 IP**의 다른 서비스
- `file:///etc/passwd` — fetch 라이브러리가 `file://`를 허용하면 로컬 파일 읽기
- `gopher://internal:6379/_SET...` — **gopher**로 raw TCP 페이로드를 보내 Redis/Memcached 같은 비-HTTP 서비스 명령 주입

핵심은 "내부에서 보면 서버 자신이 보낸 정상 요청"으로 보인다는 점이다. 방화벽과
인증은 보통 "외부 → 내부"만 막고, "내부 서버 → 내부 서비스"는 신뢰한다.

## 3. 실제 사례 — Capital One (2019)

- Capital One의 WAF(웹 애플리케이션 방화벽)에 SSRF 취약점이 있었다.
- 공격자는 이를 통해 EC2 인스턴스의 **메타데이터 엔드포인트
  `http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>`** 을 호출.
- 이 엔드포인트는 인스턴스에 붙은 **IAM 역할의 임시 자격증명**(AccessKey/Secret/Token)을 반환한다.
- 탈취한 자격증명으로 **S3 버킷에 접근** → 약 1억 건 이상의 신용카드 신청 기록(SSN, 계좌 등 포함) 유출.

이 사건이 교과서적인 이유: SSRF 자체는 "서버가 요청을 대신 보낸다"는 비교적
얌전해 보이는 결함이지만, **클라우드 환경에서는 메타데이터 서비스가 곧 IAM
자격증명 분배 창구**이므로, SSRF 하나가 곧바로 **권한 상승 + 데이터 유출**로
이어진다. 당시 노출됐던 **IMDSv1**은 토큰 없이 단순 GET만으로 메타데이터를
읽을 수 있었던 점이 치명타였다.

## 4. 공격 변형 (우회 기법)

단순 문자열 블랙리스트가 왜 못 막는지 보여주는 변형들:

```text
# blind SSRF — 응답이 화면에 안 돌아와도, 타이밍/외부 콜백으로 내부 존재 추론
http://internal-host:8080/   (지연 시간 차이로 포트 열림 판별)

# DNS rebinding — 검증 시점엔 정상 IP, 실제 fetch 시점엔 내부 IP로 재해석
http://attacker.com/  → (검증 통과) → TTL 0으로 169.254.169.254 재바인딩

# redirect 우회 — 허용된 외부 URL이 302로 내부 URL을 가리킴
https://allowed.example.com/r → 302 Location: http://169.254.169.254/...

# IP 인코딩 우회 (모두 127.0.0.1 / 169.254.169.254 동일)
http://2130706433/        (10진수)
http://0x7f000001/        (16진수)
http://0177.0.0.1/        (8진수)
http://[::1]/  http://0.0.0.0/  http://[::ffff:a9fe:a9fe]/
```

이 목록이 보여주는 결론: **목적지 문자열을 검사하는 블랙리스트는 인코딩과
재해석으로 거의 항상 뚫린다.** 그래서 방어의 무게중심은 "최종 IP" 검증과
"명시 허용"으로 옮겨야 한다.

## 5. 방어 — 왜 egress allowlist + 메타데이터 차단이 핵심인가

### 5.1 Egress allowlist (나가는 목적지 명시 허용)

- **① 왜 필요**: SSRF의 본질은 "서버가 임의 목적지로 나갈 수 있음"이다. 나갈 수
  있는 호스트/IP/포트를 화이트리스트로 좁히면 공격 표면 자체가 사라진다.
- **② 안 하면**: 블랙리스트로 "내부 IP만 막기"를 하면, 위 4절의 인코딩/rebinding/redirect로
  하나씩 뚫린다. 방어자가 모든 표기법을 따라잡아야 하는 비대칭 싸움이 된다.
- **③ 대안**: 응답 본문 필터링, IP 검증 라이브러리 — 보조 수단일 뿐 근본 통제가 아님.
- **④ 트레이드오프**: allowlist는 정당한 외부 연동(새 결제사 도메인 등)이 늘 때마다
  운영 갱신이 필요하다. 대신 그 마찰이 "함부로 임의 목적지로 안 나간다"는 보장을 준다.

### 5.2 Link-local / 사설 / 메타데이터 IP 차단 (DNS 해석 후 검증)

차단 대상 범위:

```text
169.254.0.0/16   (link-local — 메타데이터 169.254.169.254 포함)
127.0.0.0/8      (loopback)
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
0.0.0.0/8        (this host)
::1  fe80::/10  fc00::/7   (IPv6 loopback/link-local/ULA)
```

- **① 왜 필요**: 외부에서 못 가는 모든 사설/특수 대역으로의 요청을 막는다.
- **② 안 하면**: 메타데이터 한 줄(169.254.169.254)이 곧 IAM 자격증명 유출(=Capital One).
- **③ 대안**: 도메인 이름만 검사 → DNS rebinding으로 무력화.
- **④ 트레이드오프 / 핵심**: 반드시 **DNS 해석을 직접 수행한 뒤, 그 최종 IP로
  검증하고, 같은 IP로 connect** 해야 한다(resolve-then-connect를 하나로 묶기).
  "검증 시점 IP ≠ 연결 시점 IP" 틈이 rebinding 공격의 통로다.

### 5.3 IMDSv2 강제 (AWS — 클라우드 측 근본 완화)

- **① 왜 필요**: 애플리케이션 레벨 방어가 새는 경우에도 메타데이터를 못 읽게 하는 2차 방어선.
- IMDSv2는 **PUT으로 토큰을 먼저 받고, 그 토큰을 헤더에 실어야** 메타데이터를 읽을 수 있다.
- **② 안 하면**: IMDSv1(단순 GET)은 SSRF 한 방으로 자격증명이 나온다.
- **③ 대안**: IMDS 자체를 hop limit=1로 제한, 컨테이너에선 메타데이터 차단.
- **④ 트레이드오프**: 일부 구형 SDK/스크립트가 IMDSv1을 가정 → 마이그레이션 필요.

```bash
# 인스턴스에 IMDSv2 강제 (토큰 없는 IMDSv1 요청은 거부됨)
aws ec2 modify-instance-metadata-options \
  --instance-id i-0abc123 \
  --http-tokens required \
  --http-put-response-hop-limit 1
```

### 5.4 운영 원칙

- **raw URL fetch 금지** — 사용자 URL을 라이브러리에 그대로 넘기지 말고 **전용
  프록시/브로커**를 거치게 한다(프록시에서 IP 검증 + allowlist + 스킴 제한).
- **허용 스킴 제한** — `http`/`https`만. `file://`, `gopher://`, `ftp://`, `dict://` 차단.
- **redirect 비허용 또는 재검증** — 302 따라가기를 끄거나, 따라갈 때마다 5.2를 다시 적용.
- **응답을 사용자에게 그대로 반사하지 않기** — blind SSRF가 정보 누출 SSRF로
  격상되는 걸 막는다(에러 본문/리다이렉트 헤더도 마스킹).
- **네트워크 분리** — 외부 콘텐츠를 fetch하는 워커는 **내부 관리망에 라우팅 자체가
  안 되는** 격리 서브넷에 둔다. 코드가 새도 패킷이 못 간다.

## 6. 설정 / 코드 예시

공격 재현 (이미지 프록시가 임의 URL을 그대로 fetch할 때):

```bash
# 메타데이터에서 IAM 역할 임시 자격증명 시도 (IMDSv1 가정)
curl -s 'https://victim.example.com/proxy?url=\
http://169.254.169.254/latest/meta-data/iam/security-credentials/'

# 내부 관리 포트 스캔/접근
curl -s 'https://victim.example.com/proxy?url=http://localhost:8080/admin'
```

방어 — fetch 전에 최종 IP를 검증하는 Python 예시:

```python
import ipaddress, socket

_BLOCKED = [
    ipaddress.ip_network(n) for n in (
        "169.254.0.0/16", "127.0.0.0/8", "10.0.0.0/8",
        "172.16.0.0/12", "192.168.0.0/16", "0.0.0.0/8",
        "::1/128", "fe80::/10", "fc00::/7",
    )
]
_ALLOWED_HOSTS = {"cdn.partner.com", "images.trusted.io"}  # egress allowlist

def safe_target(url):
    from urllib.parse import urlparse
    u = urlparse(url)
    if u.scheme not in ("http", "https"):
        raise ValueError("scheme not allowed")
    if u.hostname not in _ALLOWED_HOSTS:
        raise ValueError("host not in allowlist")
    # DNS 해석 후 최종 IP로 검증 — 이 IP로 직접 connect 해야 rebinding 방어됨
    ip = socket.getaddrinfo(u.hostname, None)[0][4][0]
    addr = ipaddress.ip_address(ip)
    if any(addr in net for net in _BLOCKED):
        raise ValueError("blocked internal address")
    return ip  # 검증된 이 IP로 직접 연결 (resolve-then-connect)
```

nginx를 outbound 프록시로 둘 때 — 메타데이터/내부 대역 차단:

```nginx
# 외부 fetch 워커는 이 프록시로만 나가게 강제 (직접 egress 차단)
location / {
    # 메타데이터 / 내부 대역 deny
    if ($host = "169.254.169.254") { return 403; }
    proxy_pass http://$host$request_uri;
}
# 더 견고하게는 nginx보다 IP 검증을 수행하는 전용 SSRF 브로커 권장.
```

AWS SDK 측 메타데이터 hop 제한(컨테이너 SSRF 완화):

```bash
# ECS/EC2: 메타데이터 응답 hop limit을 1로 → 컨테이너 네트워크에서 도달 어렵게
aws ec2 modify-instance-metadata-options --instance-id i-0abc123 \
  --http-put-response-hop-limit 1 --http-tokens required
```

## 7. 관련 노트

- [[broken-access-control]] — SSRF로 도달한 내부 엔드포인트가 인증·인가를 안 하면 피해 폭증
- [[security-misconfiguration]] — IMDSv1 방치, 관리 포트 노출 등은 설정 결함과 맞물림
- [[../cloud-security/cloud-security|cloud-security]] — IAM 임시 자격증명/메타데이터 모델
- [[../security|↑ security]]
