---
title: "troubleshooting · CORS preflight 차단"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, cors, http, browser, security]
home_hub: "60-troubleshooting"
related: ["[[frontend]]", "[[../network/network]]"]
---

# CORS preflight — `No 'Access-Control-Allow-Origin'`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

```
Access to fetch at 'https://api.example.com/x' from origin 'https://app.example.com'
has been blocked by CORS policy: Response to preflight request doesn't pass
access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

- 브라우저 console error
- fetch / XHR 실패 (네트워크 탭에 OPTIONS 만 보이고 actual request 없음)
- Postman / curl 은 정상 (CORS 는 브라우저 only)

## 원인

브라우저가 cross-origin POST / PUT / DELETE 또는 custom header 가 있는 GET 전에 **preflight (OPTIONS)** 발송:
1. server 가 OPTIONS 응답 안 함 → 401 / 404 / 5xx
2. `Access-Control-Allow-Origin` 헤더 누락
3. `Access-Control-Allow-Methods` / `Allow-Headers` 부족
4. credentials (cookie) 요청 시 `Allow-Credentials` 또는 `Allow-Origin: *` 충돌

## 해결 — server 측 (CORS 는 server 가 권한 부여)

### Spring (Java)
```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    var c = new CorsConfiguration();
    c.setAllowedOrigins(List.of("https://app.example.com"));
    c.setAllowedMethods(List.of("GET","POST","PUT","DELETE","PATCH","OPTIONS"));
    c.setAllowedHeaders(List.of("Authorization","Content-Type"));
    c.setAllowCredentials(true);
    c.setMaxAge(3600L);   // preflight cache

    var src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/**", c);
    return src;
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .cors(c -> c.configurationSource(corsConfigurationSource()))
        .authorizeHttpRequests(a -> a
            .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()   // ← 핵심
            .anyRequest().authenticated())
        .build();
}
```

→ [[../backend/spring-security-401]] 참조.

### Express (Node)
```js
import cors from 'cors';
app.use(cors({
  origin: ['https://app.example.com'],
  methods: ['GET','POST','PUT','DELETE','PATCH'],
  credentials: true,
  maxAge: 3600,
}));
```

### Nginx
```nginx
add_header 'Access-Control-Allow-Origin' 'https://app.example.com' always;
add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
add_header 'Access-Control-Allow-Credentials' 'true' always;

if ($request_method = 'OPTIONS') {
    add_header 'Access-Control-Max-Age' 3600;
    return 204;
}
```

## Frontend workaround — Same-origin proxy

직접 server 변경 불가 시 (3rd party API) Next.js / Vite proxy:

```js
// next.config.js
async rewrites() {
  return [{ source: '/api/:path*', destination: 'https://3rdparty.com/:path*' }];
}

// vite.config.ts
server: {
  proxy: {
    '/api': { target: 'https://3rdparty.com', changeOrigin: true, rewrite: p => p.replace(/^\/api/, '') }
  }
}
```

→ 같은 origin 처럼 보임 (CORS 회피).

## Credentials 함정

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

이 조합 거절. `Allow-Origin` 은 구체적 origin 명시 필요.

```js
// client 측 cookie 전송
fetch(url, { credentials: 'include' });
```

→ server 가 정확한 origin + credentials true 응답해야 함.

## 진단

```bash
# preflight 확인
curl -i -X OPTIONS https://api.example.com/x \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: POST"

# 응답에 Access-Control-* 헤더 확인
```

DevTools Network tab:
- `OPTIONS` request 의 응답 status / 헤더
- actual request 가 보이지 않으면 preflight 실패

## 재발 방지

- API gateway 단에서 CORS 표준 설정 (모든 서비스 일관)
- 새 endpoint 추가 시 preflight 검증 (Playwright e2e)
- 모니터링: 4xx with `Origin` 헤더 알람

## 관련

- [[frontend]] / [[../backend/spring-security-401]] / [[../network/network]]
