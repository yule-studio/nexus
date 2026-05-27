---
title: "Dockerfile best practices ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:10:00+09:00
tags: [devops, docker, dockerfile, best-practice]
---

# Dockerfile best practices ★

**[[docker|↑ docker]]**

---

## 1. 10가지 원칙

| 원칙 | 적용 |
| --- | --- |
| **multi-stage** | build / runtime image 분리 |
| **layer cache 최적화** | 자주 변경 ≠ 위 |
| **.dockerignore** | node_modules / .git 제외 |
| **non-root user** | USER appuser |
| **specific tag (latest 금지)** | `eclipse-temurin:21-jre-jammy` |
| **scratch / distroless / alpine** | 최소 base |
| **HEALTHCHECK** | k8s liveness 대신 |
| **single process** | 1 컨테이너 = 1 프로세스 (supervisord X) |
| **SIGTERM 처리** | graceful shutdown |
| **labels** | OCI annotations |

---

## 2. Multi-stage (Spring Boot 예시)

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-jammy AS build
WORKDIR /workspace
COPY gradle gradle
COPY gradlew settings.gradle build.gradle ./
RUN ./gradlew dependencies --no-daemon       # cache layer
COPY src src
RUN ./gradlew bootJar --no-daemon

# Runtime stage
FROM eclipse-temurin:21-jre-jammy
RUN groupadd -r app && useradd -r -g app app
WORKDIR /app
COPY --from=build /workspace/build/libs/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseG1GC", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

→ image size: 1.5GB (JDK + source) → 300MB (JRE only).

---

## 3. Cache 최적화 — gradle/maven

```dockerfile
# X — 매번 의존성 다시 받음
COPY . .
RUN ./gradlew bootJar

# O — 의존성 layer 분리
COPY gradle gradle
COPY gradlew settings.gradle build.gradle ./
RUN ./gradlew dependencies          # 의존성 layer (변경 적음)
COPY src src
RUN ./gradlew bootJar                # 코드 변경 시 만 rebuild
```

---

## 4. Node.js 예시

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

---

## 5. .dockerignore (필수)

```
.git
.gitignore
node_modules
*.log
.env
.env.*
.idea
.vscode
target/
build/
*.iml
README.md
Dockerfile
docker-compose*.yml
```

→ build context 크기 ↓ + 보안 (.env 누설 방지).

---

## 6. Distroless (Google) — runtime 만

```dockerfile
FROM gcr.io/distroless/java21-debian12
COPY --from=build /workspace/build/libs/app.jar /app.jar
USER nonroot
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

- shell / apt / package manager X.
- CVE 90% 감소.
- 디버그 어려움 (debug variant 별도).

---

## 7. ARG vs ENV

```dockerfile
ARG VERSION=1.0       # build time only
ENV APP_VERSION=$VERSION   # runtime persistent
```

→ ARG = `docker build --build-arg` 로 inject. ENV = 컨테이너 env var.

---

## 8. 함정

1. **latest tag** — production 에서 절대 X (재현 X).
2. **root user** — 컨테이너 break out 시 host root.
3. **ADD vs COPY** — ADD 는 URL / tar auto-extract (예측 X) → COPY 만.
4. **RUN 매번 새 shell** — `cd /app && cmd` (X) → WORKDIR.
5. **secret in ARG** → image layer 에 남음 → BuildKit secret 또는 runtime 주입.

---

## 9. 관련

- [[docker|↑ docker]]
- [[security]]
- [[concepts]]
- [[practice/practice]]
