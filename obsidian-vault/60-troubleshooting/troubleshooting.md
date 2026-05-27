# 60-troubleshooting/ — 에러 해결 기록

> 만났던 에러 / 트러블의 **증상 / 원인 / 해결 / 재발 방지**. 다시
> 같은 에러를 만났을 때 빠르게 찾기 위한 grep-친화 노트.

**[[../index|↑ vault 인덱스]]**

## 영역

| 영역       | 진입점                              | 예시 에러                                               |
| -------- | -------------------------------- | --------------------------------------------------- |
| ai-agent | [[ai-agent/ai-agent\|ai-agent/]] | LLM context overflow / tool call 실패 / hallucination |
| backend  | [[backend/backend\|backend/]]    | N+1 / NPE / 트랜잭션 / 메모리 누수 / GC                      |
| database | [[database/database\|database/]] | 인덱스 미사용 / deadlock / connection pool                |
| devops   | [[devops/devops\|devops/]]       | 빌드 실패 / 배포 롤백 / secret 누락                           |
| frontend | [[frontend/frontend\|frontend/]] | hydration / re-render / CORS / event leak           |
| network  | [[network/network\|network/]]    | TLS / CORS / DNS / proxy / sandbox 차단               |

## 파일명 컨벤션

```
troubleshooting-<symptom-kebab>.md
```

또는 더 짧게 `<symptom>.md` (영역 폴더가 prefix 역할).

## 노트 표준 구조

```markdown
# <에러 / 증상 한 줄>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <역할> | 최초 |

## 증상
- 정확한 에러 메시지 (grep 용)
- 발생 환경 (버전 / OS / 의존성)
- 재현 절차

## 가설
처음에 의심한 것 + 왜 그게 아니었는지.

## 원인 (root cause)
실제 원인 1 문장 + 메커니즘 설명.

## 해결
- 즉시 대응 (workaround)
- 근본 해결 (코드 / 설정 변경)

## 재발 방지
- 회귀 테스트 / 모니터링 추가
- 정책 / 정책 문서 갱신

## 관련
- [[../20-areas/.../concept-X]] (관련 개념)
- [[../10-projects/.../task-log-Y]] (이 트러블을 만난 작업)
```

## 운영 규칙

1. **에러 메시지를 본문에 정확히 적는다** (grep 용). 한 글자라도 다르면
   못 찾는다.
2. 가설 → 원인 → 해결 순서 (디버깅 시계열). "해결만 적기" 금지 — 다음
   사람이 같은 가설을 다시 만든다.
3. 재발 방지 섹션이 비면 노트로 인정하지 않음 (회귀 테스트 / 정책 변경
   링크 필수).
4. 자동화 에이전트의 `hookify` mistake_ledger 와 cross-link (같은
   signature 시).

## 자동화 에이전트 라우팅

Troubleshooting → `60-troubleshooting/<area>/`

## 관련

- [[../index|↑ vault 인덱스]]
- [[../50-snippets/README|50-snippets]] — 트러블 해결용 조각
- yule-studio-agent repo: `hookify` 플러그인 mistake_ledger
