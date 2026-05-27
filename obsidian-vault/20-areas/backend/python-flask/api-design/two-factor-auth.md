---
title: "2FA / TOTP (Google Authenticator) — Flask"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T17:25:00+09:00
tags: [backend, python-flask, api-design, recipe, auth, 2fa, totp]
---

# 2FA / TOTP (Google Authenticator) — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/two-factor-auth|↗ Java 원본]]**

> `pyotp` + `qrcode` 기반.

---

## 1. 무엇을 / 1.1 흐름 / 1.2 비기능 / 3. DB 스키마 / 8. 함정

(Java 원본 [[../../spring/java-spring/api-design/two-factor-auth#1|§1, §3, §8]] 와 동일)

---

## 2. 의존성

```python
pip install pyotp qrcode[pil] cryptography
```

---

## 3. TOTP service

```python
# app/common/totp.py
import pyotp
import qrcode
import io
import base64
from cryptography.fernet import Fernet  # 또는 AES-GCM 직접


class TotpService:
    def __init__(self, encryption_key: bytes) -> None:
        # encryption_key 는 vault / KMS 에서. base64 32 bytes.
        self._fernet = Fernet(base64.urlsafe_b64encode(encryption_key))

    def generate_secret(self) -> str:
        return pyotp.random_base32()                         # 32 char base32

    def build_otpauth_uri(self, user_email: str, issuer: str, secret: str) -> str:
        return pyotp.totp.TOTP(secret).provisioning_uri(
            name=user_email,
            issuer_name=issuer,
        )

    def generate_qr_png(self, otpauth_uri: str, size: int = 8) -> bytes:
        qr = qrcode.QRCode(version=1, box_size=size, border=4)
        qr.add_data(otpauth_uri)
        qr.make(fit=True)
        img = qr.make_image(fill_color="black", back_color="white")
        buf = io.BytesIO()
        img.save(buf, format="PNG")
        return buf.getvalue()

    def verify(self, encrypted_secret: bytes, code: str) -> bool:
        raw_secret = self._fernet.decrypt(encrypted_secret).decode("utf-8")
        # window=1: ±30초 허용 (clock skew)
        return pyotp.TOTP(raw_secret).verify(code, valid_window=1)

    def encrypt_secret(self, raw_secret: str) -> bytes:
        return self._fernet.encrypt(raw_secret.encode("utf-8"))
```

> **Python 한정**: `pyotp.TOTP(secret).verify(code, valid_window=1)` 가 직전/현재/다음 윈도우 (±30s) 검증. `valid_window=0` 면 정확히 현재만.

---

## 4. UseCase

### 4.1 Enroll

```python
def enroll_2fa(user_id: int, email: str) -> dict:
    existing = repo.find_by_user(user_id)
    if existing and existing.status == TwoFactorStatus.ENABLED:
        raise BusinessException(ResponseCode.CONFLICT, "2FA 가 이미 활성화되어 있습니다.")

    secret = totp_service.generate_secret()
    encrypted = totp_service.encrypt_secret(secret)

    repo.upsert(
        user_id=user_id,
        secret_encrypted=encrypted,
        status=TwoFactorStatus.PENDING,
    )

    uri = totp_service.build_otpauth_uri(email, "Shop", secret)
    qr_png = totp_service.generate_qr_png(uri)

    return {
        "otpAuthUri": uri,
        "qrDataUrl": f"data:image/png;base64,{base64.b64encode(qr_png).decode()}",
    }
```

### 4.2 Enable + recovery codes

```python
import secrets


def enable_2fa(user_id: int, code: str) -> list[str]:
    tf = repo.find_by_user(user_id)
    if tf is None or tf.status != TwoFactorStatus.PENDING:
        raise BusinessException(ResponseCode.CONFLICT, "PENDING 상태가 아닙니다.")

    if not totp_service.verify(tf.secret_encrypted, code):
        raise BusinessException(ResponseCode.UNAUTHORIZED, "TOTP 코드가 유효하지 않습니다.")

    # 10 recovery codes (raw 는 1회만 노출)
    raw_codes = [_random_recovery_code() for _ in range(10)]
    hashed = [hashlib.sha256(c.encode()).hexdigest() for c in raw_codes]

    repo.activate(user_id, hashed)
    return raw_codes


def _random_recovery_code() -> str:
    """1234-5678-90AB 형태."""
    hex_str = secrets.token_hex(6).upper()
    return f"{hex_str[:4]}-{hex_str[4:8]}-{hex_str[8:12]}"
```

### 4.3 Verify

```python
def verify_2fa(pending_token: str, code: str) -> dict:
    # pending_token = JWT (type=2FA_PENDING, 5분 TTL)
    claims = decode_token(pending_token)
    if claims.get("token_type") != "2FA_PENDING":
        raise BusinessException(ResponseCode.UNAUTHORIZED)

    user_id = claims["id"]
    tf = repo.find_by_user(user_id)
    if tf is None:
        raise BusinessException(ResponseCode.NOT_FOUND)

    # brute force lock 체크
    if tf.locked_until and tf.locked_until > datetime.now(timezone.utc):
        raise BusinessException(ResponseCode.FORBIDDEN, "2FA 가 잠겨 있습니다.")

    if not totp_service.verify(tf.secret_encrypted, code):
        tf.failed_attempts += 1
        if tf.failed_attempts >= 5:
            tf.locked_until = datetime.now(timezone.utc) + timedelta(minutes=15)
            tf.failed_attempts = 0
        repo.save(tf)
        raise BusinessException(ResponseCode.UNAUTHORIZED, "유효하지 않은 인증 코드")

    tf.failed_attempts = 0
    tf.locked_until = None
    repo.save(tf)

    # 정식 access + refresh 발급
    user = user_repo.find_by_id(user_id)
    return {
        "accessToken": create_access_token(user.email, user.id, user.role.name),
        "refreshToken": create_refresh_token(user.email),
    }
```

---

## 5. Controller

```python
@auth_2fa_bp.post("/enroll")
@jwt_required()
def enroll_endpoint():
    claims = get_jwt()
    result = enroll_2fa(claims["id"], claims["sub"])
    return ok("2FA 등록 — TOTP 코드 확인 필요", result)


@auth_2fa_bp.post("/enable")
@jwt_required()
def enable_endpoint():
    payload = EnableSchema().load(request.get_json())
    claims = get_jwt()
    codes = enable_2fa(claims["id"], payload["code"])
    return ok(
        "2FA 활성화 완료. 복구 코드 10개를 안전한 곳에 보관하세요.",
        {"recoveryCodes": codes},
    )


@auth_2fa_bp.post("/verify")
def verify_endpoint():
    payload = VerifySchema().load(request.get_json())
    tokens = verify_2fa(payload["pendingToken"], payload["code"])
    return ok("로그인 성공", tokens)
```

---

## 6. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/two-factor-auth#8 함정 모음|§8]] 의 10 함정 그대로)

### Python/Flask 한정

### 함정 11 — `pyotp.TOTP(secret).verify(code)` 의 `code` 가 str
6자리 `"012345"` — int 로 받으면 앞 0 잘림. **항상 `str`** + 정규식 검증.

### 함정 12 — `Fernet` 이 AES-GCM 아님
`cryptography.fernet.Fernet` = AES-128-CBC + HMAC. 더 강한 보안은 `AESGCM` 직접 사용:
```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
```

### 함정 13 — `secrets.token_hex` vs `random.choice`
recovery code 는 반드시 `secrets` (cryptographically secure). `random` 절대 X.

---

## 7. 운영 체크리스트

(Java 원본 §9 그대로)

---

## 8. 관련

- [[../../spring/java-spring/api-design/two-factor-auth|↗ Java 원본]]
- [[rbac-permissions]]
- [[../common/security-config]]
- [[api-design|↑ hub]]
