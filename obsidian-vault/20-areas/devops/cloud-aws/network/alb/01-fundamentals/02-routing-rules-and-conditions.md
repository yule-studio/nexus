---
title: "ALB 01-2 — Routing Rules + Conditions (Path / Host / Header / Query / Source IP)"
kind: knowledge
status: current
created_at: 2026-05-22T21:50:00+09:00
tags: [aws, alb, fundamentals, routing, rules, conditions, redirect]
home_hub: alb
---

# ALB Routing Rules + Conditions

> Listener Rule = Priority + Condition + Action. 5 condition + 5 action. Default rule (lowest priority).

## 1. Rule 구조

```
Listener (443)
├─ Rule 1 (priority 10): IF path=/api/v1/auth/* THEN forward tg-auth
├─ Rule 2 (priority 20): IF path=/api/v1/study/* THEN forward tg-study
├─ Rule 3 (priority 30): IF host=admin.bkurs.com THEN forward tg-admin
├─ Rule 4 (priority 40): IF source-ip=corp-cidr THEN forward tg-internal
└─ Default rule:        fixed response 404
```

## 2. 5 Condition Type

| Condition | 예 |
| --- | --- |
| **path-pattern** | `/api/v1/*`, `/admin/*` |
| **host-header** | `api.bkurs.com`, `*.bkurs.com` |
| **http-header** | `x-version: beta` |
| **http-request-method** | `GET`, `POST` |
| **query-string** | `?lang=ko`, `?version=v2` |
| **source-ip** | `10.10.0.0/24`, `203.0.113.0/24` |

## 3. 5 Action Type

| Action | 의미 |
| --- | --- |
| **forward** | Target Group 으로 (weighted 가능) |
| **redirect** | 다른 URL 로 redirect (301 / 302) |
| **fixed-response** | static HTTP response |
| **authenticate-oidc** | OIDC IdP 인증 (Google / Okta) |
| **authenticate-cognito** | Cognito User Pool 인증 |

## 4. Path-pattern Rule

```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 10 \
  --conditions '[{
    "Field": "path-pattern",
    "PathPatternConfig": { "Values": ["/api/v1/auth/*"] }
  }]' \
  --actions '[{
    "Type": "forward",
    "TargetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/tg-auth/..."
  }]'
```

### Path syntax
```
Exact:        /api/v1/login
Wildcard *:   /api/*  (multi-level)
Wildcard ?:   /api/v? (single char)
Multi:        /api/* AND /admin/*  (single rule, OR)
```

## 5. Host-header Rule

```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 30 \
  --conditions '[{
    "Field": "host-header",
    "HostHeaderConfig": { "Values": ["admin.bkurs.com","internal.bkurs.com"] }
  }]' \
  --actions '[{"Type":"forward","TargetGroupArn":"..."}]'
```

→ SNI 활용 (multi-domain on same ALB).

## 6. HTTP Header Rule

```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 40 \
  --conditions '[{
    "Field": "http-header",
    "HttpHeaderConfig": {
      "HttpHeaderName": "x-api-version",
      "Values": ["beta", "v2-preview"]
    }
  }]' \
  --actions '[{"Type":"forward","TargetGroupArn":"...beta-tg..."}]'
```

활용:
- A/B testing
- Canary header
- API versioning

## 7. Source IP Rule

```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 50 \
  --conditions '[{
    "Field": "source-ip",
    "SourceIpConfig": { "Values": ["10.10.0.0/24","203.0.113.0/24"] }
  }]' \
  --actions '[{"Type":"forward","TargetGroupArn":"...internal-tg..."}]'
```

→ Corporate IP / VPN IP 만 internal API access.

## 8. Combined Conditions (AND)

```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 60 \
  --conditions '[
    { "Field": "host-header", "HostHeaderConfig": { "Values": ["api.bkurs.com"] } },
    { "Field": "path-pattern", "PathPatternConfig": { "Values": ["/admin/*"] } },
    { "Field": "source-ip", "SourceIpConfig": { "Values": ["10.10.0.0/24"] } }
  ]' \
  --actions '[{"Type":"forward","TargetGroupArn":"...admin-tg..."}]'
```

→ host **AND** path **AND** source-ip 모두 match.

## 9. Forward with Weighted Routing (Blue-Green / Canary)

```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 5 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/*"]}}]' \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        { "TargetGroupArn": "arn:...:targetgroup/blue/...",  "Weight": 90 },
        { "TargetGroupArn": "arn:...:targetgroup/green/...", "Weight": 10 }
      ],
      "TargetGroupStickinessConfig": { "Enabled": false }
    }
  }]'
```

→ 90% Blue / 10% Green. Canary deployment.

## 10. Redirect Action

```bash
# HTTP → HTTPS
aws elbv2 create-rule --listener-arn $HTTP_LISTENER --priority 1 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/*"]}}]' \
  --actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "Host": "#{host}",
      "Path": "/#{path}",
      "Query": "#{query}",
      "StatusCode": "HTTP_301"
    }
  }]'

# www → apex
aws elbv2 create-rule --listener-arn $HTTPS_LISTENER --priority 5 \
  --conditions '[{"Field":"host-header","HostHeaderConfig":{"Values":["www.bkurs.com"]}}]' \
  --actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Host": "bkurs.com",
      "Path": "/#{path}",
      "StatusCode": "HTTP_301"
    }
  }]'

# Static site
aws elbv2 create-rule --priority 10 ... \
  --actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Host": "static.bkurs.com",
      "Path": "/#{path}",
      "StatusCode": "HTTP_302",
      "Protocol": "HTTPS"
    }
  }]'
```

## 11. Fixed Response Action

```bash
# Maintenance mode
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions '[{
    "Type": "fixed-response",
    "FixedResponseConfig": {
      "ContentType": "text/html",
      "MessageBody": "<h1>Under maintenance</h1>",
      "StatusCode": "503"
    }
  }]'

# CORS preflight
aws elbv2 create-rule --priority 1 \
  --conditions '[
    { "Field": "http-request-method", "HttpRequestMethodConfig": { "Values": ["OPTIONS"] } }
  ]' \
  --actions '[{
    "Type": "fixed-response",
    "FixedResponseConfig": {
      "StatusCode": "204"
    }
  }]'

# Health endpoint (ALB level)
aws elbv2 create-rule --priority 2 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/healthz"]}}]' \
  --actions '[{
    "Type": "fixed-response",
    "FixedResponseConfig": { "ContentType": "text/plain", "MessageBody": "OK", "StatusCode": "200" }
  }]'
```

## 12. OIDC / Cognito Auth Action

```bash
# OIDC (Google / Okta / Auth0)
aws elbv2 create-rule --priority 5 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/admin/*"]}}]' \
  --actions '[
    {
      "Type": "authenticate-oidc",
      "Order": 1,
      "AuthenticateOidcConfig": {
        "Issuer": "https://accounts.google.com",
        "AuthorizationEndpoint": "https://accounts.google.com/o/oauth2/v2/auth",
        "TokenEndpoint": "https://oauth2.googleapis.com/token",
        "UserInfoEndpoint": "https://openidconnect.googleapis.com/v1/userinfo",
        "ClientId": "...",
        "ClientSecret": "...",
        "OnUnauthenticatedRequest": "authenticate",
        "Scope": "openid email profile",
        "SessionCookieName": "AWSELBAuthSessionCookie",
        "SessionTimeout": 604800
      }
    },
    {
      "Type": "forward",
      "Order": 2,
      "TargetGroupArn": "arn:...:tg-admin"
    }
  ]'
```

→ user 의 OIDC token → ALB 가 verify → backend 에 `x-amzn-oidc-data` header 로 전달.

## 13. Rule 한도

| | |
| --- | --- |
| Rules per ALB | 100 (soft) |
| Conditions per Rule | 5 (각 5 values) |
| Listeners per ALB | 50 |
| Target Groups per ALB | 100 |

## 14. Priority

```
1-50000:        custom rules (낮을수록 먼저)
Default:        lowest priority (모든 rule 미매치 시)

평가:
- Priority 1 → 2 → ... → 50000 → Default
- 첫 match 의 action 적용
```

## 15. CDK 예

```typescript
listener.addAction('OIDC', {
  conditions: [elbv2.ListenerCondition.pathPatterns(['/admin/*'])],
  priority: 10,
  action: elbv2.ListenerAction.authenticateOidc({
    next: elbv2.ListenerAction.forward([adminTg]),
    authorizationEndpoint: '...',
    clientId: '...',
    clientSecret: cdk.SecretValue.secretsManager('oidc-secret'),
    issuer: 'https://accounts.google.com',
    tokenEndpoint: '...',
    userInfoEndpoint: '...',
  }),
});
```

## 16. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Priority 0 사용 시도 | 1+ (Default 가 lowest) |
| Path pattern `?` (single char) 모름 | `*` (multi) |
| Host 의 wildcard 가정 | `*.example.com` 명시 |
| Default action 미정의 → 404 | fixed-response 또는 forward |
| 너무 많은 rule (100+) | service 분리 또는 Path 통합 |
| Authentication action 후 forward 없음 | 2-action 명시 |

## 17. 다음

- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-https-and-acm]]
- ▶ [[../02-features/02-sticky-sessions-and-health-check]]

## 18. 영상 / 자료

| 자료 |
| --- |
| [Listener Rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html) |
| [Redirect Action](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#redirect-actions) |
| [Authentication](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html) |

## 19. 참고

- [[../alb]]
- [[01-listener-and-target-group]]
- [[../02-features/01-https-and-acm]]
