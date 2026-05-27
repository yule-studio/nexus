---
title: "troubleshooting · OutOfMemoryError: Metaspace"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, jvm, gc, memory, metaspace, classloader]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[oom-heap]]"
---

# OutOfMemoryError: Metaspace

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — classloader leak / devtools / hot-reload |

## 증상

```
java.lang.OutOfMemoryError: Metaspace
    at java.lang.ClassLoader.defineClass1(Native Method)
    ...
```

- dev 환경에서 hot-reload 를 10-20회 반복 후 발생
- spring-boot-devtools 가 자동 restart 마다 새 classloader 로딩
- production 에서는 deploy 후 시간이 지나며 서서히 증가하다가 OOM
- 또는 cglib / asm / scripting (groovy / nashorn / kotlin script) 활발한 앱

## 가설

- 처음에는 heap 문제로 의심 → `-Xmx` 키워도 해결 안 됨
- 다음에는 GC tuning → 영향 없음
- 실제: heap 이 아닌 **Metaspace** 영역 누수 — class metadata / classloader

## 원인 (root cause)

Metaspace 는 JVM 의 클래스 metadata 저장소 (Java 8 부터 PermGen 대체). 다음이 누수 원인:

```
1. ClassLoader leak — 이미 사용 안 하는 classloader 가 GC 안 됨
   → 그 classloader 가 로드한 모든 class metadata 도 살아있음

2. ThreadLocal / static reference 가 옛 class 보유
   → 새 deploy / reload 후에도 이전 버전 class 가 살아있음

3. Dynamic proxy / cglib / scripting 가 익명 class 끊임없이 생성

4. spring-boot-devtools 가 매 restart 마다 새 classloader
```

대표 패턴 (Tomcat 호스팅 시):
```java
// 어떤 라이브러리가 ThreadLocal 에 자기 객체 set
public class LibThreadLocal {
    private static final ThreadLocal<Object> CONTEXT = new ThreadLocal<>();
}
// undeploy 시 ThreadLocal 가 remove 안 되면 → Thread (Tomcat worker) 가 옛 classloader 객체 참조 → classloader 못 GC
```

## 해결

### 즉시 (workaround)

```
-XX:MaxMetaspaceSize=512m       # 한도 명시 (무한 증가 차단)
-XX:MetaspaceSize=256m          # 초기 크기
```

→ OOM 은 여전. 다만 heap 과 분리해서 모니터 가능.

### 근본

**(1) ClassLoader leak 추적** — Eclipse MAT

```
1. jmap -dump:format=b,file=meta.bin <pid>
2. Eclipse MAT 열기
3. Histogram → java.lang.ClassLoader
4. 같은 classloader 가 여러 개 → leak
5. Path to GC Roots → 누가 살려두는지 (보통 ThreadLocal / static collection)
```

**(2) ThreadLocal cleanup**

```java
public class RequestContext {
    private static final ThreadLocal<RequestContext> CONTEXT = new ThreadLocal<>();

    public static void set(RequestContext ctx) {
        CONTEXT.set(ctx);
    }

    public static void clear() {
        CONTEXT.remove();  // ← 필수
    }
}

// Servlet filter
public class ContextFilter extends OncePerRequestFilter {
    protected void doFilterInternal(...) throws ... {
        try {
            RequestContext.set(...);
            chain.doFilter(req, res);
        } finally {
            RequestContext.clear();  // ← 반드시 finally
        }
    }
}
```

**(3) DevTools / Hot-reload 영역 분리**

`spring-boot-devtools` 는 두 classloader 분리 (base + restart). restart classloader 만 매번 새로 생성. 의도된 동작이지만 무한 reload 면 누수 가능.

```yaml
# dev profile 만 enable
spring:
  devtools:
    restart:
      enabled: true   # dev only
      additional-paths: src/main/java
```

prod build (`-Pprod`) 에서는 devtools dependency 제외:
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <optional>true</optional>
</dependency>
```

**(4) cglib / dynamic proxy 캐시 확인**

```java
// Spring proxy 캐시는 자동 정리되지만 직접 ByteBuddy / cglib 사용 시
ProxyConfig config = new ProxyConfig();
// 같은 target 에 반복 proxy 생성 금지 — singleton 으로
```

**(5) Scripting engine cache**

```java
// 잘못된 사용 — request 마다 새 engine
ScriptEngine eng = mgr.getEngineByName("javascript");

// 옳음 — singleton + compile cache
@Bean
public ScriptEngine scriptEngine() {
    return new ScriptEngineManager().getEngineByName("javascript");
}
```

## 진단 명령

```bash
# 현재 metaspace 사용량
jcmd <pid> VM.metaspace
jstat -gcmetacapacity <pid>

# Live class count
jcmd <pid> GC.class_histogram | head -30
jcmd <pid> VM.class_hierarchy java.lang.ClassLoader

# Classloader 개수
jcmd <pid> GC.class_stats | grep -i classloader

# 누가 살려두는지
jcmd <pid> GC.run                       # GC 강제
jcmd <pid> GC.heap_dump /tmp/heap.bin   # dump
# → MAT 분석
```

## 재발 방지

- **JVM 옵션 표준**:
  ```
  -XX:MaxMetaspaceSize=512m
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/app/
  -XX:+ExitOnOutOfMemoryError
  ```
- **Micrometer 모니터링**:
  ```
  jvm_memory_used_bytes{area="nonheap",id="Metaspace"}
  - alert: MetaspaceHigh
    expr: jvm_memory_used_bytes{id="Metaspace"} / jvm_memory_max_bytes{id="Metaspace"} > 0.85
  ```
- **ArchUnit 룰**: 새 `ThreadLocal` 추가 시 finally cleanup 검증 (PR template 체크리스트)
- **Dev / prod profile 분리**: devtools 는 dev 만, prod 빌드 dependency 제외 (Maven `<scope>runtime</scope>` + profile)

## 관련

- [[backend]]
- [[oom-heap]] — heap OOM (다른 영역)
- [[../../20-areas/backend/jvm/jvm|20-areas/jvm]]
