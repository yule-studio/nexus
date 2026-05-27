# 30-resources/ — 외부 자료 요약

> 책 / 강의 / 공식 문서 / 아티클 / 논문 / 코드 예시 의 **요약 + 내
> 해석**. 자료를 다 읽는 게 아니라 핵심을 추출.

**[[../index|↑ vault 인덱스]]**

## 영역

| 카테고리          | 진입점                                                | 예시                                                        |
| ------------- | -------------------------------------------------- | --------------------------------------------------------- |
| books         | [[obsidian-vault/30-resources/books/books\|books]] | "Effective Java", "Designing Data-Intensive Applications" |
| articles      | [[articles/]]                                      | 블로그 글 / 회사 엔지니어링 블로그                                      |
| lectures      | [[lectures/]]                                      | 컨퍼런스 / 강의 영상                                              |
| official-docs | [[official-docs/]]                                 | Spring / Discord / Anthropic 등 공식 docs                    |
| papers        | [[papers/]]                                        | 논문                                                        |
| examples      | [[examples/]]                                      | 참고할 만한 오픈소스 코드                                            |

## 파일명 컨벤션

```
<topic-kebab>.md          # 단일 자료
<source>-<topic-kebab>.md # 출처 명시 (예: effective-java-chapter-5.md)
```

## 노트 표준 구조

```markdown
# <자료 제목>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <역할> | 최초 요약 |

## 출처
- 제목 / 저자 / URL
- 출판일 / 접근일

## 한 줄 요약
이 자료가 말하는 핵심 1 줄.

## 핵심 발견
- 점 1
- 점 2
- 점 3

## 내 해석 / 적용
어떻게 내 프로젝트 / 학습에 가져올지.

## 인용
> 핵심 인용 (페이지 / 시점 명시)

## 관련
- [[20-areas/.../concept-X]] (이 자료가 다루는 개념)
- [[40-patterns/.../pattern-Y]] (이 자료에서 추출한 패턴)
```

## 운영 규칙

1. **출처 + 날짜 필수**. 출처 없는 요약은 30-resources 가 아니다.
2. 단순 복붙 / 번역 금지 — 내 해석이 본문의 30% 이상.
3. 자료가 다루는 개념은 별도로 `20-areas/` 에도 정리하고 [[cross-link]].
4. 자료가 다루는 재사용 패턴은 `40-patterns/` 로 추출.
5. 자료가 deprecated 되면 `90-archive/migrated/` 로 (정보 보존).

## 관련

- [[../index|↑ vault 인덱스]]
- [[../20-areas/README|20-areas]] — 개념 정리
- [[../40-patterns/README|40-patterns]] — 추출된 패턴
