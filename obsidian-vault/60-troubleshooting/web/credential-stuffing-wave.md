---
title: "troubleshooting · 크리덴셜 스터핑 물결 (로그인 폭주/계정 탈취)"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, credential-stuffing, account-takeover, rate-limit, bot, auth]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[../../20-areas/security/web-security/authentication-session|authentication-session]]"]
---

# 크리덴셜 스터핑 물결 (로그인 폭주/계정 탈취)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — 탐지/레이트리밋/유출비번/MFA |

**[[web|↑ web]]**

## 증상

- 로그인 엔드포인트 트래픽이 평소의 수 배~수십 배로 **급증**. 그런데 가입/구매 같은 다른 엔드포인트는 잠잠하다 — 로그인만 콕 집어 때린다.
- **실패율(HTTP 401)이 폭증**. 정상 시 한 자릿수 %이던 로그인 실패율이 90%대까지 치솟는다. "대부분 틀리는데 트래픽은 폭증" = 누군가 남의 비번을 무차별 대입 중.
- **다수 IP에서 분산된 시도, IP당 소량씩**. 한 IP가 1000번 때리는 게 아니라 1만 개 IP가 각자 5~10번씩 — 단순 IP 레이트리밋/계정 락의 사각지대를 노린다.
- 평소와 **다른 시간대/지역** 트래픽(새벽, 평소 사용자 없는 ASN/국가)에서 유입.
- 갑작스런 **"계정 탈취 / 부정 결제" CS 문의 급증** — 적중한 계정에서 실제 피해가 발생.
- 인증 DB / 세션 스토어 부하 상승(bcrypt 검증 비용 × 폭주 = CPU 포화), 로그인 지연.
- 봇이라 **User-Agent 패턴 / 요청 간격이 기계적** — 동일 UA 대량 반복, 사람이라면 불가능한 균일한 간격, 마우스/JS 미실행(헤드리스).

```
정상:     ingress↑ 완만 · 401 비율 5% · IP 분포 자연스러움
스터핑:   ingress↑↑ 급증 · 401 비율 90%+ · IP 1만개 각 5~10회 · UA 단조
                                  └─ "많이 틀리는데 폭주" 가 핵심 시그니처
```

## 가설

- "마케팅/이벤트로 진짜 사용자가 몰렸나?" → 그러면 **성공률이 정상**이어야 한다. 401 비율 90%는 정상 유입이 아니다.
- "한 계정을 노린 brute-force인가?" → 로그를 보면 **계정마다 1~2회씩**, 서로 다른 수만 계정에 분산. brute-force(한 계정 × 여러 비번)가 아니라 **여러 계정 × 1~2 비번**.
- "단순 IP 차단으로 막히나?" → 분산 봇넷이라 IP 하나 막아도 만 개가 남는다. IP축 단독 방어로는 안 풀린다.

## 원인 (root cause + 메커니즘)

**크리덴셜 스터핑(credential stuffing)** = 다른 사이트에서 유출된 **(이메일, 비밀번호) 목록**을 자동화 도구(Sentry MBA, OpenBullet 같은 "combo list runner")로 우리 로그인에 **대량 대입**하는 공격이다.

핵심은 **비밀번호 재사용**이다. 사용자가 여러 사이트에 같은 비번을 쓰기 때문에, 전혀 무관한 남의 사이트 유출본(combo list)이 **우리 사이트에서도 일정 비율로 적중**한다. 업계 경험칙상 적중률(valid 비율)은 0.1~2% 수준 — 낮아 보여도 100만 건 대입이면 수천~수만 계정이 뚫린다.

brute-force 와 명확히 구분된다:

```
brute-force      : 계정 1개  × 비번 N개      → 계정 락 / IP 락으로 쉽게 차단
credential stuffing : 계정 N개 × 비번 1~2개   → 계정당 시도가 적어 락 회피
                       + 분산 IP(봇넷)        → IP 락도 회피
```

스터핑이 단순 락을 회피하는 이유가 바로 이 분산성이다. 계정당 1~2회면 "5회 실패 시 계정 락" 정책에 안 걸리고, IP당 소량이면 "IP당 분당 N회" 임계에도 안 걸린다. **유효 자격증명을 들고 오기 때문에**(추측이 아님) 적중분은 정상 로그인과 구분조차 어렵다.

산업 전반의 **상시 위협**이다 — 대형 스트리밍/커머스/게임 사이트가 신규 대형 유출이 다크웹에 풀릴 때마다 반복적으로 겪는다. 즉 "우리 DB가 안 털렸어도" 남의 유출 때문에 우리가 공격당한다.

근원을 거슬러 올라가면 **유출 자체**가 약한 비밀번호 저장(평문/MD5/단순 SHA)에서 비롯된 경우가 많다 → [[../../20-areas/security/web-security/cryptographic-failures|cryptographic-failures]]. 우리가 통제 못 하는 남의 사이트 실수가 우리 사용자를 위협한다.

## 해결 (즉시 → 근본)

### 1. 즉시 — 다축 레이트 리밋 + 차단

IP **단독**이 아니라 **계정 + IP + 디바이스 핑거프린트** 다축으로 레이트 리밋을 건다. 비정상 ASN/알려진 봇넷/데이터센터 IP를 차단하고, 실패 임계 초과 시 **단계적 마찰**(지연 → CAPTCHA → 차단)을 준다.

```python
# Redis 토큰 버킷 — 계정/IP 다축 (둘 중 하나라도 소진되면 마찰)
import time, redis
r = redis.Redis()

def allow(axis_key: str, capacity: int, refill_per_sec: float) -> bool:
    now = time.time()
    p = r.pipeline()
    p.hmget(axis_key, "tokens", "ts")
    tokens, ts = p.execute()[0]
    tokens = float(tokens) if tokens else capacity
    ts = float(ts) if ts else now
    # 경과 시간만큼 토큰 보충 (상한 capacity)
    tokens = min(capacity, tokens + (now - ts) * refill_per_sec)
    if tokens < 1:
        r.hset(axis_key, mapping={"tokens": tokens, "ts": now})
        return False                      # 소진 → 마찰/차단
    r.hset(axis_key, mapping={"tokens": tokens - 1, "ts": now})
    r.expire(axis_key, 3600)
    return True

# 로그인 시: 계정축 AND IP축 모두 통과해야 시도 허용
ok = allow(f"rl:acct:{email}", capacity=5, refill_per_sec=0.05) \
   and allow(f"rl:ip:{client_ip}", capacity=20, refill_per_sec=0.2)
if not ok:
    require_captcha()                     # 단계적 마찰
```

### 2. 유출 비밀번호 차단 (breached password check)

가입 / 로그인 / 비번 변경 시 입력 비번이 알려진 유출 목록에 있으면 거부한다. HaveIBeenPwned **Pwned Passwords** API의 **k-익명성(k-anonymity)** 모델을 쓰면 비번 원문도, 전체 해시도 외부로 안 나간다 — SHA-1 해시의 **앞 5자리 prefix만 전송**하고, 서버는 그 prefix로 시작하는 모든 해시 suffix 목록을 돌려준다. 매칭은 우리 쪽에서 한다.

```python
import hashlib, requests

def is_pwned(password: str) -> bool:
    sha1 = hashlib.sha1(password.encode()).hexdigest().upper()
    prefix, suffix = sha1[:5], sha1[5:]
    # prefix 5자리만 전송 → API는 우리가 무슨 비번을 묻는지 모름
    resp = requests.get(f"https://api.pwnedpasswords.com/range/{prefix}",
                        headers={"Add-Padding": "true"}, timeout=2)
    for line in resp.text.splitlines():
        h, count = line.split(":")
        if h == suffix:
            return int(count) > 0          # 유출 목록에 존재 → 거부
    return False

# 로그인 성공했더라도, 유출된 비번이면 강제 변경 유도(soft) 가능
if is_pwned(password):
    reject("이 비밀번호는 외부 유출 목록에 포함되어 안전하지 않습니다. 변경이 필요합니다.")
```

### 3. MFA (다중 인증)

적중(비번이 맞아도) 해도 **2차 인증**으로 탈취를 차단한다. 전 사용자 강제는 UX 마찰이 크므로 **위험 기반(risk-based)** — 낯선 디바이스/지역/ASN에서의 로그인에만 OTP/passkey를 요구한다.

```
정상 디바이스 + 정상 지역  → 비번만 (마찰 0)
낯선 디바이스/지역/봇 의심  → MFA 강제 (적중해도 차단)
```

### 4. 봇 관리 / 디바이스 핑거프린트 / 행위 분석

- 디바이스 핑거프린트(브라우저/캔버스/TLS JA3 지문)로 동일 봇 클러스터 식별.
- **헤드리스 탐지** — `navigator.webdriver`, JS 챌린지 미실행, 마우스 이벤트 부재.
- 행위 분석 — 사람이라면 불가능한 균일 간격/속도 패턴 탐지.

### 5. 자격 비노출 (정보 누출 차단)

- 로그인 실패 메시지를 **일반화** — "아이디가 없습니다" vs "비밀번호가 틀립니다"를 구분하면 **계정 존재 여부(user enumeration)**가 누설된다. 항상 동일하게 "아이디 또는 비밀번호가 올바르지 않습니다".
- **타이밍 일정화** — 존재하는 계정은 bcrypt 검증을 거쳐 느리고, 없는 계정은 빨리 반환되면 응답 시간 차로 존재 여부가 새어나간다. 없는 계정에도 더미 해시 검증을 돌려 시간을 맞춘다.

## 핵심 방어 비교 — 레이트 리밋 vs MFA (4구조)

이 둘은 같은 층의 경쟁이 아니라 **물량을 막는 층**과 **적중을 무력화하는 층**으로 역할이 다르다.

1. **왜 필요한가** — 레이트 리밋/봇 관리는 **자동화 비용을 올려** 대량 대입의 물량 자체를 줄인다(공격자가 더 많은 IP/우회 인프라를 사야 채산이 안 맞게 만든다). MFA는 **적중이 일어난 뒤**를 책임진다 — 비번이 맞아도 2차 요소가 없으면 못 들어간다. 하나는 "얼마나 시도하게 둘지", 다른 하나는 "맞춰도 들어오게 둘지"를 다룬다.
2. **안 하면 무슨 문제** — 레이트 리밋만 있고 MFA가 없으면, 분산 봇넷이 임계 아래로 천천히 흘려보내(low-and-slow) 결국 적중분이 그대로 계정 탈취로 직결된다. 레이트 리밋은 물량을 줄일 뿐 **0으로 못 만든다**. MFA만 있고 레이트 리밋이 없으면, 인증 DB/세션 스토어가 폭주 검증 비용으로 포화돼 정상 사용자까지 로그인 못 하는 **DoS**가 되고, OTP 발송 비용(SMS 요금) 폭탄을 맞는다.
3. **대안** — 유출 비번 차단(2번)으로 "재사용된 약한 비번" 자체를 제거하거나, passkey/WebAuthn으로 비밀번호를 아예 없애 스터핑의 전제(비번 대입)를 무력화할 수 있다. 위험 기반 인증 엔진(디바이스/지역/속도 신호 종합 점수)으로 마찰을 동적으로 거는 방식도 있다.
4. **트레이드오프** — 레이트 리밋은 너무 빡빡하면 정상 사용자(공유 IP/NAT 뒤 수천 명, 모바일 캐리어 CGNAT)를 오탐 차단하고, 너무 느슨하면 low-and-slow를 못 막는다. MFA는 **UX 마찰**(가입/로그인 이탈률 상승)과 **미적용 사용자**(MFA 미등록 계정은 여전히 노출)라는 빈틈이 있다 → 그래서 위험 기반으로 "의심스러울 때만" 걸어 마찰을 상쇄한다.

→ 가장 **근본**은 사용자의 **비밀번호 재사용** 자체다. 그건 우리가 통제할 수 없으므로(남의 사이트 습관), 우리가 할 수 있는 건 **유출 비번 차단(2번)으로 재사용된 비번을 거부**하고 **MFA로 적중의 영향을 무력화**하는 것이다. 레이트 리밋은 시간을 버는 1차 방어일 뿐 최종 해결이 아니다.

## 진단

```promql
# 로그인 실패율 — 스터핑의 1차 시그니처 (정상 5%대, 스터핑 시 90%+)
sum(rate(login_attempts_total{result="fail"}[1m]))
  / sum(rate(login_attempts_total[1m]))

# 계정당 시도 분포 — 분산성 확인 (계정당 1~2회면 brute-force 아님 = 스터핑)
sum by (account) (rate(login_attempts_total[5m]))

# 유입 IP 다양성 급증 (봇넷 분산 신호)
count(count by (client_ip) (login_attempts_total))
```

```bash
# 액세스 로그에서 단조로운 UA / 분산 IP 빠르게 확인
awk '$status==401 {print $remote_addr}' access.log | sort -u | wc -l   # 고유 IP 수 (수만 = 봇넷)
grep ' 401 ' access.log | awk -F'"' '{print $6}' | sort | uniq -c | sort -rn | head  # UA 편중
```

## 재발 방지

- **로그인 성공/실패율 · 계정당 시도 분포 대시보드** + 이상 급증 알람:
  ```yaml
  - alert: CredentialStuffingWave
    expr: sum(rate(login_attempts_total{result="fail"}[5m]))
        / sum(rate(login_attempts_total[5m])) > 0.5
    for: 3m
  ```
- **유출 비번 차단 상시화** — 가입/변경뿐 아니라 로그인 성공 시점에도 점검해 재사용 비번을 점진적으로 퇴출.
- **MFA 채택률 추적** — 미등록 계정이 노출면이므로 채택률을 KPI로 관리, 고위험 계정은 강제 등록.
- **신규 대형 유출 모니터링** — 다크웹/HIBP 신규 combo list 공개 시 사전 경계 강화, 영향 가능 계정 선제 비번 리셋.
- **침해 시 강제 비번 리셋 런북** — 적중 의심 계정 일괄 세션 무효화 + 비번 리셋 + 알림 자동화.
- **PR 룰** — 인증 경로 변경 시 다축 레이트 리밋 / 메시지 일반화 / 타이밍 일정화 검토 의무.

## 관련

- [[web]]
- [[../../20-areas/security/web-security/authentication-session|authentication-session]] — 인증/세션 취약점 원리(약한 인증·세션 고정)
- [[retry-storm-metastable]] — 로그인 폭주가 인증 DB 부하 → 재시도 폭풍으로 번짐
- [[../../20-areas/security/web-security/cryptographic-failures|cryptographic-failures]] — 유출의 근원(약한 비번 해시 저장)
- [[../troubleshooting|↑ 60-troubleshooting]]
- 참고: OWASP "Credential Stuffing Prevention Cheat Sheet"; HaveIBeenPwned Pwned Passwords (k-anonymity); NIST SP 800-63B (breached password / MFA 권고)
