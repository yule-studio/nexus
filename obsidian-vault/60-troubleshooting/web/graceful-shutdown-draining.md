---
title: "troubleshooting · 배포 때마다 502/503 — 커넥션 드레이닝 부재"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, deploy, graceful-shutdown, draining, kubernetes, load-balancer]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[../devops/codedeploy-rollback]]"]
---

# 배포 때마다 502/503 — 커넥션 드레이닝 부재

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — SIGTERM/preStop/deregistration delay/keep-alive |

**[[web|↑ web]]**

## 1. 증상

- 배포·롤링 업데이트·오토스케일 **scale-in 할 때마다** 짧게 5xx(502/503) 스파이크가 뜬다.
- 클라이언트 로그에는 `connection reset by peer`, 게이트웨이/프록시 로그에는 `upstream prematurely closed connection` 또는 `upstream connect error / no healthy upstream` 이 짧게 버스트.
- "무중단 배포(rolling)"라고 설정해 놨는데도 사용자 체감 에러와 모니터링(APM/이상치 알람)에는 분명히 에러 버스트가 잡힌다.
- **트래픽이 적을 땐 거의 안 보이다가**, 부하가 있을 때(피크 시간대 배포, 동시 scale-in)에 도드라진다. 그래서 "스테이징에선 멀쩡했는데 프로덕션에서만 터진다"는 형태로 자주 보고된다.
- 배포 1회당 수백 ms~수 초 동안의 짧은 에러 윈도우라서, 1분 단위 평균 지표로 보면 묻히고 초/요청 단위로 봐야 보인다.

## 2. 원인 (root cause + 메커니즘)

인스턴스/파드를 내릴 때 **트래픽을 끊는 시점**과 **프로세스를 죽이는 시점** 사이에 여러 타이밍 경합이 있다. 핵심은 "트래픽을 먼저 끊고(드레이닝) → 그다음 종료"라는 순서가 보장되지 않아서다.

### 2.1 LB 디레지스터 지연 (새 트래픽 유입)

파드/인스턴스가 종료를 시작해도, 그 사실이 **LB·서비스 엔드포인트로 전파되는 데 수백 ms~수 초**가 걸린다. (K8s 기준: 파드 terminating → Endpoints/EndpointSlice 갱신 → kube-proxy / LB controller 반영까지 비동기.) 그 사이에 LB는 **이미 죽어가는 인스턴스로 새 요청을 계속 보낸다**. 그 요청은 이미 닫힌 리스너에 도달해 연결 거부/리셋(502/503)이 된다.

### 2.2 in-flight 요청 강제 종료 (이미 들어온 요청)

앱이 SIGTERM을 받자마자 프로세스를 즉시 끝내면, **처리 중이던 요청**이 응답 없이 끊긴다. K8s는 SIGTERM 전송 후 `terminationGracePeriodSeconds`(기본 30s)를 기다렸다가 SIGKILL을 보내는데, 앱이 이 유예시간을 활용하지 않고 바로 `process.exit()` 하거나, 반대로 유예시간보다 오래 걸리는 요청을 SIGKILL로 끊으면 동일한 증상이 난다.

### 2.3 keep-alive 경합

LB↔upstream 사이의 **idle keep-alive 커넥션**을 서버가 닫는 그 순간, 클라(LB)가 같은 커넥션으로 막 다음 요청을 올려보내면 race가 발생한다. 특히 **서버 keep-alive timeout이 LB의 idle timeout보다 짧으면**, LB는 "아직 살아있다"고 믿는 커넥션으로 요청을 보내는데 서버는 이미 그 커넥션을 닫는 중이라 리셋이 난다. 이건 배포와 무관하게 정상 운영 중에도 간헐 502를 만든다(ALB ↔ 백엔드에서 흔함).

> 정리: (2.1)은 새 트래픽 유입을 못 막아서, (2.2)는 이미 들어온 요청을 지키지 못해서, (2.3)은 커넥션 수명 합의가 어긋나서 생긴다. 세 가지가 합쳐져 "배포 때마다 5xx"가 된다.

## 3. 해결

### 3.1 preStop 훅으로 디레지스터 대기 (K8s)

파드가 SIGTERM을 받기 **직전에** preStop 훅이 실행된다. 여기서 잠깐 대기해 LB가 엔드포인트에서 이 파드를 빼는 시간을 벌고, 동시에 readiness를 false로 떨어뜨려 새 트래픽을 끊는다.

```yaml
readinessProbe:
  httpGet: { path: /readyz, port: 8080 }
  periodSeconds: 2
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 10"]   # LB deregistration 전파 대기
terminationGracePeriodSeconds: 45          # preStop(10s) + in-flight 처리 여유
```

`sleep` 값은 "엔드포인트 전파 + LB가 새 연결을 멈추는 데 걸리는 시간"을 덮어야 한다. 보통 5~15s. `terminationGracePeriodSeconds`는 `preStop 시간 + 가장 긴 in-flight 요청` 보다 커야 SIGKILL로 잘리지 않는다.

### 3.2 앱의 graceful shutdown (이미 들어온 요청 보호)

SIGTERM을 받으면 (a) 새 연결/요청 수락 중단, (b) in-flight 요청 완료까지 대기, (c) 그 후 정상 종료한다.

```js
// Node.js (Express/Fastify 류)
const server = app.listen(8080);
let shuttingDown = false;
app.get("/readyz", (_, res) => res.status(shuttingDown ? 503 : 200).end());

process.on("SIGTERM", () => {
  shuttingDown = true;                  // readiness=false → 새 트래픽 차단
  server.close(() => process.exit(0));  // 새 연결 거부 + in-flight 완료 후 종료
  setTimeout(() => process.exit(1), 30_000).unref(); // 안전망: 영원히 기다리지 않기
});
```

```go
// Go: net/http graceful shutdown
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
defer stop()
<-ctx.Done()
shutCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
_ = srv.Shutdown(shutCtx) // 새 연결 차단 + 진행 중 핸들러 완료 대기
```

Spring Boot라면 `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s` 로 동일 효과.

### 3.3 LB deregistration delay / connection draining

LB 쪽에도 드레이닝을 설정한다. ALB target group의 `deregistration_delay.timeout_seconds`(connection draining)는 타깃이 빠질 때 진행 중 요청을 끝낼 시간을 준다. K8s에서는 readiness gate로 "엔드포인트가 실제로 빠진 뒤에만 파드를 죽이도록" 보강할 수 있다.

```hcl
resource "aws_lb_target_group" "app" {
  deregistration_delay = 30   # 기본 300, 드레이닝 시간(초)
}
```

### 3.4 keep-alive 정렬

서버 keep-alive timeout을 **LB idle timeout보다 길게** 둔다(예: ALB 60s면 서버 65s). 이렇게 하면 커넥션을 항상 LB가 먼저 닫게 되어 (2.3) race가 사라진다. 종료 단계에선 응답에 `Connection: close`를 붙여 LB가 그 커넥션을 재사용하지 않게 한다.

```js
// Node: ALB idle 60s 가정
server.keepAliveTimeout = 65_000;
server.headersTimeout   = 66_000; // keepAliveTimeout 보다 약간 크게
```

## 4. 핵심 방어 — preStop 드레이닝 vs 앱 graceful shutdown (4구조)

이 둘은 자주 "둘 중 하나만 하면 되지 않냐"는 오해를 받는데, **서로 다른 경합을 막으므로 둘 다 필요**하다.

1. **왜 필요한가** — preStop 드레이닝은 *새 트래픽 유입*을 끊는 장치이고(2.1), graceful shutdown은 *이미 들어와 처리 중인 요청*을 지키는 장치다(2.2). 무중단 배포는 "더 이상 새 요청을 안 받는다 + 받은 요청은 끝까지 처리한다"가 동시에 성립해야 달성된다.
2. **안 하면 무슨 문제** — preStop만 있고 graceful shutdown이 없으면: 새 트래픽은 안 들어오지만 SIGTERM 순간 처리 중이던 요청이 끊겨 502. graceful shutdown만 있고 preStop이 없으면: in-flight는 지키지만 LB가 디레지스터 전파 전에 보낸 새 요청이 닫힌 리스너에 도달해 503. 즉 한쪽만 하면 반대쪽 경합이 그대로 남는다.
3. **대안** — (a) sidecar/mesh(예: Envoy)가 드레이닝을 대신 관리, (b) LB connection draining(3.3)만으로 버티기, (c) 헬스체크 fail 후 고정 sleep. 다만 (b)·(c)는 앱이 in-flight를 안 지키면 여전히 끊기고, mesh는 운영 복잡도가 올라간다. 그래서 **앱 graceful shutdown + preStop 조합이 가장 이식성 좋은 기본값**이다.
4. **트레이드오프** — 드레이닝/유예시간만큼 **종료가 느려진다**. 파드 1개 종료에 preStop 10s + in-flight 처리가 더해져, 롤링 배포·scale-in 전체 시간이 길어지고 긴급 롤백 속도도 늦어진다. 또 `terminationGracePeriodSeconds`를 너무 키우면 진짜 hang된 파드도 오래 살아있어 자원을 잡는다. 그래서 유예시간은 "정상 요청 p99 + 전파 여유"에 맞춰 *최소한으로* 잡고, 무한정 기다리지 않게 안전망 타임아웃(3.2의 setTimeout)을 둔다.

구체 예시: ALB idle 60s + readiness 2s 주기 환경에서 preStop sleep 10s / graceful 30s / `terminationGracePeriodSeconds` 45s 로 두면, 부하 5k rps 롤링 배포에서 5xx가 배포당 수십 건 → 0건으로 떨어진다.

## 5. 재발 방지

- **배포 중 5xx를 SLO 알람으로 노출** — 배포 윈도우라고 에러를 알람에서 제외하면, 드레이닝이 깨져도 아무도 모른다. 배포 중에도 5xx율을 1초/요청 단위로 측정.
- **부하 거는 채로 롤링 배포를 연습** — 평상시 부하(또는 합성 부하)를 주면서 배포를 돌려 에러율 0을 확인하는 게임데이를 정기화. 트래픽 없을 때 테스트하면 (2.1)·(2.3)이 절대 안 잡힌다.
- **readiness / liveness 분리** — liveness가 종료 중 readiness를 따라가면 불필요한 재시작이 생긴다. readiness만 false로, liveness는 살려둔다.
- **graceful shutdown 회귀 테스트** — SIGTERM 보낸 뒤 in-flight 요청이 200으로 끝나는지, 새 요청은 거부되는지 자동 테스트로 고정. 코드 변경으로 graceful 경로가 깨지는 걸 CI에서 잡는다.
- keep-alive timeout 정렬(3.4)을 인프라 변경 시 함께 점검 — LB idle timeout을 줄이면 서버 값도 같이 조정.

## 관련 노트

- [[web]]
- [[../devops/codedeploy-rollback]]
- [[../devops/k8s-crashloopbackoff]]
- [[../network/tcp-keepalive]]
- [[retry-storm-metastable]] — 드롭된 요청의 재시도가 부하를 증폭
- [[../troubleshooting|↑ 60-troubleshooting]]
