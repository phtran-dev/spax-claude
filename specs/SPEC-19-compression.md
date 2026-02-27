# SPEC-19: DICOM Compression Module

## Mục tiêu
Implement DICOM compression: manual trigger per-study + background worker. Nén file tại chỗ (cùng path), xóa file gốc, update DB.

## Dependencies
- SPEC-02 (schema — compression_task table)
- SPEC-05 (StorageService, VolumeManager — đọc/ghi/xóa file)
- SPEC-13 (admin pattern)

## Files cần tạo

### 1. `src/main/java/com/spax/compression/CompressionType.java`

```java
public enum CompressionType {
    JPEG_LS_LOSSLESS  ("1.2.840.10008.1.2.4.80"),  // lossless, compress ~40-60%
    JPEG_2000_LOSSLESS("1.2.840.10008.1.2.4.90"),  // lossless, compress ~40-60%
    JPEG_BASELINE     ("1.2.840.10008.1.2.4.50"),  // lossy, compress ~70-85%
    JPEG_2000_LOSSY   ("1.2.840.10008.1.2.4.91");  // lossy, compress ~90%

    private final String transferSyntaxUid;

    CompressionType(String transferSyntaxUid) {
        this.transferSyntaxUid = transferSyntaxUid;
    }

    public String getTransferSyntaxUid() { return transferSyntaxUid; }

    public static CompressionType fromTransferSyntax(String tsUid) {
        for (CompressionType ct : values()) {
            if (ct.transferSyntaxUid.equals(tsUid)) return ct;
        }
        return null;
    }
}
```

### 2. `src/main/java/com/spax/compression/DicomCompressor.java`

Wrapper quanh dcm4che `Transcoder` API:

```java
@Component
public class DicomCompressor {

    /**
     * Compress a DICOM file to target transfer syntax.
     *
     * @param inputStream  raw DICOM bytes (caller must close)
     * @param type         target compression type
     * @return compressed DICOM bytes
     * @throws IOException if transcoding fails
     */
    public byte[] compress(InputStream inputStream, CompressionType type) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        // dcm4che Transcoder does: read DICOM → transcode pixels → write DICOM
        Transcoder transcoder = new Transcoder(inputStream);
        transcoder.setDestinationTransferSyntax(type.getTransferSyntaxUid());

        transcoder.transcode((t, dataset) -> {
            DicomOutputStream dos = new DicomOutputStream(output, type.getTransferSyntaxUid());
            dos.writeDataset(dataset.createFileMetaInformation(type.getTransferSyntaxUid()), dataset);
            dos.close();
        });

        return output.toByteArray();
    }

    /**
     * Check if a DICOM file is already in the target transfer syntax.
     * Used for idempotency check before compression.
     *
     * @param currentTransferSyntax  from instance.transfer_syntax_uid in DB
     * @param targetType             desired compression type
     * @return true if already compressed with target syntax (skip)
     */
    public boolean isAlreadyCompressed(String currentTransferSyntax, CompressionType targetType) {
        return targetType.getTransferSyntaxUid().equals(currentTransferSyntax);
    }
}
```

**Note về dcm4che Transcoder**:
- `dcm4che-imageio` dependency cần native codecs (`dcm4che-imageio-opencv`) để xử lý JPEG-LS và JPEG 2000
- `Transcoder` là class chính trong `org.dcm4che3.imageio.codec`
- Native libs bundled trong `dcm4che-imageio-opencv` JAR (Linux/macOS/Windows)
- Test với uncompressed DICOM (Explicit Little Endian `1.2.840.10008.1.2.1`) trước

### 3. `src/main/java/com/spax/compression/CompressionTask.java`

JPA entity:

```java
@Entity
@Table(name = "compression_task")
public class CompressionTask {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String studyInstanceUid;
    private String compressionType;   // enum name: JPEG_LS_LOSSLESS, etc.
    private int totalFiles;
    private int processedFiles;
    private int failedFiles;
    private String status;           // PENDING, IN_PROGRESS, COMPLETED, FAILED
    private String triggeredBy;      // null = auto (lifecycle), else username
    private Integer ruleId;          // FK to public.lifecycle_rule.id if auto
    private String errorSummary;
    private Instant createdAt;
    private Instant completedAt;
    // getters/setters or Lombok
}
```

### 4. `src/main/java/com/spax/compression/CompressionService.java`

Business logic — create task, async worker:

```java
@Service
public class CompressionService {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private DicomCompressor dicomCompressor;
    @Autowired private VolumeManager volumeManager;

    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    /**
     * Create a compression task and start processing it asynchronously.
     *
     * @param studyUid       Study to compress
     * @param type           Target compression type
     * @param ruleId         null if manual, lifecycle rule ID if auto
     * @param triggeredBy    username if manual, null if auto
     * @return task ID
     */
    public long createTask(String studyUid, CompressionType type, Integer ruleId, String triggeredBy) {
        // Count total files
        int totalFiles = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM instance WHERE study_instance_uid = ?",
            Integer.class, studyUid
        );

        if (totalFiles == 0) {
            throw new IllegalArgumentException("Study not found or has no instances: " + studyUid);
        }

        // Create task record
        long taskId = jdbcTemplate.queryForObject("""
            INSERT INTO compression_task
              (study_instance_uid, compression_type, total_files, status, triggered_by, rule_id)
            VALUES (?, ?, ?, 'PENDING', ?, ?)
            RETURNING id
            """,
            Long.class,
            studyUid, type.name(), totalFiles, triggeredBy, ruleId
        );

        // Submit async
        executor.submit(() -> processTask(taskId, studyUid, type));

        return taskId;
    }

    private void processTask(long taskId, String studyUid, CompressionType type) {
        // Mark IN_PROGRESS
        jdbcTemplate.update(
            "UPDATE compression_task SET status = 'IN_PROGRESS' WHERE id = ?", taskId
        );

        // Get all instances of study
        List<Map<String, Object>> instances = jdbcTemplate.queryForList("""
            SELECT id, volume_id, storage_path, transfer_syntax_uid
            FROM instance
            WHERE study_instance_uid = ?
            ORDER BY instance_number
            """, studyUid
        );

        int processed = 0;
        int failed = 0;
        List<String> errors = new ArrayList<>();

        for (Map<String, Object> inst : instances) {
            try {
                processInstance(inst, type, taskId);
                processed++;

                // Update progress every 10 files
                if (processed % 10 == 0) {
                    jdbcTemplate.update(
                        "UPDATE compression_task SET processed_files = ? WHERE id = ?",
                        processed, taskId
                    );
                }
            } catch (Exception e) {
                failed++;
                errors.add("Instance " + inst.get("id") + ": " + e.getMessage());
                log.warn("Failed to compress instance {}: {}", inst.get("id"), e.getMessage());
            }
        }

        // Recalculate sizes
        recalculateSizes(studyUid);

        // Final status
        String status = failed == 0 ? "COMPLETED" : (processed == 0 ? "FAILED" : "COMPLETED");
        jdbcTemplate.update("""
            UPDATE compression_task SET
                status = ?,
                processed_files = ?,
                failed_files = ?,
                error_summary = ?,
                completed_at = now()
            WHERE id = ?
            """,
            status, processed, failed,
            errors.isEmpty() ? null : String.join("\n", errors.subList(0, Math.min(10, errors.size()))),
            taskId
        );
    }

    private void processInstance(Map<String, Object> inst, CompressionType type, long taskId) throws IOException {
        int volumeId = (int) inst.get("volume_id");
        String storagePath = (String) inst.get("storage_path");
        String currentTsUid = (String) inst.get("transfer_syntax_uid");

        // Idempotency: skip if already correct transfer syntax
        if (dicomCompressor.isAlreadyCompressed(currentTsUid, type)) {
            log.debug("Instance already has target transfer syntax, skipping");
            return;
        }

        StorageProvider provider = volumeManager.getProvider(volumeId);

        // 1. Read original file
        byte[] compressed;
        try (InputStream in = provider.read(storagePath)) {
            // 2. Compress
            compressed = dicomCompressor.compress(in, type);
        }

        // 3. Delete original
        provider.delete(storagePath);

        // 4. Write compressed (same path)
        try (InputStream in = new ByteArrayInputStream(compressed)) {
            provider.write(storagePath, in, compressed.length);
        }

        // 5. Update instance in DB
        jdbcTemplate.update("""
            UPDATE instance SET
                transfer_syntax_uid = ?,
                file_size = ?
            WHERE id = ?
            """,
            type.getTransferSyntaxUid(), (long) compressed.length, inst.get("id")
        );
    }

    private void recalculateSizes(String studyUid) {
        // Update series sizes
        jdbcTemplate.update("""
            UPDATE series SET series_size = (
                SELECT COALESCE(SUM(file_size), 0) FROM instance
                WHERE series_instance_uid = series.series_instance_uid
                AND study_instance_uid = ?
            )
            WHERE study_instance_uid = ?
            """, studyUid, studyUid);

        // Update study size
        jdbcTemplate.update("""
            UPDATE study SET study_size = (
                SELECT COALESCE(SUM(file_size), 0) FROM instance
                WHERE study_instance_uid = ?
            )
            WHERE study_instance_uid = ?
            """, studyUid, studyUid);

        // Update compress_tsuid on series
        // (All instances in series should have same tsUid after compression)
        jdbcTemplate.update("""
            UPDATE series SET
                compress_tsuid = (
                    SELECT DISTINCT transfer_syntax_uid FROM instance
                    WHERE series_instance_uid = series.series_instance_uid
                    AND study_instance_uid = ?
                    LIMIT 1
                ),
                compress_time = now()
            WHERE study_instance_uid = ?
            """, studyUid, studyUid);
    }

    /**
     * Get task status.
     */
    public Map<String, Object> getTaskStatus(long taskId) {
        return jdbcTemplate.queryForMap(
            "SELECT * FROM compression_task WHERE id = ?", taskId
        );
    }

    /**
     * List compression tasks (with optional status filter).
     */
    public List<Map<String, Object>> listTasks(String status, int limit) {
        if (status != null) {
            return jdbcTemplate.queryForList(
                "SELECT * FROM compression_task WHERE status = ? ORDER BY created_at DESC LIMIT ?",
                status, limit
            );
        }
        return jdbcTemplate.queryForList(
            "SELECT * FROM compression_task ORDER BY created_at DESC LIMIT ?", limit
        );
    }
}
```

### 5. `src/main/java/com/spax/compression/CompressionController.java`

```java
@RestController
@RequestMapping("/api/v1/{tenant}/admin")
public class CompressionController {

    @Autowired private CompressionService compressionService;

    /**
     * POST /api/v1/{tenant}/admin/studies/{studyUid}/compress
     * Body: { "compressionType": "JPEG_LS_LOSSLESS" }
     *
     * Returns: { "taskId": 123 }
     */
    @PostMapping("/studies/{studyUid}/compress")
    public ResponseEntity<Map<String, Object>> triggerCompression(
            @PathVariable String studyUid,
            @RequestBody Map<String, String> request,
            @RequestParam(defaultValue = "admin") String triggeredBy) {

        String compressionTypeName = request.get("compressionType");
        CompressionType type;
        try {
            type = CompressionType.valueOf(compressionTypeName);
        } catch (IllegalArgumentException e) {
            return ResponseEntity.badRequest().body(Map.of(
                "error", "Invalid compressionType. Valid values: " +
                    Arrays.toString(CompressionType.values())
            ));
        }

        long taskId = compressionService.createTask(studyUid, type, null, triggeredBy);

        return ResponseEntity.accepted().body(Map.of("taskId", taskId, "studyUid", studyUid));
    }

    /**
     * GET /api/v1/{tenant}/admin/compressions
     * Query params: status=PENDING|IN_PROGRESS|COMPLETED|FAILED, limit=20
     */
    @GetMapping("/compressions")
    public ResponseEntity<List<Map<String, Object>>> listTasks(
            @RequestParam(required = false) String status,
            @RequestParam(defaultValue = "20") int limit) {
        return ResponseEntity.ok(compressionService.listTasks(status, limit));
    }

    /**
     * GET /api/v1/{tenant}/admin/compressions/{taskId}
     * Returns: full task details including progress
     */
    @GetMapping("/compressions/{taskId}")
    public ResponseEntity<Map<String, Object>> getTask(@PathVariable long taskId) {
        Map<String, Object> task = compressionService.getTaskStatus(taskId);
        if (task == null) return ResponseEntity.notFound().build();
        return ResponseEntity.ok(task);
    }
}
```

## Lưu ý quan trọng
- `DicomCompressor.compress()` dùng `dcm4che-imageio` + `dcm4che-imageio-opencv` native codecs
- Idempotent: check `transfer_syntax_uid` trước khi compress — nếu đã đúng → skip
- In-place replace: DELETE → WRITE cùng path (không phải atomic nhưng acceptable)
- Nếu WRITE fail sau DELETE → file bị mất. Mitigation: kiểm tra compressed != null/empty trước khi delete
- Virtual thread per task — nhiều compression tasks chạy parallel không block nhau
- `series.compress_tsuid` và `series.compress_time` update sau khi tất cả instances trong series done

## Kiểm tra thành công
- `POST /admin/studies/{uid}/compress {"compressionType": "JPEG_LS_LOSSLESS"}` → taskId trả về
- `GET /admin/compressions/{taskId}` → progress tăng dần
- Sau COMPLETED → file size giảm, `transfer_syntax_uid` trong DB đổi sang `1.2.840.10008.1.2.4.80`
- OHIF viewer load được ảnh sau khi nén (JPEG-LS lossless)
- Re-compress cùng study → idempotent (processed=0, đã skip all)
- Compression của 100 instances CT study hoàn thành không OOM
