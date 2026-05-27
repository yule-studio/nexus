---
title: "Docker 실습 04 — Multi-stage build"
kind: practice
status: current
created_at: 2026-05-22T11:55:00+09:00
tags: [devops, docker, practice, dockerfile, multi-stage]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../dockerfile/best-practices]]"
---

# 실습 04 — Multi-stage build (image size 90% 감소)

> 시간: 30분. 학습 포인트: builder ↔ runtime 분리 / final image 가 build tool 없이 산출물만 |
> Spring Boot + Node + Go 세 가지 예제. **dive** 로 layer 검증.

## 1. 왜 multi-stage?

**single-stage 의 문제**:
- JDK + Maven + 빌드 캐시까지 final image 에 남음 → 800MB+
- 보안 표면 ↑ (불필요한 빌드 도구 노출)
- pull 시간 ↑ (CI / k8s 배포 느림)

**multi-stage 의 효과**:
- final image 는 JRE + jar 만 → 150MB
- 빌드 의존성 (npm / mvn / gcc) 사라짐
- layer 캐시 효율 ↑ (소스 변경 → 빌더만 재실행)

영상: [Docker Multi-stage build — Bret Fisher](https://www.youtube.com/watch?v=zpkqNPwEzac)

## 2. 실습 A — Spring Boot

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7

# ===== Stage 1: builder =====
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app

# 의존성 캐시 — pom 만 먼저 copy 하면 src 변경에도 dep 재다운로드 안 함
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B dependency:go-offline

# 소스 복사 + 빌드
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B package -DskipTests

# ===== Stage 2: runtime =====
FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app

# non-root user (보안)
RUN groupadd -r app && useradd -r -g app app
USER app

# builder 에서 jar 만 복사
COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

빌드 + 비교:

```bash
docker build -t myapp:single -f Dockerfile.single .
docker build -t myapp:multi -f Dockerfile .

docker images | grep myapp
# myapp  single  ...  870MB
# myapp  multi   ...  140MB
```

## 3. 실습 B — Node (React 정적 빌드)

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS runtime
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`nginx.conf`:

```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;
  location / { try_files $uri $uri/ /index.html; }   # SPA fallback
}
```

빌드:

```bash
docker build -t myreact:multi .
# final image: nginx + dist → 50MB (builder 빠진 node_modules 600MB 제외)
```

## 4. 실습 C — Go (static binary)

Go 는 **scratch / distroless** 까지 가능:

```dockerfile
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/app ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/app /app
ENTRYPOINT ["/app"]
```

final image: **15MB** (distroless: libc 없음, shell 없음 — 보안 최강).

## 5. 더 많은 stage — fan-out

```dockerfile
FROM node:20 AS web-builder
# ...

FROM golang:1.23 AS api-builder
# ...

FROM busybox AS migrations
COPY ./migrations /m

FROM gcr.io/distroless/static
COPY --from=web-builder /app/dist /web
COPY --from=api-builder /out/app /app
COPY --from=migrations /m /migrations
ENTRYPOINT ["/app"]
```

병렬 stage build — BuildKit 가 자동 DAG 분석 후 병렬.

## 6. layer 검증 — dive

```bash
brew install dive
dive myapp:multi
```

각 layer 의 크기 / 추가된 파일 / 효율 (`% efficiency`) 시각화.

또는 GUI:
- Docker Desktop > Images > {image} > "Open file system browser"
- layer 별 변경 파일 browse

## 7. 자주 묻는 함정

| 함정 | 회피 |
| --- | --- |
| 마지막 stage 가 아닌 곳에 `EXPOSE` / `ENTRYPOINT` | 최종 runtime stage 에만 |
| `FROM ... AS builder` 의 stage 이름 typo | `--from=` 로 참조 시 일치 확인 |
| `COPY --from=builder /app/target/app.jar /app/app.jar` 의 와일드카드 | shell expansion 없음 — 정확한 경로 |
| 빌드 캐시 무효화 — 자주 변하는 file 을 위로 | 의존성 (pom/package.json) 먼저, 소스는 나중 |
| Stage 간 layer 캐시 분리 | `--cache-from=type=registry,ref=my/img:cache` 사용 |

## 8. 변형 — target 지정 빌드

```bash
# builder stage 까지만 빌드 (테스트용)
docker build --target builder -t myapp:builder .

# 또는 dev stage / prod stage 분리
FROM ... AS dev
# ...
FROM ... AS prod
# ...
# docker build --target prod -t myapp:prod .
```

## 9. 검증

```bash
docker run --rm -p 8080:8080 myapp:multi
curl localhost:8080/healthcheck  # 200 OK
```

build 시간 비교:

```bash
time docker build --no-cache -t myapp:single -f Dockerfile.single .
time docker build --no-cache -t myapp:multi -f Dockerfile .
# multi-stage 가 더 빠를 수도 (병렬 stage)
```

## 10. 다음

- ▶ [[05-buildkit-cache-and-secret]] — BuildKit `--mount=type=cache,secret`
- ▶ [[06-multi-platform-build]] — buildx multi-arch
- ▶ [[../dockerfile/best-practices]] — 전체 best practice
- ▶ [[../desktop/builder-and-buildkit]] — Desktop GUI 빌드

## 참고

- 공식: https://docs.docker.com/build/building/multi-stage/
- distroless: https://github.com/GoogleContainerTools/distroless
- dive: https://github.com/wagoodman/dive
