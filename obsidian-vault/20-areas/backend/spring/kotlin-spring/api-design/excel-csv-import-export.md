---
title: "Excel / CSV Import·Export (스트리밍 + 검증) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T12:15:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - excel
  - csv
---

# Excel / CSV Import·Export (스트리밍 + 검증) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin use {} / sequences / 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/excel-csv-import-export|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[file-upload-s3]].

---

## 1. 무엇을 만드는가 / 1.1 용량별 전략

(Java 원본 [[../../java-spring/api-design/excel-csv-import-export#1 무엇을 만드는가|§1]] 와 동일)

---

## 2. 의존성

```kotlin
implementation("org.apache.poi:poi-ooxml:5.2.5")             // Excel (xlsx)
implementation("com.opencsv:opencsv:5.9")                    // CSV
```

---

## 3. CSV Export — 스트리밍 (작은~중간)

```kotlin
@RestController
@RequestMapping("/api/v1/exports")
@PreAuthorize("hasRole('ADMIN')")
class CsvExportController(
    private val orders: OrderQueryRepository,
) {

    @GetMapping("/orders.csv")
    fun orders(
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) from: LocalDate,
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) to: LocalDate,
    ): ResponseEntity<StreamingResponseBody> {
        val body = StreamingResponseBody { output ->
            OutputStreamWriter(output, StandardCharsets.UTF_8).use { writer ->
                CSVWriter(writer).use { csv ->
                    // UTF-8 BOM (엑셀 한글 깨짐 방지)
                    writer.write('﻿'.code)
                    csv.writeNext(arrayOf("주문번호", "구매자", "금액", "상태", "주문일시"))

                    orders.streamBetween(from, to) { row ->
                        csv.writeNext(
                            arrayOf(
                                row.orderNumber,
                                row.buyerName,
                                row.amountKrw.toString(),
                                row.status,
                                row.createdAt.toString(),
                            ),
                        )
                    }
                }
            }
        }

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("text/csv; charset=UTF-8"))
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"orders.csv\"")
            .body(body)
    }
}
```

> **Kotlin idiom 포인트**:
> - `use { }` (extension on `Closeable`) — try-with-resources Kotlin 판. 중첩하면 try-with-resources `;` 분리 대신 nested lambda.
> - `arrayOf(...)` — `new String[]{ ... }`.

### 3.1 Repository — Cursor stream

```kotlin
interface OrderQueryRepository {
    fun streamBetween(from: LocalDate, to: LocalDate, consumer: (OrderExportRow) -> Unit)
}

@Repository
class JpaOrderQueryRepository(
    private val em: EntityManager,
) : OrderQueryRepository {

    @Transactional(readOnly = true)
    override fun streamBetween(from: LocalDate, to: LocalDate, consumer: (OrderExportRow) -> Unit) {
        em.createQuery(
            "from OrderJpaEntity o where o.createdAt between :from and :to",
            OrderJpaEntity::class.java,
        )
            .setParameter("from", from.atStartOfDay().toInstant(ZoneOffset.UTC))
            .setParameter("to", to.atStartOfDay().plusDays(1).toInstant(ZoneOffset.UTC))
            .setHint(QueryHints.HINT_FETCH_SIZE, 1000)
            .resultStream
            .use { stream ->
                stream.forEach { consumer(toRow(it)) }
            }
    }
}
```

> **Kotlin idiom**: `(OrderExportRow) -> Unit` 함수 타입 — Java `Consumer<OrderExportRow>` 와 동등하지만 lambda 직접 받음.

---

## 4. Excel Export — POI SXSSF (Streaming)

```kotlin
@GetMapping("/users.xlsx")
fun usersXlsx(response: HttpServletResponse) {
    response.contentType = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    response.setHeader(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"users.xlsx\"")

    SXSSFWorkbook(100).use { workbook ->       // 메모리 최근 100 행만
        val sheet = workbook.createSheet("Users")

        // 헤더
        sheet.createRow(0).apply {
            listOf("ID", "이메일", "이름", "가입일").forEachIndexed { i, h ->
                createCell(i).setCellValue(h)
            }
        }

        // 데이터 streaming
        var rowIdx = 1
        users.streamAll { u ->
            sheet.createRow(rowIdx++).apply {
                createCell(0).setCellValue(u.id)
                createCell(1).setCellValue(u.email)
                createCell(2).setCellValue(u.name)
                createCell(3).setCellValue(u.createdAt.toString())
            }
        }

        workbook.write(response.outputStream)
        workbook.dispose()                       // 임시 파일 정리
    }
}
```

> **함정**: `XSSFWorkbook` (non-streaming) 으로 100만 행 = OOM 확정. **무조건 SXSSF**.

---

## 5. Excel Export — Async Job (큰 데이터)

100,000+ 행은 응답 timeout 위험. **job 모델**.

```sql
CREATE TABLE export_jobs (
    id            CHAR(26) PRIMARY KEY,
    requested_by  CHAR(26) NOT NULL REFERENCES users(id),
    kind          VARCHAR(50) NOT NULL,
    params        JSONB NOT NULL,
    status        VARCHAR(20) NOT NULL,
    s3_key        VARCHAR(500),
    row_count     BIGINT,
    error         TEXT,
    requested_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at    TIMESTAMPTZ,
    completed_at  TIMESTAMPTZ
);
```

```kotlin
@Service
class ExportJobService(
    private val jobs: ExportJobRepository,
    private val ids: IdGenerator,
    private val clock: Clock,
    private val executor: TaskExecutor,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Transactional
    fun create(requestedBy: UserId, kind: String, params: Map<String, Any?>): ExportJobId {
        val job = ExportJob(
            id = ExportJobId(ids.next()),
            requestedBy = requestedBy,
            kind = kind,
            params = params,
            status = "PENDING",
            s3Key = null, rowCount = null, error = null,
            requestedAt = Instant.now(clock),
            startedAt = null, completedAt = null,
        )
        jobs.save(job)

        executor.execute { runJob(job.id) }
        return job.id
    }

    @Async
    fun runJob(jobId: ExportJobId) {
        val job = jobs.findById(jobId) ?: return
        job.markRunning(Instant.now())
        jobs.save(job)

        runCatching {
            val s3Key = "exports/${jobId.value}.xlsx"
            val rowCount = writeExcelToS3(job.kind, job.params, s3Key)
            job.markSucceeded(s3Key, rowCount, Instant.now())
        }.onFailure { ex ->
            job.markFailed(ex.message ?: ex.javaClass.simpleName, Instant.now())
            log.error("export job {} failed", jobId, ex)
        }
        jobs.save(job)
    }
}
```

---

## 6. CSV Import — 한 줄씩 검증

```kotlin
@RestController
@RequestMapping("/api/v1/imports")
@PreAuthorize("hasAnyRole('SELLER', 'ADMIN')")
class CsvImportController(
    private val service: ImportJobService,
) {

    /**
     * 클라가 [[file-upload-s3]] 로 먼저 S3 업로드 → fileId 전달.
     * 본 endpoint 는 job 만 생성.
     */
    @PostMapping("/products")
    fun importProducts(
        @Valid @RequestBody req: ImportRequest,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Map<String, String>>> {
        val jobId = service.create(UserId(auth.name), "PRODUCT", req.fileId)
        return ResponseEntity.accepted().body(
            CommonResponse.success(
                ResponseCode.OK,
                mapOf("jobId" to jobId.value),
                "Import 시작. /imports/{jobId} 로 진행 확인.",
            ),
        )
    }
}

data class ImportRequest(@field:NotBlank val fileId: String)
```

```kotlin
@Service
class ProductImportJobRunner(
    private val s3: S3FileService,
    private val productImport: ProductImportService,
    private val jobs: ImportJobRepository,
    private val errors: ImportJobErrorRepository,
    private val ids: IdGenerator,
) {

    @Async
    fun run(jobId: ImportJobId) {
        val job = jobs.findById(jobId) ?: return
        runCatching {
            s3.downloadStream(job.bucket, job.s3Key).use { input ->
                InputStreamReader(input, StandardCharsets.UTF_8).use { reader ->
                    CSVReader(reader).use { csvReader ->
                        csvReader.readNext()                  // header skip

                        var processed = 0
                        var succeeded = 0
                        var failed = 0

                        generateSequence { csvReader.readNext() }.forEach { row ->
                            processed++
                            runCatching {
                                productImport.importRow(job.requestedBy, row)
                                succeeded++
                            }.onFailure { ex ->
                                failed++
                                errors.save(
                                    ImportError(
                                        id = ids.next(),
                                        jobId = jobId,
                                        rowNumber = processed,
                                        message = ex.message ?: ex.javaClass.simpleName,
                                        rawRow = row.joinToString(","),
                                        createdAt = Instant.now(),
                                    ),
                                )
                            }
                            // 진행률 — 100건마다 update
                            if (processed % 100 == 0) {
                                job.updateProgress(processed, succeeded, failed)
                                jobs.save(job)
                            }
                        }

                        job.markCompleted(processed, succeeded, failed, Instant.now())
                    }
                }
            }
        }.onFailure { ex ->
            job.markFailed(ex.message ?: ex.javaClass.simpleName, Instant.now())
        }
        jobs.save(job)
    }
}
```

> **Kotlin idiom 포인트**:
> - `generateSequence { csvReader.readNext() }` — Java `while ((row = readNext()) != null)` 패턴이 Kotlin 의 lazy sequence 로 (null 반환 시 자동 종료).
> - 중첩 `use { }` — 3 단 try-with-resources.
> - 행 단위 `runCatching` — 한 행 실패가 전체 import 를 깨뜨리지 않음.

→ 행 단위 검증 + 오류 row 별도 테이블 저장. UI 에서 "Import 결과: 1000건 처리, 8건 실패 — 상세보기" 가능.

---

## 7. Import 결과 응답 / 8. 보안 / 검증

(Java 원본 [[../../java-spring/api-design/excel-csv-import-export#7 Import 결과 응답|§7-§8]] 동일)

```kotlin
private fun String?.safeCellValue(): String? {
    if (this.isNullOrEmpty()) return this
    val first = this[0]
    return if (first in setOf('=', '+', '-', '@')) "'$this" else this
}
```

> **Kotlin idiom**: extension `String?.safeCellValue()` — Java static helper 가 receiver method 로.

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/excel-csv-import-export#9 함정 모음|§9]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `use { }` 와 lambda return
`use { ... }` 가 inline 이므로 `return` 은 non-local. 의도적으로 외부 함수 종료가 아니면 `return@use` 또는 explicit type.

### 함정 12 — `generateSequence` 의 thread-safety
sequence 는 lazy 평가라 worker thread 와 다른 thread 에서 소비하면 underlying CSVReader 의 thread-safety 가 깨질 수 있음. 한 thread 안에서 소비.

### 함정 13 — `Stream` 과 `Sequence` 혼용
JPA 의 `resultStream` 은 Java `Stream`, Kotlin 의 `Sequence` 와 다름. `.use { it.forEach(...) }` 로 직접 처리 권장. asSequence() 변환 시 종료 시점 모호.

---

## 10. 운영 체크리스트

- [ ] SXSSF 100 window
- [ ] UTF-8 BOM
- [ ] CSV injection 방어 (extension `safeCellValue`)
- [ ] `use { }` 로 stream try-with-resources
- [ ] Import 행 단위 트랜잭션 (runCatching)
- [ ] async job 모델 + S3 + presigned download
- [ ] file size + row count 한계
- [ ] job 진행률 폴링 + 결과 push
- [ ] error 결과 별도 테이블 + UI

---

## 11. 관련

- [[file-upload-s3]] — Import 의 업로드
- [[../../java-spring/api-design/excel-csv-import-export|↗ Java 원본]]
- [[../common/response-envelope]]
- [[api-design|↑ api-design hub]]
