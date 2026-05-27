---
title: "파일/이미지 업로드 (S3 presigned URL) — Kotlin Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T12:35:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - file-upload
  - s3
---

# 파일/이미지 업로드 (S3 presigned URL) — Kotlin Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin Domain entity 패턴 + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/file-upload-s3|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 왜 presigned URL / 2. 무엇을 만드는가 / 7. CORS

(Java 원본 [[../../java-spring/api-design/file-upload-s3#1 왜 presigned URL|§1]], §2, §7 와 동일 — 흐름 / API spec / S3 CORS JSON 은 언어 무관)

---

## 3. 도메인

```kotlin
// domain/file/FileMeta.kt — JPA Entity 와 분리된 도메인 클래스
class FileMeta private constructor(
    val id: FileId,
    val ownerId: UserId,
    val kind: Kind,
    val s3Bucket: String,
    val s3Key: String,
    val contentType: String,
    val sizeBytes: Long,
    val originalFilename: String?,
    private var _status: Status,
    val createdAt: Instant,
    private var _completedAt: Instant?,
) {
    enum class Kind { PROFILE_IMAGE, PRODUCT_IMAGE, REVIEW_IMAGE, ATTACHMENT, VIDEO }
    enum class Status { PENDING, COMPLETED, FAILED, DELETED }

    val status: Status get() = _status
    val completedAt: Instant? get() = _completedAt

    fun markCompleted(verifiedSize: Long, now: Instant) {
        require(_status == Status.PENDING) { "이미 완료된 파일" }
        require(verifiedSize == sizeBytes) {
            "선언 크기와 실제 크기가 다릅니다: $sizeBytes vs $verifiedSize"
        }
        _status = Status.COMPLETED
        _completedAt = now
    }

    fun markDeleted() { _status = Status.DELETED }

    companion object {
        fun initiate(
            id: FileId, ownerId: UserId, kind: Kind,
            bucket: String, key: String, contentType: String,
            sizeBytes: Long, originalName: String?, now: Instant,
        ): FileMeta = FileMeta(
            id, ownerId, kind, bucket, key, contentType, sizeBytes, originalName,
            Status.PENDING, now, null,
        )
    }
}
```

> **Kotlin idiom 포인트**:
> - mutable state 가 있는 도메인은 `data class` 가 아니라 일반 `class` + `private constructor` + `companion.factory()`.
> - `var _status` + `val status: Status get() = _status` — public read-only / private write.
> - `require(...)` — `IllegalArgumentException` 강제. 단, 도메인 예외는 `BusinessException` 으로 wrap 권장 (try/check).

DB:
```sql
CREATE TABLE files (
    id              CHAR(26) PRIMARY KEY,
    owner_id        CHAR(26) NOT NULL REFERENCES users(id),
    kind            VARCHAR(30) NOT NULL,
    s3_bucket       VARCHAR(100) NOT NULL,
    s3_key          VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL CHECK (size_bytes > 0),
    original_name   VARCHAR(255),
    status          VARCHAR(20) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE UNIQUE INDEX ux_files_s3 ON files (s3_bucket, s3_key);
CREATE INDEX ix_files_owner_kind ON files (owner_id, kind, status);
```

---

## 4. S3 Service

```kotlin
implementation("software.amazon.awssdk:s3:2.25.50")
implementation("software.amazon.awssdk:s3-transfer-manager:2.25.50")
```

```kotlin
@Component
class S3FileService(
    private val s3: S3Client,
    private val presigner: S3Presigner,
    @Value("\${app.s3.bucket}") private val bucket: String,
    @Value("\${app.s3.upload-url-ttl-minutes:5}") private val uploadTtlMinutes: Long,
    @Value("\${app.s3.download-url-ttl-minutes:15}") private val downloadTtlMinutes: Long,
    @Value("\${app.s3.cdn-base-url:}") private val cdnBaseUrl: String,
) {

    fun buildUploadUrl(key: String, contentType: String, sizeBytes: Long): PresignedUpload {
        val putRequest = PutObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .contentType(contentType)
            .contentLength(sizeBytes)
            .build()
        val presigned = presigner.presignPutObject {
            it.signatureDuration(Duration.ofMinutes(uploadTtlMinutes))
                .putObjectRequest(putRequest)
        }
        return PresignedUpload(
            uploadUrl = presigned.url().toString(),
            bucket = bucket,
            key = key,
            expiresAt = Instant.now().plus(Duration.ofMinutes(uploadTtlMinutes)),
        )
    }

    fun buildDownloadUrl(key: String): String {
        // CDN URL 이 있으면 우선 (캐시 + 비용 ↓)
        if (cdnBaseUrl.isNotBlank()) return "$cdnBaseUrl/$key"

        val getRequest = GetObjectRequest.builder().bucket(bucket).key(key).build()
        return presigner.presignGetObject {
            it.signatureDuration(Duration.ofMinutes(downloadTtlMinutes))
                .getObjectRequest(getRequest)
        }.url().toString()
    }

    /** 업로드 완료 검증 — HEAD 로 실제 size / contentType 확인 */
    fun verify(key: String): HeadObjectResponse = runCatching {
        s3.headObject(HeadObjectRequest.builder().bucket(bucket).key(key).build())
    }.getOrElse { ex ->
        if (ex is NoSuchKeyException) {
            throw BusinessException(ResponseCode.NOT_FOUND, "S3 에 파일이 없습니다.")
        }
        throw ex
    }

    fun deleteAsync(key: String) {
        s3.deleteObject(DeleteObjectRequest.builder().bucket(bucket).key(key).build())
    }
}

data class PresignedUpload(
    val uploadUrl: String,
    val bucket: String,
    val key: String,
    val expiresAt: Instant,
)
```

```yaml
app:
  s3:
    bucket: ${S3_BUCKET}
    region: ap-northeast-2
    upload-url-ttl-minutes: 5
    download-url-ttl-minutes: 15
    cdn-base-url: ${CDN_BASE_URL:}
```

---

## 5. UseCase

```kotlin
@Service
class IssueUploadUrlUseCase(
    private val files: FileMetaRepository,
    private val s3: S3FileService,
    private val ids: IdGenerator,
    private val clock: Clock,
) {
    companion object {
        private const val MAX_IMAGE_BYTES = 10L * 1024 * 1024          // 10 MB
        private const val MAX_VIDEO_BYTES = 500L * 1024 * 1024         // 500 MB
        private val ALLOWED_IMAGE = setOf("image/jpeg", "image/png", "image/webp")
        private val ALLOWED_VIDEO = setOf("video/mp4", "video/quicktime")
        private val ALLOWED_EXTS = setOf(".jpg", ".jpeg", ".png", ".webp", ".mp4", ".mov")
    }

    @Transactional
    fun handle(
        ownerId: UserId,
        kind: FileMeta.Kind,
        contentType: String,
        sizeBytes: Long,
        originalFilename: String?,
    ): Result {
        validate(kind, contentType, sizeBytes)

        val fileId = FileId(ids.next())
        val key = buildKey(kind, ownerId, fileId, originalFilename)

        val presigned = s3.buildUploadUrl(key, contentType, sizeBytes)
        val meta = FileMeta.initiate(
            id = fileId, ownerId = ownerId, kind = kind,
            bucket = presigned.bucket, key = presigned.key,
            contentType = contentType, sizeBytes = sizeBytes,
            originalName = originalFilename, now = Instant.now(clock),
        )
        files.save(meta)

        return Result(fileId, presigned.uploadUrl, presigned.expiresAt)
    }

    private fun validate(kind: FileMeta.Kind, contentType: String, sizeBytes: Long) {
        if (sizeBytes <= 0) throw BusinessException(ResponseCode.BAD_REQUEST, "유효하지 않은 파일 크기")

        val isImage = kind in setOf(
            FileMeta.Kind.PROFILE_IMAGE,
            FileMeta.Kind.PRODUCT_IMAGE,
            FileMeta.Kind.REVIEW_IMAGE,
        )

        when {
            isImage -> {
                if (contentType !in ALLOWED_IMAGE)
                    throw BusinessException(ResponseCode.BAD_REQUEST, "허용되지 않는 이미지 형식")
                if (sizeBytes > MAX_IMAGE_BYTES)
                    throw BusinessException(ResponseCode.BAD_REQUEST, "이미지는 최대 10MB")
            }
            kind == FileMeta.Kind.VIDEO -> {
                if (contentType !in ALLOWED_VIDEO)
                    throw BusinessException(ResponseCode.BAD_REQUEST, "허용되지 않는 동영상 형식")
                if (sizeBytes > MAX_VIDEO_BYTES)
                    throw BusinessException(ResponseCode.BAD_REQUEST, "동영상은 최대 500MB")
            }
        }
    }

    private fun buildKey(kind: FileMeta.Kind, ownerId: UserId, fileId: FileId, original: String?): String {
        val now = LocalDate.now()
        val ext = original?.substringAfterLast('.', "")?.lowercase()?.let { ".$it" }.orEmpty()
        return "%s/%d/%02d/%s/%s%s".format(
            kind.name.lowercase(),
            now.year, now.monthValue,
            ownerId.value,
            fileId.value,
            sanitizeExt(ext),
        )
    }

    private fun sanitizeExt(ext: String): String =
        if (ext.lowercase() in ALLOWED_EXTS) ext.lowercase() else ""

    data class Result(val fileId: FileId, val uploadUrl: String, val expiresAt: Instant)
}
```

```kotlin
@Service
class CompleteUploadUseCase(
    private val files: FileMetaRepository,
    private val s3: S3FileService,
    private val events: ApplicationEventPublisher,
    private val clock: Clock,
) {

    @Transactional
    fun handle(fileId: FileId, currentUser: UserId) {
        val meta = files.findById(fileId)
            ?: throw BusinessException(ResponseCode.NOT_FOUND, "파일")
        if (meta.ownerId != currentUser) throw BusinessException(ResponseCode.FORBIDDEN)

        // S3 에 실제 업로드 확인
        val head = s3.verify(meta.s3Key)
        meta.markCompleted(head.contentLength(), Instant.now(clock))
        files.save(meta)

        // 후처리 트리거
        events.publishEvent(
            FileUploaded(fileId, meta.kind, meta.s3Bucket, meta.s3Key, Instant.now(clock)),
        )
    }
}

data class FileUploaded(
    val fileId: FileId,
    val kind: FileMeta.Kind,
    val bucket: String,
    val key: String,
    val occurredAt: Instant,
) : DomainEvent
```

> **Kotlin idiom**: `when { ... }` (with no subject) — Java if/else chain 보다 가독성 좋음.

---

## 6. Controller

```kotlin
@Tag(name = "파일 업로드")
@RestController
@RequestMapping("/api/v1/files")
@PreAuthorize("isAuthenticated()")
class FileController(
    private val issue: IssueUploadUrlUseCase,
    private val complete: CompleteUploadUseCase,
    private val getFile: GetFileUseCase,
) {

    @PostMapping("/presigned-url")
    fun presigned(
        @Valid @RequestBody req: PresignedRequest,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<IssueUploadUrlUseCase.Result>> {
        val result = issue.handle(
            UserId(auth.name),
            req.kind, req.contentType, req.sizeBytes, req.originalFilename,
        )
        return CommonResponse.success(ResponseCode.OK, result, "URL 발급").toResponseEntity()
    }

    @PostMapping("/{id}/complete")
    fun complete(
        @PathVariable id: String,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Unit>> {
        complete.handle(FileId(id), UserId(auth.name))
        return CommonResponse.success<Unit>(ResponseCode.OK, "업로드 완료").toResponseEntity()
    }
}

data class PresignedRequest(
    @field:NotNull val kind: FileMeta.Kind,
    @field:NotBlank val contentType: String,
    @field:Positive val sizeBytes: Long,
    val originalFilename: String?,
)
```

---

## 8. 후처리 (Async Listener — Spring 내부 식)

```kotlin
@Component
class FileUploadedListener(
    private val imageProcessor: ImageProcessor,
    private val s3: S3FileService,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun on(event: FileUploaded) {
        if (event.kind != FileMeta.Kind.PRODUCT_IMAGE) return
        runCatching {
            val input = s3.downloadStream(event.bucket, event.key)
            listOf(256, 512, 1024).forEach { size ->
                val resized = imageProcessor.resize(input, size)
                val thumbKey = event.key.replaceFirst(Regex("^[^/]+/"), "thumbnails/$size/")
                s3.upload(event.bucket, thumbKey, resized, "image/webp")
            }
        }.onFailure { ex ->
            log.error("thumbnail failed for {}", event.key, ex)
        }
    }
}
```

→ Spring 내부 async 는 인스턴스 부하 ↑. **Lambda 가 권장** (분리된 컴퓨트).

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/file-upload-s3#9 함정 모음|§9]] 의 11가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 12 — `@Value("\${...}")` $ escape
Kotlin string template 과 충돌 — `@Value("${...}")` 는 Kotlin 이 evaluate 시도. **`\$` escape** 필수.

### 함정 13 — Domain entity 를 data class
`FileMeta` 가 `var status` mutable 인 경우 `data class` 의 자동 equals 가 status 까지 비교 → repository identity 와 어긋남. **일반 class + factory** 사용.

### 함정 14 — String?.substringAfterLast 의 fallback
`"foo".substringAfterLast('.', "")` 는 separator 없을 때 ""반환 (Kotlin stdlib). Java 의 `lastIndexOf < 0` 분기 불필요.

---

## 10. 운영 체크리스트

- [ ] bucket private + presigned 만
- [ ] CloudFront / CDN 으로 download
- [ ] S3 CORS 설정 (도메인 화이트리스트)
- [ ] lifecycle policy — uploads/ 의 1일 이내 PENDING 파일 자동 삭제
- [ ] virus scan (큰 첨부 시)
- [ ] thumbnail Lambda 또는 별도 워커
- [ ] file_size / file_count 모니터 (사용자별)
- [ ] presigned URL 시간 짧게
- [ ] DELETE 시 soft + S3 lifecycle 결합
- [ ] S3 access log 활성
- [ ] `@Value` 의 `\$` escape 확인

---

## 11. 관련

- [[../../java-spring/api-design/file-upload-s3|↗ Java 원본]]
- [[review]] — review image 의 file 참조
- [[../common/security-config]] — CORS
- [[api-design|↑ api-design hub]]
