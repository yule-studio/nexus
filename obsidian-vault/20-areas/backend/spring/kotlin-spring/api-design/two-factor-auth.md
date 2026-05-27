---
title: "2FA / TOTP (Google Authenticator) — Kotlin Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T13:45:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - auth
  - 2fa
  - totp
---

# 2FA / TOTP (Google Authenticator) — Kotlin Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + buildList / scope fn + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/two-factor-auth|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가 / 1.1 로그인 흐름 / 1.2 비기능 / 3. DB 스키마

(Java 원본 §1, §3 와 동일)

---

## 2. 도메인

```kotlin
// domain/auth/UserTwoFactor.kt
class UserTwoFactor(
    val userId: UserId,
    private var _secretEncrypted: ByteArray,
    private var _status: Status,
    private val _recoveryCodes: MutableList<RecoveryCode>,
    private var _failedAttempts: Int,
    private var _lockedUntil: Instant?,
    private var _enabledAt: Instant?,
) {
    enum class Status { DISABLED, PENDING, ENABLED }

    val secretEncrypted: ByteArray get() = _secretEncrypted
    val status: Status get() = _status
    val recoveryCodes: List<RecoveryCode> get() = _recoveryCodes.toList()

    fun isEnabled(): Boolean = _status == Status.ENABLED

    fun verify(code: Int, verifier: TotpVerifier, now: Instant) {
        if (_lockedUntil?.let { now.isBefore(it) } == true) {
            throw BusinessException(
                ResponseCode.FORBIDDEN,
                "2FA 가 잠겨 있습니다. ${_lockedUntil} 이후 시도해 주세요.",
            )
        }
        if (_status != Status.ENABLED) {
            throw BusinessException(ResponseCode.BAD_REQUEST, "2FA 가 활성화되지 않았습니다.")
        }
        if (!verifier.verify(_secretEncrypted, code, now)) {
            _failedAttempts++
            if (_failedAttempts >= 5) {
                _lockedUntil = now.plus(Duration.ofMinutes(15))
                _failedAttempts = 0
            }
            throw BusinessException(ResponseCode.UNAUTHORIZED, "유효하지 않은 인증 코드")
        }
        _failedAttempts = 0
        _lockedUntil = null
    }

    fun consumeRecovery(rawCode: String, now: Instant): RecoveryCode {
        val hash = sha256Hex(rawCode)
        val idx = _recoveryCodes.indexOfFirst { it.tokenHash == hash && !it.used }
        if (idx < 0) {
            throw BusinessException(ResponseCode.UNAUTHORIZED, "유효하지 않은 복구 코드")
        }
        val updated = _recoveryCodes[idx].markUsed(now)
        _recoveryCodes[idx] = updated
        return updated
    }

    fun replaceSecret(encrypted: ByteArray) {
        _secretEncrypted = encrypted
        _status = Status.PENDING
    }

    fun activate(hashedCodes: List<RecoveryCode>, now: Instant) {
        _status = Status.ENABLED
        _enabledAt = now
        _recoveryCodes.clear()
        _recoveryCodes.addAll(hashedCodes)
    }
}

data class RecoveryCode(val tokenHash: String, val used: Boolean, val usedAt: Instant?) {
    fun markUsed(now: Instant): RecoveryCode = copy(used = true, usedAt = now)
}
```

> **Kotlin idiom 포인트**:
> - data class `RecoveryCode.copy(used = true, usedAt = now)` — immutable update.
> - `_lockedUntil?.let { now.isBefore(it) } == true` — null + condition 한 줄.

---

## 4. TOTP Service

```kotlin
implementation("com.warrenstrange:googleauth:1.5.0")
implementation("com.google.zxing:core:3.5.3")
implementation("com.google.zxing:javase:3.5.3")
```

```kotlin
@Component
class TotpService(
    private val secretEncryption: SecretEncryption,
) : TotpVerifier {

    companion object {
        private val GAUTH = GoogleAuthenticator(
            GoogleAuthenticatorConfig.GoogleAuthenticatorConfigBuilder()
                .setWindowSize(1)
                .setCodeDigits(6)
                .setTimeStepSizeInMillis(30_000)
                .build(),
        )
    }

    fun generateSecret(): String = GAUTH.createCredentials().key

    fun buildOtpAuthUri(userEmail: String, issuer: String, secret: String): String {
        val enc = { v: String -> URLEncoder.encode(v, StandardCharsets.UTF_8) }
        return "otpauth://totp/${enc(issuer)}:${enc(userEmail)}?secret=$secret&issuer=${enc(issuer)}"
    }

    fun generateQrPng(otpAuthUri: String, size: Int): ByteArray {
        val bitMatrix = QRCodeWriter().encode(otpAuthUri, BarcodeFormat.QR_CODE, size, size)
        return ByteArrayOutputStream().use { out ->
            MatrixToImageWriter.writeToStream(bitMatrix, "PNG", out)
            out.toByteArray()
        }
    }

    override fun verify(secretEncrypted: ByteArray, code: Int, now: Instant): Boolean {
        val rawSecret = secretEncryption.decrypt(secretEncrypted).toString(StandardCharsets.UTF_8)
        return GAUTH.authorize(rawSecret, code)
    }
}
```

> **Kotlin idiom**: local function `val enc = { v: String -> URLEncoder.encode(...) }` — DSL 처럼.

---

## 5. UseCase

### 5.1 Enroll

```kotlin
@Service
class Enroll2FAUseCase(
    private val repo: UserTwoFactorRepository,
    private val totp: TotpService,
    private val secretEnc: SecretEncryption,
    private val clock: Clock,
) {

    @Transactional
    fun handle(userId: UserId, email: String): EnrollResponse {
        val existing = repo.findByUserId(userId)
        if (existing?.status == UserTwoFactor.Status.ENABLED) {
            throw BusinessException(ResponseCode.CONFLICT, "2FA 가 이미 활성화되어 있습니다.")
        }

        val secret = totp.generateSecret()
        val encrypted = secretEnc.encrypt(secret.toByteArray(StandardCharsets.UTF_8))

        val tf = existing?.apply { replaceSecret(encrypted) }
            ?: UserTwoFactor(
                userId = userId, _secretEncrypted = encrypted,
                _status = UserTwoFactor.Status.PENDING,
                _recoveryCodes = mutableListOf(),
                _failedAttempts = 0, _lockedUntil = null, _enabledAt = null,
            )
        repo.save(tf)

        val uri = totp.buildOtpAuthUri(email, "Shop", secret)
        val qrPng = runCatching { totp.generateQrPng(uri, 200) }
            .getOrElse { throw BusinessException(ResponseCode.INTERNAL_SERVER_ERROR, "QR 생성 실패") }

        return EnrollResponse(
            otpAuthUri = uri,
            qrDataUrl = "data:image/png;base64,${Base64.getEncoder().encodeToString(qrPng)}",
        )
    }

    data class EnrollResponse(val otpAuthUri: String, val qrDataUrl: String)
}
```

### 5.2 Enable

```kotlin
@Service
class Enable2FAUseCase(
    private val repo: UserTwoFactorRepository,
    private val totp: TotpService,
    private val clock: Clock,
) {
    private val random = SecureRandom()

    @Transactional
    fun handle(userId: UserId, code: Int): List<String> {
        val tf = repo.findByUserId(userId)
            ?: throw BusinessException(ResponseCode.NOT_FOUND, "2FA 등록 정보가 없습니다.")
        if (tf.status != UserTwoFactor.Status.PENDING) {
            throw BusinessException(ResponseCode.CONFLICT, "이미 활성화된 2FA")
        }
        if (!totp.verify(tf.secretEncrypted, code, Instant.now(clock))) {
            throw BusinessException(ResponseCode.UNAUTHORIZED, "TOTP 코드가 유효하지 않습니다.")
        }

        // 10 개 recovery codes
        val rawCodes = List(10) { randomCode() }
        val hashedCodes = rawCodes.map { raw ->
            RecoveryCode(tokenHash = sha256Hex(raw), used = false, usedAt = null)
        }
        tf.activate(hashedCodes, Instant.now(clock))
        repo.save(tf)

        return rawCodes     // 클라에 한 번만 노출
    }

    private fun randomCode(): String {
        val bytes = ByteArray(6).also(random::nextBytes)
        val hex = HexFormat.of().formatHex(bytes).uppercase()
        return "${hex.substring(0, 4)}-${hex.substring(4, 8)}-${hex.substring(8, 12)}"
    }
}
```

> **Kotlin idiom**: `List(10) { randomCode() }` — `for (i in 0..9)` + add 의 한 expression.

### 5.4 Verify2FAUseCase

```kotlin
@Service
class Verify2FAUseCase(
    private val jwt: JwtTokenProvider,
    private val repo: UserTwoFactorRepository,
    private val users: UserRepository,
    private val totp: TotpService,
    private val rtService: RefreshTokenService,
    private val clock: Clock,
) {

    @Transactional
    fun handle(pendingToken: String, code: Int, device: String?, ip: String?): LoginResult {
        if (!jwt.is2FAPending(pendingToken)) {
            throw BusinessException(ResponseCode.UNAUTHORIZED, "유효하지 않은 2FA 토큰")
        }
        val userId = UserId(jwt.getUserId(pendingToken).toString())
        val tf = repo.findByUserId(userId)
            ?: throw BusinessException(ResponseCode.NOT_FOUND)
        val user = users.findById(userId)
            ?: throw BusinessException(ResponseCode.NOT_FOUND)

        tf.verify(code, totp, Instant.now(clock))
        repo.save(tf)

        val access = jwt.generateAccessToken(
            email = user.email.value,
            userId = user.id.value,
            role = user.role.name,
            sessionId = UUID.randomUUID().toString(),
        )
        val rt = rtService.issue(user.id, device, ip)
        return LoginResult(access, rt.raw, rt.expiresAt, null, false)
    }
}
```

---

## 6. Controller

```kotlin
@Tag(name = "2FA")
@RestController
@RequestMapping("/api/v1/auth/2fa")
class TwoFactorController(
    private val enroll: Enroll2FAUseCase,
    private val enable: Enable2FAUseCase,
    private val disable: Disable2FAUseCase,
    private val verify: Verify2FAUseCase,
    private val recovery: RecoveryUseCase,
) {

    @PostMapping("/enroll")
    fun enroll(auth: Authentication): ResponseEntity<CommonResponse<Enroll2FAUseCase.EnrollResponse>> {
        val user = auth.principal as User
        val res = enroll.handle(user.id, user.email.value)
        return CommonResponse.success(ResponseCode.OK, res, "2FA 등록 — TOTP 코드 확인 필요")
            .toResponseEntity()
    }

    @PostMapping("/enable")
    fun enable(
        @Valid @RequestBody req: Enable2FARequest,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Map<String, List<String>>>> {
        val user = auth.principal as User
        val codes = enable.handle(user.id, req.code)
        return CommonResponse.success(
            ResponseCode.OK,
            mapOf("recoveryCodes" to codes),
            "2FA 활성화 완료. 복구 코드 10개를 안전한 곳에 보관하세요.",
        ).toResponseEntity()
    }

    @SecurityRequirements()
    @PostMapping("/verify")
    fun verify(
        @Valid @RequestBody req: Verify2FARequest,
        http: HttpServletRequest,
    ): ResponseEntity<CommonResponse<TokenResponse>> {
        val result = verify.handle(req.pendingToken, req.code, http.getHeader("X-DEVICE-UUID"), http.clientIp())
        return CommonResponse.success(
            ResponseCode.OK,
            TokenResponse(result.accessToken, result.refreshToken, "Bearer", 900L),
            "로그인 성공",
        ).toResponseEntity()
    }
}

data class Enable2FARequest(@field:Min(0) @field:Max(999_999) val code: Int)
data class Verify2FARequest(
    @field:NotBlank val pendingToken: String,
    @field:Min(0) @field:Max(999_999) val code: Int,
)
```

---

## 7. SecretEncryption (AES-GCM)

```kotlin
@Component
class SecretEncryption(
    @Value("\${app.secret-encryption.key}") base64Key: String,
) {
    private val key: SecretKey = SecretKeySpec(Base64.getDecoder().decode(base64Key), "AES")
    private val random = SecureRandom()

    companion object {
        private const val NONCE_LEN = 12
        private const val TAG_LEN_BIT = 128
    }

    fun encrypt(plain: ByteArray): ByteArray = runCatching {
        val nonce = ByteArray(NONCE_LEN).also(random::nextBytes)
        val c = Cipher.getInstance("AES/GCM/NoPadding").apply {
            init(Cipher.ENCRYPT_MODE, key, GCMParameterSpec(TAG_LEN_BIT, nonce))
        }
        val ct = c.doFinal(plain)
        ByteBuffer.allocate(nonce.size + ct.size).apply { put(nonce); put(ct) }.array()
    }.getOrElse { throw IllegalStateException(it) }

    fun decrypt(envelope: ByteArray): ByteArray = runCatching {
        val nonce = envelope.copyOfRange(0, NONCE_LEN)
        val ct = envelope.copyOfRange(NONCE_LEN, envelope.size)
        Cipher.getInstance("AES/GCM/NoPadding").apply {
            init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(TAG_LEN_BIT, nonce))
        }.doFinal(ct)
    }.getOrElse { throw IllegalStateException(it) }
}
```

---

## 8. 함정 모음

(원본 [[../../java-spring/api-design/two-factor-auth#8 함정 모음|§8]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `ByteArray` equals
`ByteArray` 는 reference equality. data class 의 secret 비교 시 `contentEquals` 사용. (`UserTwoFactor` 는 일반 class 라 무관.)

### 함정 12 — `_lockedUntil?.let { ... } == true` 패턴
nullable Boolean 비교는 `?: false` 또는 `== true` — 명시적으로. `if (_lockedUntil != null && now.isBefore(_lockedUntil!!))` 보다 안전.

### 함정 13 — `List(N) { ... }` 의 lazy 인식
`List(10) { randomCode() }` 는 eager evaluation — 모든 10개 즉시 생성. `generateSequence` 와 혼동 X.

---

## 9. 운영 체크리스트

(Java 원본 §9 그대로 + Kotlin)

- [ ] `ByteArray contentEquals` 사용
- [ ] `@field:` validation target 명시

---

## 10. 관련

- [[../../java-spring/api-design/two-factor-auth|↗ Java 원본]]
- [[rbac-permissions]] — admin 2FA 필수
- [[../common/security-config]]
- [[api-design|↑ api-design hub]]
