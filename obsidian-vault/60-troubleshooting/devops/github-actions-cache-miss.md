---
title: "troubleshooting · GitHub Actions cache miss — 매 빌드 의존성 재다운"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, github-actions, cache, ci]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[docker-build-cache-miss]]"]
---

# GitHub Actions cache miss

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

- workflow 가 매번 npm install / mvn dependency 5-10분
- `actions/cache` step log 에 "Cache not found"
- 다른 branch 에서 cache 가 있는데 못 가져옴

## 원인

`actions/cache` 의 key 가 매번 달라서 (hashFiles 의 lockfile path 잘못) 또는 restore-keys 누락.

## 해결

### 정확한 key + restore-keys
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

- `key` exact match — 정확한 lockfile hash
- `restore-keys` — prefix match (`-npm-` 로 시작하는 가장 최근 cache)

→ lockfile 변경 시: exact miss → restore-keys 가 부분 hit → npm install 짧음.

### setup-* action 의 cache 옵션 (권장)
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: 'npm'           # 자동 npm cache
    cache-dependency-path: 'package-lock.json'

- uses: actions/setup-java@v4
  with:
    distribution: 'temurin'
    java-version: '21'
    cache: 'gradle'        # 자동 gradle cache

- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

→ 가장 단순. 표준 cache key 자동 생성.

### Branch scope (GHA cache 의 함정)
- PR branch 는 base branch (main) 의 cache 만 read 가능
- main 의 cache 는 모든 branch 가 read
- **다른 PR branch 끼리는 cache 공유 X**

→ main 에 자주 push 되면 cache 항상 hit. PR 만의 cache 는 같은 PR 내에서만.

### Cache 크기 / 한도
- repo 당 10GB
- 7일 idle 후 자동 삭제
- LRU eviction

```yaml
# 큰 cache 는 분리
- uses: actions/cache@v4
  with:
    path: ~/.gradle/caches
    key: gradle-${{ hashFiles('**/*.gradle*') }}

- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: maven-${{ hashFiles('**/pom.xml') }}
```

### Docker layer cache 별도
[[docker-build-cache-miss]] — `docker/build-push-action` 의 `cache-from: type=gha`.

## 진단

```yaml
- uses: actions/cache@v4
  id: cache
  with: ...

- run: |
    if [ "${{ steps.cache.outputs.cache-hit }}" = "true" ]; then
      echo "Cache hit"
    else
      echo "Cache miss — will re-install"
    fi

- if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci   # cache miss 시만 실행
```

```bash
# CLI 로 cache 목록
gh cache list

# Cache 삭제
gh cache delete <key>
```

## CI 자체 cache vs custom

| 옵션 | 장점 | 단점 |
| --- | --- | --- |
| `setup-X` 의 `cache: true` | 단순, 표준 | 세밀한 제어 X |
| `actions/cache` | 유연 | key 설계 필요 |
| Docker layer cache (`type=gha`) | image 빌드 | layer 만 |
| Buildkit remote cache (S3) | 큰 빌드 | 비용 |

## 재발 방지

- `actions/setup-*` 의 `cache` 옵션 우선 (단순함)
- lockfile (`package-lock.json` / `yarn.lock`) commit 의무
- hashFiles glob 정확히 (monorepo 면 `**/`)
- CI 빌드 시간 모니터링 — spike 시 cache miss 의심

## 관련

- [[devops]] / [[docker-build-cache-miss]]
