---
title: "2FA / TOTP (Google Authenticator) — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-26T23:55:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, auth, 2fa, totp]
---

# 2FA / TOTP (Google Authenticator) — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/two-factor-auth|↗ Java 원본]]** · **[[../../python-flask/api-design/two-factor-auth|↗ Flask 변형]]**

> `pyotp` + `qrcode` + `cryptography` (AES-GCM). 코드 / 모델 / 함정 대부분 Flask 와 동일 — async / Depends 만 다름.

---

## 1-3-8. 무엇을 / DB 스키마 / 함정

(Java 원본 [[../../spring/java-spring/api-design/two-factor-auth#1|§1, §3, §8]] 와 동일)

---

## 2. TOTP service

(Flask 변형 [[../../python-flask/api-design/two-factor-auth#3 TOTP service|§3]] 의 `TotpService` 그대로 — sync 라이브러리이고 cpu-bound 라 async 불필요)

---

## 3. UseCase — async

```python
# app/services/two_factor_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.domain.two_factor import UserTwoFactor, TwoFactorStatus
from app.common.totp import totp_service
from app.errors import BusinessException, NotFoundException, ConflictException, UnauthorizedException


async def enroll_2fa(db: AsyncSession, user_id: int, email: str) -> dict:
    existing = await db.scalar(select(UserTwoFactor).where(UserTwoFactor.user_id == user_id))
    if existing and existing.status == TwoFactorStatus.ENABLED.value:
        raise ConflictException("2FA 가 이미 활성화되어 있습니다.")

    secret = totp_service.generate_secret()
    encrypted = totp_service.encrypt_secret(secret)

    if existing:
        existing.secret_encrypted = encrypted
        existing.status = TwoFactorStatus.PENDING.value
    else:
        db.add(UserTwoFactor(
            user_id=user_id,
            secret_encrypted=encrypted,
            status=TwoFactorStatus.PENDING.value,
        ))
    await db.commit()

    uri = totp_service.build_otpauth_uri(email, "Shop", secret)
    qr_png = totp_service.generate_qr_png(uri)
    return {
        "otpAuthUri": uri,
        "qrDataUrl": f"data:image/png;base64,{base64.b64encode(qr_png).decode()}",
    }


async def enable_2fa(db: AsyncSession, user_id: int, code: str) -> list[str]:
    tf = await db.scalar(select(UserTwoFactor).where(UserTwoFactor.user_id == user_id))
    if tf is None or tf.status != TwoFactorStatus.PENDING.value:
        raise ConflictException("PENDING 상태가 아닙니다.")

    if not totp_service.verify(tf.secret_encrypted, code):
        raise UnauthorizedException("TOTP 코드가 유효하지 않습니다.")

    raw_codes = [_random_recovery_code() for _ in range(10)]
    hashed = [hashlib.sha256(c.encode()).hexdigest() for c in raw_codes]

    tf.status = TwoFactorStatus.ENABLED.value
    tf.enabled_at = datetime.now(timezone.utc)
    # recovery codes 별도 테이블에 저장 (생략)
    await db.commit()

    return raw_codes


async def verify_2fa(db: AsyncSession, pending_token: str, code: str) -> dict:
    claims = decode_token(pending_token)
    if claims.get("token_type") != "2FA_PENDING":
        raise UnauthorizedException("2FA pending 토큰이 아닙니다.")

    user_id = int(claims["sub"])
    tf = await db.scalar(select(UserTwoFactor).where(UserTwoFactor.user_id == user_id))
    if tf is None:
        raise NotFoundException()

    # brute force lock 체크
    if tf.locked_until and tf.locked_until > datetime.now(timezone.utc):
        raise BusinessException(403, "FORBIDDEN", "2FA 가 잠겨 있습니다.")

    if not totp_service.verify(tf.secret_encrypted, code):
        tf.failed_attempts += 1
        if tf.failed_attempts >= 5:
            tf.locked_until = datetime.now(timezone.utc) + timedelta(minutes=15)
            tf.failed_attempts = 0
        await db.commit()
        raise UnauthorizedException("유효하지 않은 인증 코드")

    tf.failed_attempts = 0
    tf.locked_until = None
    await db.commit()

    user = await db.get(User, user_id)
    return {
        "accessToken": create_access_token(str(user.id), additional={"role": user.role}),
        "refreshToken": create_refresh_token(str(user.id)),
    }
```

> **FastAPI/async idiom**: `await db.scalar(select(...))` / `await db.commit()`. CPU-bound `totp_service.verify` 는 sync — 호출 가능 (millisec).

---

## 4. Controller

```python
@router.post("/enroll")
async def enroll(
    user: CurrentUserDep,
    db: SessionDep,
):
    result = await enroll_2fa(db, user.id, user.email)
    return CommonResponse.success(ResponseCode.OK, "2FA 등록 — TOTP 코드 확인 필요", result)


@router.post("/enable", response_model=CommonResponse[dict])
async def enable(
    payload: Enable2FARequest,
    user: CurrentUserDep,
    db: SessionDep,
):
    codes = await enable_2fa(db, user.id, payload.code)
    return CommonResponse.success(
        ResponseCode.OK,
        "2FA 활성화 완료. 복구 코드 10개를 안전한 곳에 보관하세요.",
        {"recoveryCodes": codes},
    )


@router.post("/verify", response_model=CommonResponse[TokenResponse])
async def verify(payload: Verify2FARequest, db: SessionDep):
    tokens = await verify_2fa(db, payload.pendingToken, payload.code)
    return CommonResponse.success(ResponseCode.OK, "로그인 성공", TokenResponse(**tokens))


class Enable2FARequest(BaseModel):
    code: str = Field(min_length=6, max_length=6, pattern=r"^\d{6}$")


class Verify2FARequest(BaseModel):
    pendingToken: str
    code: str = Field(min_length=6, max_length=6, pattern=r"^\d{6}$")
```

---

## 5. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/two-factor-auth#8 함정 모음|§8]] + Flask 변형 함정 그대로)

### FastAPI 한정

### 함정 11 — TOTP `code` 가 int / str 혼동
앞 0 잘림. **pydantic field `pattern=r"^\d{6}$"`** 로 6자리 숫자 string 강제.

### 함정 12 — async UseCase 안 sync cryptography 호출
`totp_service.verify` 가 cryptography 라이브러리 — sync 함수. millisec 이라 OK 지만 verify 가 느리면 `asyncio.to_thread`.

---

## 6. 운영 체크리스트

(Java 원본 §9 + Python)

---

## 7. 관련

- [[../../spring/java-spring/api-design/two-factor-auth|↗ Java 원본]]
- [[../../python-flask/api-design/two-factor-auth|↗ Flask 변형]]
- [[rbac-permissions]]
- [[../common/security-config]]
- [[api-design|↑ hub]]
