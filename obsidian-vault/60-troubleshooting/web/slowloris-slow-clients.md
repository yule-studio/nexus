---
title: "troubleshooting · Slowloris / 느린 클라이언트 DoS"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, dos, slowloris, timeout, connection, nginx]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[retry-storm-metastable]]"]
---

# Slowloris / 느린 클라이언트 DoS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — 느린 연결 고갈/타임아웃/비동기 프록시 |

**[[web|↑ web]]**

---

## 1. 증상

- 트래픽 대역폭도, 초당 요청 수(RPS)도 평범한데 **서버가 신규 연결을 못 받는다**. CPU·메모리·네트워크는 한가한데 사이트는 죽어 있다.
- 커넥션/워커 슬롯이 전부 **"열려 있지만 거의 안 보내는" 연결**로 점유됨. 즉 연결은 살아 있지만 데이터가 흐르지 않는다.
- nginx 로그/응답에 `503 Service Temporarily Unavailable`, 에러 로그에 `worker_connections are not enough` / `too many open files`.
- 백엔드(WAS) 스레드풀 포화 → [[../backend/thread-pool-exhausted|thread-pool-exhausted]] 와 동일한 증상이 앞단에서 유발됨.
- 동시 연결 수는 비정상적으로 높은데 **처리량(완료 요청 수)은 바닥**. established 는 많은데 throughput 은 거의 0.
- **적은 공격 자원으로 가능** — 노트북 한 대, 수백 개 연결만으로도 중형 서버를 마비시킬 수 있다. 볼류메트릭 DDoS 와 달리 대역폭 폭증 신호가 없어 탐지가 늦다.

진단 한 줄:

```bash
# established 인데 거의 데이터가 안 흐르는 연결이 비정상적으로 많은지
ss -tan state established | wc -l
ss -tan state established '( dport = :443 or dport = :80 )' | head
# nginx: reading(헤더 읽는 중) 상태가 비정상적으로 누적되는지
curl -s http://127.0.0.1/nginx_status   # stub_status: Reading / Writing / Waiting
```

`Reading` 카운트가 비정상적으로 높게 고정돼 있으면 헤더를 천천히 보내는 연결이 슬롯을 점유 중이라는 강한 신호다.

---

## 2. 원인 (root cause + 메커니즘)

**Slowloris** (2009, Robert "RSnake" Hansen 공개) 류의 **저대역폭(low-and-slow) DoS**. 메커니즘은 단순하다.

1. 공격자가 서버에 수백~수천 개의 TCP 연결을 연다.
2. 각 연결로 HTTP 요청을 **아주 천천히** 보낸다 — 헤더를 몇 초에 한 바이트씩, 그리고 **요청을 끝내지 않는다**(마지막 빈 줄 `\r\n\r\n` 을 안 보냄).
3. 서버는 "요청 헤더가 아직 안 끝났다"고 판단하고 연결을 **계속 붙들어 둔다**. 타임아웃이 길거나 없으면 무한정 점유.
4. 주기적으로 한 바이트씩 흘려보내 연결이 idle 타임아웃에 걸리지 않게 keepalive 한다.

핵심은 **연결당 워커/스레드를 점유하는 모델**이 특히 취약하다는 점이다. Apache `prefork` MPM 처럼 "연결 1개 = 프로세스/스레드 1개"인 구조에서는 **동시 연결 한도가 곧 가용량**인데, 그 한도를 느린 연결이 전부 차지해 버린다. 정상 사용자를 받을 빈 슬롯이 남지 않는다.

변형:

- **slow POST (R.U.D.Y, "R-U-Dead-Yet")** — 헤더는 정상적으로 끝내되 `Content-Length` 를 크게 선언하고 **body 를 찔끔씩** 전송. 서버는 body 수신을 기다리며 연결을 붙든다.
- **slow read** — 요청은 정상적으로 보내고, **응답을 아주 느리게 읽는다**. TCP 수신 윈도를 작게 광고해 서버가 응답 버퍼를 비우지 못하게 만들어 송신 측 자원을 점유.

세 변형 모두 "정상 프로토콜을 느리게 진행"하므로, 시그니처 기반 IDS 로는 잘 안 걸린다. 패킷 하나하나는 합법이다.

---

## 3. 해결

### 3.1 타임아웃을 짧고 명확히

헤더 수신 / body 수신 / 송신 / 유휴 각각에 상한을 둔다. nginx 예시:

```nginx
client_header_timeout 5s;    # 헤더 전부 받는 데 허용하는 시간
client_body_timeout   10s;   # body 한 청크와 다음 청크 사이 최대 간격
send_timeout          10s;   # 클라가 응답을 안 읽고 멈추면 끊는다 (slow read 방어)
keepalive_timeout     15s;   # idle keepalive 연결 정리
```

Apache 는 `mod_reqtimeout` 으로 동일 방어:

```apache
RequestReadTimeout header=5-20,MinRate=500 body=10,MinRate=500
# 헤더는 최소 5초 허용, 단 초당 500바이트 미만이면 끊음 (min data rate)
```

### 3.2 비동기/이벤트 기반 리버스 프록시로 버퍼링 (근본 격리)

nginx / Envoy 같은 **이벤트 루프 기반 프록시**를 앞에 둔다. 이들은 연결 1개에 스레드를 묶지 않고, 단일 워커가 수만 연결을 epoll/kqueue 로 다중화한다. 따라서 느린 연결이 수천 개여도 워커는 고갈되지 않는다. 그리고 nginx 는 **요청이 완성될 때까지 버퍼링**한 뒤 백엔드(WAS)에는 **완성된 요청만** 한 번에 전달한다 → thread-per-connection 백엔드를 슬로우 연결에서 격리한다. (이것이 nginx 가 Apache prefork 보다 Slowloris 에 강한 이유다.)

```nginx
proxy_request_buffering on;   # 기본값. body 를 다 받은 뒤 백엔드로 (slow POST 흡수)
proxy_buffering on;           # 응답도 프록시가 흡수 (slow read 가 백엔드에 안 닿음)
```

### 3.3 연결 수 / 요청률 제한

IP 당 동시연결·요청률을 캡해서 한 출처가 슬롯을 독식하지 못하게 한다.

```nginx
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_req_zone  $binary_remote_addr zone=reqs:10m rate=10r/s;

server {
    limit_conn perip 20;        # IP당 동시연결 20개
    limit_req  zone=reqs burst=20 nodelay;
}
```

L4/L7 방화벽이나 `iptables` 의 `connlimit` 으로 보강할 수도 있다.

### 3.4 클라우드 / CDN / WAF 앞단

Cloudflare·CloudFront·ALB 등을 origin 앞에 둔다. 이들은 대규모 연결을 흡수하고 **완성된 요청만 origin 으로 전달**하므로, 느린 연결은 애초에 origin 에 도달하지 않는다. 분산 공격에서는 사실상 필수.

### 3.5 최소 전송 속도 강제 (min data rate)

일부 서버/프록시는 "초당 N바이트 미만이면 연결을 끊는" 규칙을 지원한다(위 Apache `MinRate`, ASP.NET Core `MinRequestBodyDataRate`). 절대 시간 타임아웃보다 정밀하게 "느림"을 판정할 수 있다.

---

## 4. 핵심 트레이드오프 — 타임아웃 강화 vs 비동기 프록시 버퍼링

둘 다 필요하지만 성격이 다르다. 4구조로 본다.

### ① 왜 필요한가
열려만 있고 진행 안 하는 연결을 **언젠가는 강제로 끊어야** 슬롯이 회수된다. 타임아웃은 그 "끊는 기준"이고, 비동기 프록시는 애초에 "끊을 일이 적게" 만드는 아키텍처적 격리다.

### ② 안 하면 무슨 일이 나나
- 타임아웃을 안 잡으면(또는 기본값이 느슨하면) 느린 연결이 무한 점유 → 슬롯 고갈 → 신규 연결 거부.
- 비동기 프록시 격리가 없으면, 타임아웃을 잡아도 thread-per-connection 백엔드는 매 슬로우 연결마다 스레드를 소비하며 흔들린다. 타임아웃은 "출혈을 멈출 뿐" 근본 구조는 그대로다.

### ③ 대안
- 타임아웃만으로: 즉효이고 설정 한 줄이면 끝. 그러나 임계값을 잘못 잡으면 부작용이 크다.
- 비동기 프록시만으로: 구조적으로 강하지만, 프록시 계층 자체를 운영·튜닝해야 하고 버퍼링이 켜진 상태여야(`proxy_request_buffering on`) 의미가 있다.

### ④ 트레이드오프
- **타임아웃 강화는 즉효지만, 정당한 느린 사용자를 끊을 위험**이 있다. 2G/약전계 모바일, 대용량 업로드, 위성 회선 사용자는 실제로 헤더·body 가 천천히 도착한다. `client_header_timeout 5s` 가 이들을 끊을 수 있어 **임계 튜닝**이 필요하다. 너무 짧으면 정상 사용자 차단, 너무 길면 방어 무력화.
- **비동기 프록시 버퍼링은 아키텍처로 근본 격리**하지만, 프록시 계층을 도입·운영해야 하고 버퍼링이 메모리/디스크를 쓰며(대용량 업로드 시 임시 파일) 첫 바이트 지연(TTFB)에 영향을 줄 수 있다.

**정석은 둘 병행**이다. 비동기 프록시로 백엔드를 격리하고, 프록시 자체에는 현실적인 타임아웃·연결 제한을 건다. 모바일 사용자를 위해 타임아웃은 측정 기반으로 보수적으로 잡되(예: 헤더 5초가 아니라 10~15초), 대신 `limit_conn` 으로 한 IP 의 점유를 캡한다.

구체 예시: 헤더 타임아웃을 1초로 공격적으로 줄였더니 Slowloris 는 막혔지만, 약전계 모바일 결제 페이지에서 503 이 튀었다. → 헤더 타임아웃은 10초로 완화하고, 대신 `client_header_timeout` 보다 `limit_conn perip 20` + 앞단 WAF 에 방어 비중을 옮겨 정상 사용자 영향 없이 막았다.

---

## 5. 재발 방지

- **연결 상태 분포 모니터링** — 단순 동시연결 수가 아니라 `established` 대비 `reading`(헤더 수신 중) 비율을 본다. reading 이 비정상적으로 누적되면 알람. nginx `stub_status` 의 Reading/Writing/Waiting, Envoy `downstream_cx_active` 등.
- **타임아웃 기본값 점검** — 많은 서버가 기본값이 느슨하거나 무한이다(구형 Apache, 일부 임베디드 서버). 신규 서비스 배포 체크리스트에 "header/body/send/keepalive 타임아웃 명시" 항목을 넣는다.
- **부하/슬로우 클라 테스트** — 일반 부하 테스트는 빠른 클라로 때려서 Slowloris 를 못 잡는다. `slowhttptest` 같은 전용 도구로 검증한다.

```bash
# slowloris(헤더 천천히) 시뮬레이션 — 500 연결로 헤더를 찔끔씩
slowhttptest -c 500 -H -i 10 -r 200 -t GET -u https://staging.example.com -x 24 -p 3
# slow POST(R.U.D.Y) 는 -B, slow read 는 -X 모드
```

- **앞단 WAF 상시** — CDN/WAF 를 평시에도 켜 둔다. 공격 시점에만 켜는 구성은 탐지가 늦어 이미 늦는다.
- **회귀 라인 고정** — 타임아웃·`limit_conn` 설정을 IaC(터폼/Ansible)에 코드화하고, 설정 드리프트를 CI 에서 감지한다. 누군가 트러블슈팅하다 타임아웃을 늘려놓고 안 되돌리는 일이 흔하다.

---

## 관련 노트

- [[web]] — 웹 트러블슈팅 허브
- [[retry-storm-metastable]] — 자원 고갈 / 메타스테이블 장애 계열
- [[graceful-shutdown-draining]] — 연결 드레이닝과 종료
- [[../backend/thread-pool-exhausted]] — 백엔드 스레드풀 포화(슬로우 연결의 하류 증상)
- [[../network/tcp-keepalive]] — TCP keepalive 와 idle 연결 정리
- [[../troubleshooting|↑ 60-troubleshooting]]
