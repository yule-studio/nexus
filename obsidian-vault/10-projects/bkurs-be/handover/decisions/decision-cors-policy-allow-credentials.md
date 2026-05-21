---
title: "B2B 결정 — CORS 정책 (allow-credentials + origin 명시)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T04:10:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, cors, security, auth]
related:
  - "[[../bkurs-be]]"
  - "[[decision-refresh-token-rotation-cookie]]"
home_hub: bkurs-be
---

# B2B 결정 — CORS 정책 (allow-credentials + origin 명시)

## 결론

- Spring 의 `CorsConfigurationSource` bean 으로 통합 — `WebMvcConfigurer` 의 `addCorsMappings()` 와 Spring Security 의 `cors(Customizer.withDefaults())` 둘 다 같은 source 사용.
- `allowedOriginPatterns` 사용 (NOT `allowedOrigins`) — `allowCredentials=true` 와 `*` 조합이 금지되었기 때문. patterns 는 prod 도메인 wildcard 허용.
- profile 별 분리: `cors.allowed-origins` 를 application-{local,prod}.yml 에 정의.

## 왜 필요한가

- bkurs-fe (vite dev: `http://localhost:5173`) 와 bkurs-be (`http://localhost:8080`) 는 origin 이 다름 → 기본 same-origin 정책 위반.
- refresh cookie 송수신을 위해 `Access-Control-Allow-Credentials: true` 필수.
- prod 에서는 서브도메인 분리 가능성 (`app.bkurs.kr` ↔ `api.bkurs.kr`) → CORS 정책이 그대로 필요.

## 안 하면 뭐가 문제

- 브라우저가 preflight (OPTIONS) 에서 차단 → 모든 POST/PUT/DELETE 실패.
- credentials 옵션 없이는 refresh cookie 가 cross-origin 으로 전송 안 됨 → refresh 흐름 깨짐.
- 와일드카드 `*` + `allow-credentials=true` 는 Spring 6 / 모던 브라우저 모두 거부 (CVE 회피).

## 4 대안 비교

### A. CORS 미설정 (Spring 기본) — 기각

- 단점: dev 부터 안 됨. bkurs-fe 가 mock 모드로만 동작.
- **기각 사유**: 통합 자체 불가.

### B. `*` allowedOrigins + allowCredentials=false — 기각

- 장점: 가장 느슨, 빠른 prototype.
- 단점: refresh cookie 송수신 불가. localStorage refresh 로 회귀.
- **기각 사유**: refresh 정책과 충돌.

### C. application.yml 의 `cors.allowed-origins` 리스트 + Filter 직접 작성 — 기각

- 장점: 명시적.
- 단점: Spring 의 `CorsConfigurationSource` 와 중복. Spring Security 의 cors() 와 통합 어려움.
- **기각 사유**: 표준 Spring 메커니즘과 어긋남.

### D. `CorsConfigurationSource` bean + `allowedOriginPatterns` + profile 별 yml — **채택**

- 장점:
  - WebMvc + Spring Security 모두 같은 source 참조 → SSoT.
  - patterns 가 wildcard 허용 (`https://*.bkurs.kr`) → prod 서브도메인 추가에 강함.
  - profile 분리로 local/prod 격리.
- 단점: 약간의 boilerplate (config 클래스 1개).
- **채택 사유**: Spring 공식 가이드 + 모든 production 베스트 프랙티스의 표준.

## 설정 형태

application-local.yml:
```yaml
cors:
  allowed-origin-patterns:
    - http://localhost:5173
    - http://127.0.0.1:5173
  allowed-methods: [GET, POST, PUT, DELETE, PATCH, OPTIONS]
  allowed-headers: [Authorization, Content-Type, X-Requested-With]
  allow-credentials: true
  max-age: 3600
```

application-prod.yml (예정):
```yaml
cors:
  allowed-origin-patterns:
    - https://app.bkurs.kr
    - https://*.bkurs.kr   # 향후 학원별 서브도메인 대비
  ...
```

`CorsConfig` 클래스:
```java
@Configuration
@ConfigurationProperties(prefix = "cors")
@Setter
public class CorsConfig {
    private List<String> allowedOriginPatterns;
    private List<String> allowedMethods;
    private List<String> allowedHeaders;
    private boolean allowCredentials;
    private long maxAge;

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration cfg = new CorsConfiguration();
        cfg.setAllowedOriginPatterns(allowedOriginPatterns);
        cfg.setAllowedMethods(allowedMethods);
        cfg.setAllowedHeaders(allowedHeaders);
        cfg.setAllowCredentials(allowCredentials);
        cfg.setMaxAge(maxAge);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", cfg);
        return source;
    }
}
```

`SecurityConfig` 보강:
```java
http.cors(Customizer.withDefaults())   // CorsConfigurationSource bean 자동 사용
```

## 회귀

- `RefreshTokenApiTest` 의 cookie 검증 항목이 CORS preflight 통과 후의 동작을 검증 (TestRestTemplate 은 preflight 안 보내므로, preflight 자체는 별도 `CorsConfigTest` 에서 OPTIONS 요청 검증).

## 관련

- [[decision-refresh-token-rotation-cookie]] — cookie 정책 (이 결정의 사용자)
- [[../bkurs-be]] — PR 시리즈
