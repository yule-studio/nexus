---
title: "troubleshooting · GitHub Actions secret undefined"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, github-actions, ci, secret, oidc]
home_hub: "60-troubleshooting"
related: ["[[devops]]"]
---

# GitHub Actions secret 가 빈 문자열

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```yaml
- run: echo "key=${{ secrets.API_KEY }}"
# 출력: key=***       (정상)
# 또는: key=          (빈 문자열 — 문제)
```

- 같은 workflow 가 main branch 에서는 정상, fork PR 에서 빈 값
- environment 별로 다름
- secret 이름 오타

## 원인

1. **Fork PR** — `pull_request` event 는 fork 의 secret 접근 불가 (보안)
2. **Environment 차이** — `environment: prod` 의 secret 은 prod env 만
3. **Repo / org / env 의 secret hierarchy**
4. **Branch protection** — 특정 branch 만 secret 허용
5. **이름 오타** — `secrets.API_KEY` vs `secrets.api_key` (case sensitive)

## 해결

### Repo secret 확인
Repository → Settings → Secrets and variables → Actions
- Repository secrets
- Environment secrets (production / staging 별)
- Organization secrets (cross-repo)

### Environment 명시
```yaml
jobs:
  deploy:
    environment: production    # ← prod env 의 secret 사용
    runs-on: ubuntu-latest
    steps:
      - run: deploy --key ${{ secrets.PROD_API_KEY }}
```

### Fork PR 처리 (위험!)
```yaml
on:
  pull_request_target:    # fork PR 도 secret 접근 가능 — 위험
    types: [opened, synchronize]
```

→ `pull_request_target` 은 fork code 가 secret 으로 접근 가능 → supply chain attack 위험. 신중히 사용 (수동 approval gate 추천).

권장:
```yaml
on:
  pull_request:    # fork PR — secret 없이 빌드만
  push:
    branches: [main]   # main push — secret 사용 가능
```

### Secret 값이 정말 들어있는지
```yaml
- run: |
    if [ -z "${{ secrets.API_KEY }}" ]; then
      echo "::error::API_KEY is empty"
      exit 1
    fi
    echo "API_KEY length: ${#API_KEY}"
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

→ ${{ secrets.X }} 가 빈 문자열이어도 `set -e` 가 안 잡음.

### OIDC 사용 (secret 대신 — 권장)
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/github-actions
      aws-region: ap-northeast-2
```

→ AWS access key secret 없이 IAM role assume. rotation 불필요.

## Secret naming 함정

```yaml
# Repo / env / org 우선순위 (높은 → 낮은):
# 1. Environment secret
# 2. Repository secret
# 3. Organization secret

# 같은 이름이면 environment 가 우선.
```

## 진단

```yaml
# 모든 env 변수 마스킹된 형태로
- run: env | grep -i api

# Secret 정의 확인 (CLI)
# gh secret list
# gh secret list --env production
# gh variable list   # variables (secret 아닌 것)
```

## 재발 방지

- OIDC 우선 (AWS / GCP / Azure 모두 지원)
- secret 이름 표준 (UPPER_SNAKE_CASE)
- workflow 시작 시 secret presence check (`-z` 검증)
- secret rotation 정책 (90일)
- CodeQL / secret scanning 활성화

## 관련

- [[devops]] / [[ssh-key-permission]]
