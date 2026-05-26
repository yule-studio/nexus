---
title: "03. Model + ORM + migration + Custom User"
kind: guide
status: current
created_at: 2026-05-26T10:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, orm, models, migration, custom-user]
---

# 03. Model + ORM + migration + Custom User

**[[guidelines|↑ Hub]]** ← [[02-project-structure]] · → 다음 [[04-views-and-serializers]]

> Django ORM 의 핵심 — Model / Manager / QuerySet + Custom User (필수) + migration.

---

## 1. Model 기본

```python
# apps/memos/models.py
from django.db import models
from django.conf import settings


class Memo(models.Model):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,                # ← User 직접 import X
        on_delete=models.CASCADE,
        related_name="memos",
    )
    title = models.CharField(max_length=200)
    content = models.TextField(blank=True, default="")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "memos"
        ordering = ["-created_at"]                # default 정렬
        indexes = [
            models.Index(fields=["user", "-created_at"]),
        ]
        verbose_name = "메모"
        verbose_name_plural = "메모"

    def __str__(self) -> str:
        return f"Memo({self.id}: {self.title})"
```

핵심:
- **`settings.AUTH_USER_MODEL`** 사용 (User 직접 import X — circular import 방지)
- **`on_delete`** 명시 필수 (CASCADE / SET_NULL / PROTECT / RESTRICT)
- **`related_name`** = reverse 접근 (`user.memos.all()`)
- **`db_table`** = 실제 테이블명
- **`indexes`** = composite index
- **`ordering`** = QuerySet default 정렬 (필요시)

---

## 2. Field 종류

| 필드 | 의미 |
| --- | --- |
| `CharField(max_length=N)` | VARCHAR |
| `TextField` | TEXT (긴 글) |
| `IntegerField` / `BigIntegerField` | INT / BIGINT |
| `DecimalField(max_digits, decimal_places)` | 정확한 십진수 (돈) |
| `BooleanField` | BOOL |
| `DateField` / `DateTimeField` | 날짜 / 시각 |
| `EmailField` | CharField + 이메일 validation |
| `URLField` | CharField + URL validation |
| `JSONField` | JSON (PostgreSQL native / MySQL 5.7+) |
| `ForeignKey` | N:1 |
| `OneToOneField` | 1:1 |
| `ManyToManyField` | N:N (중간 테이블 자동) |
| `FileField` / `ImageField` | 파일 업로드 |

---

## 3. migration

```bash
python manage.py makemigrations memos    # apps/memos/migrations/0001_initial.py 생성
python manage.py migrate                  # DB 에 적용
```

생성된 migration:
```python
# apps/memos/migrations/0001_initial.py
class Migration(migrations.Migration):
    initial = True
    dependencies = [
        migrations.swappable_dependency(settings.AUTH_USER_MODEL),
    ]
    operations = [
        migrations.CreateModel(
            name="Memo",
            fields=[
                ("id", models.BigAutoField(primary_key=True, ...)),
                ("title", models.CharField(max_length=200)),
                ...
            ],
            options={
                "db_table": "memos",
                "ordering": ["-created_at"],
            },
        ),
    ]
```

> migration 은 **git commit 필수.** 사람이 직접 작성 X — `makemigrations` 가 자동.

### 3.1 migration 다른 명령

```bash
# 상태 보기
python manage.py showmigrations

# 특정 migration 으로 rollback
python manage.py migrate memos 0001

# SQL 미리보기
python manage.py sqlmigrate memos 0002

# 빈 migration (수동 data migration)
python manage.py makemigrations memos --empty --name fix_data
```

### 3.2 data migration — 데이터 이행

```python
# apps/memos/migrations/0003_fix_data.py
from django.db import migrations


def fix_titles(apps, schema_editor):
    Memo = apps.get_model("memos", "Memo")        # historical model
    for memo in Memo.objects.filter(title=""):
        memo.title = "(제목 없음)"
        memo.save()


def reverse_fix(apps, schema_editor):
    pass                                          # no-op


class Migration(migrations.Migration):
    dependencies = [("memos", "0002_...")]
    operations = [migrations.RunPython(fix_titles, reverse_fix)]
```

> **`apps.get_model("memos", "Memo")`** — 직접 import X. migration 시점의 schema 사용.

---

## 4. QuerySet — ORM 사용

```python
from apps.memos.models import Memo


# 전체
Memo.objects.all()

# 조건
Memo.objects.filter(user_id=1)
Memo.objects.filter(title__icontains="hello")     # LIKE
Memo.objects.filter(created_at__gte=some_date)
Memo.objects.exclude(content="")

# 정렬
Memo.objects.order_by("-created_at")

# 단건
Memo.objects.get(id=1)                            # 없으면 DoesNotExist
Memo.objects.filter(id=1).first()                 # 없으면 None

# 페이지
Memo.objects.all()[10:20]                         # LIMIT 10 OFFSET 10

# count / exists
Memo.objects.filter(user_id=1).count()
Memo.objects.filter(user_id=1).exists()

# 생성
Memo.objects.create(user=user, title="hello", content="...")

# update (in-place)
Memo.objects.filter(id=1).update(title="new")     # SQL: UPDATE WHERE — signal X
memo.title = "new"; memo.save()                   # ORM 인스턴스 — signal O

# 삭제
Memo.objects.filter(id=1).delete()
```

### 4.1 lookup 종류

| lookup | 의미 |
| --- | --- |
| `field__exact` | `=` |
| `field__iexact` | 대소문자 무시 `=` |
| `field__contains` / `__icontains` | LIKE `%v%` |
| `field__startswith` / `__istartswith` | LIKE `v%` |
| `field__gt` / `__gte` / `__lt` / `__lte` | `>` `>=` `<` `<=` |
| `field__in` | IN |
| `field__isnull` | IS NULL |
| `field__date` / `__year` / `__month` | 날짜 부분 |

---

## 5. Custom User — **첫 migration 전 필수**

Django default `auth.User` 는 username 이 필수 + email optional. 본 가이드는 **email = id**:

```python
# apps/accounts/models.py
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models


class UserManager(BaseUserManager):
    def create_user(self, email: str, password: str, **extra) -> "User":
        if not email:
            raise ValueError("email 필수")
        email = self.normalize_email(email)
        user = self.model(email=email, **extra)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email: str, password: str, **extra) -> "User":
        extra.setdefault("is_staff", True)
        extra.setdefault("is_superuser", True)
        return self.create_user(email, password, **extra)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    name = models.CharField(max_length=50)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    objects = UserManager()

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["name"]

    class Meta:
        db_table = "users"

    def __str__(self) -> str:
        return self.email
```

`settings/base.py` 에 (이미 [[02-project-structure#3 base.py|02 §3.1]] 에 추가):
```python
AUTH_USER_MODEL = "accounts.User"
```

> **반드시 첫 `migrate` 전에 설정.** 이후 변경 = 데이터 보존하면서 옮기기가 매우 까다로움.

생성:
```bash
python manage.py makemigrations accounts
python manage.py makemigrations memos
python manage.py migrate
python manage.py createsuperuser
# Email: admin@example.com
# Name: admin
# Password: ...
```

---

## 6. select_related / prefetch_related — N+1 해결

```python
# ❌ N+1 (memos 마다 user 조회)
for memo in Memo.objects.filter(...):
    print(memo.user.email)                    # 매번 SELECT users

# ✅ select_related (1:1, N:1 — JOIN)
for memo in Memo.objects.select_related("user").filter(...):
    print(memo.user.email)                    # SELECT memos JOIN users

# ✅ prefetch_related (1:N, N:N — 2번 쿼리)
for user in User.objects.prefetch_related("memos").all():
    for memo in user.memos.all():
        print(memo.title)                     # SELECT users + SELECT memos WHERE user IN (...)
```

| 관계 | 도구 |
| --- | --- |
| ForeignKey / OneToOne (memo → user) | `select_related("user")` |
| reverse FK / ManyToMany (user → memos) | `prefetch_related("memos")` |

---

## 7. Manager — custom QuerySet

```python
class MemoQuerySet(models.QuerySet):
    def for_user(self, user) -> "MemoQuerySet":
        return self.filter(user=user)

    def published(self) -> "MemoQuerySet":
        return self.filter(is_published=True)


class Memo(models.Model):
    ...
    objects = MemoQuerySet.as_manager()


# 사용
Memo.objects.for_user(user).published().order_by("-created_at")
```

→ chainable. business logic 을 model 에 두기 권장.

---

## 8. transaction.atomic

```python
from django.db import transaction


@transaction.atomic
def signup_with_initial_memo(email: str, password: str, name: str) -> User:
    user = User.objects.create_user(email=email, password=password, name=name)
    Memo.objects.create(user=user, title="첫 메모", content="환영합니다")
    return user
```

→ 함수 안 에러 시 모두 rollback.

### 8.1 부분 commit / savepoint

```python
with transaction.atomic():
    user = User.objects.create_user(...)
    try:
        with transaction.atomic():           # savepoint
            send_welcome_email(user)
    except EmailError:
        pass                                  # 이메일 실패해도 user 는 commit
```

---

## 9. raw SQL — 마지막 수단

```python
# 1) Manager.raw
memos = Memo.objects.raw("SELECT * FROM memos WHERE user_id = %s", [user.id])

# 2) connection.cursor (model 매핑 X)
from django.db import connection
with connection.cursor() as cur:
    cur.execute("SELECT COUNT(*) FROM memos")
    (count,) = cur.fetchone()
```

> **placeholder** (`%s`) 필수. f-string = SQL injection.

---

## 10. 자주 막히는 함정

### 함정 1 — `User` 직접 import
```python
from django.contrib.auth.models import User       # ❌ Custom User 와 충돌
```
**`from django.contrib.auth import get_user_model`** 또는 model FK 에선 `settings.AUTH_USER_MODEL`.

### 함정 2 — `AUTH_USER_MODEL` 늦게 변경
이미 migrate 후 변경 = 매우 까다로움. **시작 전 결정**.

### 함정 3 — N+1
관계 접근 = SQL 자동 발생. **`select_related` / `prefetch_related` 의 의식적 사용**. dev 환경에선 django-debug-toolbar 로 확인.

### 함정 4 — `Memo.objects.filter(...).update(field=v)` vs 인스턴스 `save()`
`.update` = SQL 직접 + signal / `save()` 메소드 안 탐. 의도 구분.

### 함정 5 — migration 충돌 (다중 개발자)
두 사람이 동시에 makemigrations → 0002 두 개. **`python manage.py makemigrations --merge`** 또는 직접 conflict 해결.

### 함정 6 — `auto_now_add=True` vs `default=timezone.now`
`auto_now_add` = 첫 저장 시 자동 + 변경 X. `default=` = 사용자가 override 가능.

### 함정 7 — `ordering = ["-created_at"]` 운영
모든 QuerySet 이 자동 ORDER BY → 무거움. 명시 사용 권장.

### 함정 8 — `JSONField` MySQL 8 이하
MySQL 5.7+ 만 native. 그 이하면 `TextField` + 직접 직렬화.

### 함정 9 — `on_delete=models.CASCADE` 무의식
부모 삭제 = 자식 모두 사라짐. **PROTECT / SET_NULL** 권장 경우 검토.

### 함정 10 — migration squash 후 conflict
`squashmigrations` 후 새 migration 이 squashed 와 충돌 가능. 큰 프로젝트는 신중히.

### 함정 11 — `objects.get()` 가 None 안 반환
없으면 `DoesNotExist`. 다건이면 `MultipleObjectsReturned`. **`filter().first()`** 가 안전.

### 함정 12 — `select_related` 가 ManyToMany / reverse FK
**`prefetch_related`** 사용. `select_related` 는 1:1 / N:1.

---

## 11. 다음

→ [[04-views-and-serializers]] — DRF ViewSet / Serializer / Router / validation
