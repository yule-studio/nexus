---
title: "Python + Django"
kind: area
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags:
  - area
  - backend
  - python-django
---

# Python + Django

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-05-26 | engineering-agent/tech-lead | area hub 로 승격 — guidelines / common / api-design / database / api-docs / pitfalls 연결 |

**[[../backend|↑ backend]]**

> Django 5 + DRF (Django REST Framework) + simple-jwt + Channels + Celery.
> 본 area 는 Python 으로 백엔드를 처음 시작하는 입장에서 Django 의 **batteries-included** 철학에 맞춰 ORM / admin / DRF / 인증 / 배포까지 누적.

## 학습 우선순위

이 area 를 처음 보는 사람은 다음 순서로:

1. **[[guidelines/guidelines|guidelines/]]** ← **여기부터.** 10 단계 (입문 + 기본 세팅) — Memo API 구현하며 Django 의 핵심 패턴 학습.
2. [[common/common]] — 공통 envelope / settings / exception handler 패턴
3. [[api-design/api-design]] — REST endpoint 설계 (DRF Router / nested)
4. [[database/database]] — ORM / migrations / select_related / index
5. [[api-docs/api-docs]] — drf-spectacular Swagger
6. [[pitfalls/pitfalls]] — 자주 막히는 함정 모음

## 다른 framework 비교

| 영역 | Kotlin+Spring | Flask | FastAPI | Django (← 본 area) |
| --- | --- | --- | --- | --- |
| 철학 | enterprise full-stack | micro (조립) | async-first | batteries-included |
| ORM | JPA/Hibernate | SQLAlchemy | SQLAlchemy 2.x async | **Django ORM** (내장) |
| Migration | Flyway | Flask-Migrate | Alembic | **manage.py migrate** (내장) |
| Validation | Bean Validation | marshmallow | pydantic | **DRF Serializer** |
| API doc | springdoc-openapi | flask-smorest | 자동 | **drf-spectacular** |
| Async | WebFlux (별도) | 부분 (3.x) | **기본** | 부분 (4.0+) |
| Admin | 별도 | 별도 | 별도 | **자동 (manage.py)** |

→ Django 의 강점: **admin / migration / ORM 이 처음부터 합쳐있음**.

## 영역별 노트

- **[[guidelines/guidelines|guidelines/]]** — 입문 + 기본 세팅 (메모 API)
- [[common/common]] — 공통 / 응답 envelope / 예외
- [[api-design/api-design]] — REST 설계
- [[database/database]] — ORM / migration / index
- [[api-docs/api-docs]] — drf-spectacular
- [[pitfalls/pitfalls]] — 함정

## 학습 자료

- [Django 공식](https://docs.djangoproject.com/) — 5.x
- [Django REST framework](https://www.django-rest-framework.org/)
- [drf-spectacular](https://drf-spectacular.readthedocs.io/)
- [django-environ](https://django-environ.readthedocs.io/)
- [Real Python: Django](https://realpython.com/tutorials/django/)
- [Two Scoops of Django](https://www.feldroy.com/books/two-scoops-of-django-3-x) — best practice

## 관련

- [[../backend|↑ backend]]
- [[../python-flask/python-flask|← Flask area]]
- [[../python-fastapi/python-fastapi|← FastAPI area]]
- [[../spring/kotlin-spring/kotlin-spring|↔ Kotlin+Spring area]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
