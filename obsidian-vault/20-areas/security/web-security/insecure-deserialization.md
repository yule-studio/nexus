---
title: "Insecure Deserialization & 무결성 실패 (OWASP A08)"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, deserialization, integrity, rce, supply-chain, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[injection]]", "[[../../devops/security-ops/supply-chain-security|supply-chain-security]]"]
---

# Insecure Deserialization & 무결성 실패 (OWASP A08)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — 역직렬화 RCE/가젯체인 + 무결성/공급망 |

**[[web-security|↑ web-security]]**

> OWASP A08(Software & Data Integrity Failures). 신뢰 못할 직렬화 데이터를 객체로 복원하다 코드 실행, 또는 무결성 미검증 업데이트로 악성 코드 유입.

## 1. 안전하지 않은 역직렬화(insecure deserialization)

### 1.1 원리

직렬화(serialization)는 메모리 객체를 전송/저장 가능한 바이트열로 바꾸는 것이고, 역직렬화(deserialization)는 그 바이트열을 다시 객체로 복원하는 것이다. 문제는 복원이 단순히 "값을 채우는" 작업이 아니라는 데 있다. 많은 언어 런타임은 바이트열에 적힌 **클래스 이름과 타입 정보**를 그대로 믿고, 그 타입의 생성자/초기화 훅/매직 메서드를 실행하면서 객체를 되살린다. 따라서 공격자가 직렬화 바이트를 통제하면 **어떤 클래스의 객체를 어떤 필드값으로 복원할지**를 통제할 수 있고, 복원 과정에서 자동 호출되는 메서드들(역직렬화 콜백, 소멸자 등)의 부작용을 의도대로 엮을 수 있다.

이때 핵심 기법이 **가젯 체인(gadget chain)**이다. 단일 클래스 하나로는 RCE가 안 되더라도, 클래스패스에 이미 존재하는 라이브러리들의 "쓸 만한 부작용을 가진 메서드(gadget)"들을 역직렬화 콜백 → 메서드 호출 → 리플렉션 호출 → `Runtime.exec` 식으로 줄줄이 연결하면 임의 코드 실행에 도달한다. 공격자는 새 코드를 주입하는 게 아니라 **이미 깔린 코드 조각을 재료로 조립**한다 — 그래서 의존성에 무엇이 깔려 있느냐가 곧 공격 표면이다.

### 1.2 언어별 위험 지점

- **Java** — `ObjectInputStream.readObject()`. 클래스패스에 가젯 라이브러리(대표적으로 Apache Commons Collections의 `InvokerTransformer` 계열)가 있으면 역직렬화만으로 RCE. **ysoserial** 도구가 라이브러리별 가젯 체인 페이로드를 자동 생성한다. 2015~2016년 이 계열 취약점이 WebLogic / JBoss / Jenkins / WebSphere 등에 광범위하게 터졌다.

```java
// 취약 — 신뢰 못할 바이트를 그대로 readObject
ObjectInputStream ois = new ObjectInputStream(socket.getInputStream());
Object obj = ois.readObject();   // 가젯 체인이 복원되며 코드 실행

// 완화 — 역직렬화 전 클래스 allowlist (look-ahead deserialization)
ObjectInputStream safe = new ObjectInputStream(in) {
    @Override protected Class<?> resolveClass(ObjectStreamClass d)
            throws IOException, ClassNotFoundException {
        if (!ALLOWED.contains(d.getName()))
            throw new InvalidClassException("blocked: " + d.getName());
        return super.resolveClass(d);
    }
};
```

- **PHP** — `unserialize()` 로 인한 **객체 주입(object injection)**. 복원 시 자동 실행되는 `__wakeup()` / `__destruct()` 매직 메서드를 발판으로 파일 삭제, 임의 메서드 호출 등을 엮는다.

```php
// 취약 — 신뢰 못할 쿠키/파라미터를 unserialize
$obj = unserialize($_COOKIE['data']);   // __wakeup/__destruct 부작용 트리거
```

- **Python** — `pickle.loads()` 는 **설계상** 임의 코드 실행이 가능하다. pickle 포맷 자체가 `__reduce__` 로 "복원 시 호출할 콜러블"을 지정할 수 있어, 신뢰 못할 pickle 은 사실상 원격 셸과 같다. `yaml.load()` 도 (SafeLoader 미지정 시) 임의 객체를 생성하므로 위험.

```python
import pickle, yaml
pickle.loads(untrusted_bytes)          # 임의 코드 실행 가능 — 절대 금지
yaml.safe_load(untrusted_text)         # OK — 기본 yaml.load 는 unsafe
```

- **.NET** — `BinaryFormatter` / `SoapFormatter` / `LosFormatter` 등 타입 정보를 신뢰하는 포매터. Microsoft 가 `BinaryFormatter` 를 **폐기(obsolete)** 권고하고 신규 코드 사용을 막았다.

## 2. 무결성 실패(data/software integrity)

A08 은 역직렬화에 더해 **소프트웨어/데이터 무결성 검증 실패**를 함께 묶는다. 코드·설정·업데이트가 "정말 우리가 의도한 그것인지"를 검증하지 않고 신뢰하는 모든 경로가 대상이다.

- **서명/출처 검증 없는 자동 업데이트·플러그인·CI 산출물** → 공급망 침투의 입구. 실제 사례로 **SolarWinds(2020)** — 공격자가 **빌드 파이프라인 자체**를 침해해 Orion 제품의 빌드 단계에 백도어(SUNBURST)를 끼워 넣었고, 그 결과 *정상적으로 코드 서명된* 업데이트가 수천 고객에게 정식 배포됐다. 시사점은 분명하다 — 무결성을 빌드/배포 경로에서 검증하지 않으면, 최종 산출물의 서명조차 무력화된다(서명기가 침해된 빌드 결과물에 서명해 주므로).
- **신뢰 못할 출처의 JSON/JWT 무검증 신뢰** — 역직렬화 RCE 와는 다르지만 "신뢰 못할 데이터를 검증 없이 신뢰"라는 같은 축이다. 예: 서명 검증 없이 JWT 클레임을 권한 판단에 사용, 스키마 검증 없이 외부 JSON 을 내부 도메인 객체로 매핑.

## 3. 방어

### 3.1 (4구조) 왜 "신뢰 못할 데이터는 네이티브 역직렬화 대신 데이터 전용 포맷(JSON)+스키마 검증"인가

- **① 왜 필요** — 가젯 체인은 "임의 클래스의 객체와 콜백을 복원"할 수 있어야 성립한다. JSON 같은 데이터 전용 포맷은 문자열/숫자/배열/맵만 표현하고 **타입·코드를 복원하지 않으므로** 복원 과정에서 실행될 부작용 메서드가 애초에 없다.
- **② 안 하면 문제** — `readObject`/`unserialize`/`pickle.loads` 를 외부 입력에 직접 쓰면, 입력 바이트 한 줄로 RCE 에 이른다. 입력 검증을 아무리 해도 "복원 자체가 실행"이라 검증 시점이 너무 늦다.
- **③ 대안** — 부득이 네이티브 역직렬화가 필요하면 (a) **타입 allowlist**(look-ahead / resolveClass 차단, 위 Java 예시), (b) 별도 샌드박스/프로세스 격리, (c) 역직렬화 입력에 무결성 서명(아래 3.3).
- **④ 트레이드오프** — JSON+스키마로 가면 다형성/순환 참조/객체 그래프 같은 네이티브 직렬화의 편의를 잃는다. 다만 서비스 간 통신에서 그런 편의는 대부분 불필요하며, 명시적 DTO + 스키마가 오히려 계약을 분명히 한다.

### 3.2 (4구조) 왜 "산출물 서명 + 빌드 출처(provenance) 검증"인가

- **① 왜 필요** — SolarWinds 처럼 **빌드 경로 자체가 신뢰 불가**할 수 있다. 최종 바이너리만 서명·검증해서는 "서명되기 전에 누가 끼워 넣었는가"를 잡지 못한다. 그래서 무결성 검증을 소비 시점이 아니라 **생성 시점(어떤 소스·빌더로 어떤 입력에서 나왔는가)** 까지 끌어올려야 한다.
- **② 안 하면 문제** — 침해된 파이프라인이 백도어를 심은 채 정식 서명·배포를 수행하면, 다운스트림은 그것을 "정상"으로 신뢰해 광범위하게 전파된다.
- **③ 대안** — SLSA 출처 보증, 아티팩트 서명(Sigstore/cosign), 재현 가능 빌드(reproducible build), 의존성 lockfile + 해시 핀. 깊이는 [[../../devops/security-ops/supply-chain-security|supply-chain-security]] 로 위임.
- **④ 트레이드오프** — provenance/서명 파이프라인 구축·운영 비용과 빌드 복잡도가 늘지만, 공급망 사고 1건의 영향 범위(전 고객)와 비교하면 정당화된다.

### 3.3 보조 방어

- 역직렬화 입력에 **무결성 서명(HMAC)** 을 붙여, 우리가 만든 데이터만 복원하도록 강제(외부가 임의 바이트를 못 끼움).
- 의존성에서 **가젯 라이브러리 최소화** — 안 쓰는 직렬화/리플렉션 라이브러리를 클래스패스에서 제거해 공격 재료를 줄인다.
- 역직렬화 엔드포인트 호출량·예외·신규 클래스 로드를 **모니터링**해 이상 징후를 탐지.

## 4. cross-link

- [[injection]] — "신뢰 못할 데이터가 코드가 된다"는 같은 뿌리(데이터/코드 경계 붕괴)의 다른 표면.
- [[../../devops/security-ops/supply-chain-security|supply-chain-security]] — 빌드/배포 무결성, provenance, SLSA 의 깊이 위임 대상.
- [[vulnerable-components]] — 가젯 체인의 재료가 되는 취약/불필요 의존성 관리.
- [[../security|↑ security]] — 보안 영역 허브.
