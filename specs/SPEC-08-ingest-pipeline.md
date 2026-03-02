# SPEC-08: Ingest Pipeline (IngestController + IngestService + IndexingConsumer + TransferCommitListener)

## Mục tiêu
Implement toàn bộ ingest pipeline: 3 đường nhận file → 1 queue → 1 consumer xử lý. Đây là core functionality của SPAX.

## Dependencies
- SPEC-02 (schema — patient/study/series/instance tables)
- SPEC-03 (TenantContext)
- SPEC-05 (StorageService)
- SPEC-06 (IngestQueue, IngestMessage)
- SPEC-07 (DicomParser, DicomMetadata)
- SPEC-09 (BulkInsertRepository — cần trước khi implement IndexingConsumer)
- SPEC-21 (TenantCacheService, WadoRsCacheService — cached tenant list + cache invalidation)

## Files cần tạo

### 1. `src/main/java/com/spax/ingest/IngestService.java`

Core service: nhận file path → move to temp → publish to queue:

```java
@Service
public class IngestService {
    @Autowired private IngestQueue ingestQueue;

    @Value("${spax.ingest.temp-dir:/tmp/spax/incoming}")
    private String tempDir;

    /**
     * Accept a DICOM file already on disk (from Transfer Server or REST upload).
     * Publishes to queue for async indexing.
     */
    public void acceptFile(Path filePath, String tenantCode) {
        IngestMessage msg = new IngestMessage(
            filePath.toAbsolutePath().toString(),
            tenantCode,
            Instant.now()
        );
        ingestQueue.publish(msg);
    }

    /**
     * Accept raw bytes (from STOW-RS or batch upload).
     * Write to temp file first, then publish.
     */
    public void acceptBytes(byte[] dicomBytes, String tenantCode) throws IOException {
        Path tempDirPath = Path.of(tempDir, tenantCode);
        Files.createDirectories(tempDirPath);
        Path tempFile = tempDirPath.resolve(UUID.randomUUID() + ".dcm");
        Files.write(tempFile, dicomBytes);
        acceptFile(tempFile, tenantCode);
    }
}
```

### 2. `src/main/java/com/spax/ingest/IngestController.java`

REST endpoint cho batch upload:

```java
@RestController
@RequestMapping("/api/v1/{tenant}/ingest")
public class IngestController {

    @Autowired private IngestService ingestService;

    /**
     * POST /api/v1/{tenant}/ingest
     * Accept: multipart/form-data
     * Body: one or more DICOM files with field name "file" or "files"
     *
     * Returns:
     * 200: { "received": N, "queued": N }
     * 507: Insufficient Storage (disk almost full)
     */
    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<Map<String, Object>> ingest(
            @PathVariable String tenant,
            @RequestParam("files") List<MultipartFile> files) {

        int queued = 0;
        for (MultipartFile file : files) {
            try {
                byte[] bytes = file.getBytes();
                ingestService.acceptBytes(bytes, tenant);
                queued++;
            } catch (IOException e) {
                log.warn("Failed to accept file {}: {}", file.getOriginalFilename(), e.getMessage());
            }
        }

        return ResponseEntity.ok(Map.of("received", files.size(), "queued", queued));
    }
}
```

### 3. `src/main/java/com/spax/transfer/TransferCommitListener.java`

Integration point với Transfer Server (Orthanc plugin):

```java
@RestController
@RequestMapping("/api/v1/transfer/commit")
public class TransferCommitListener {

    @Autowired private IngestService ingestService;

    /**
     * POST /api/v1/transfer/commit
     * Called by Transfer Server when a batch of files has been committed to temp dir.
     *
     * Body:
     * {
     *   "tenantCode": "hospital_a",
     *   "files": ["/tmp/spax/transfer/xxx.dcm", ...]
     * }
     *
     * Returns 200 immediately. Actual indexing is async.
     */
    @PostMapping
    public ResponseEntity<Void> commit(@RequestBody TransferCommitRequest request) {
        for (String filePath : request.files()) {
            ingestService.acceptFile(Path.of(filePath), request.tenantCode());
        }
        return ResponseEntity.ok().build();
    }

    public record TransferCommitRequest(String tenantCode, List<String> files) {}
}
```

### 4. `src/main/java/com/spax/ingest/IndexingConsumer.java`

Core indexing worker — consume từ queue, parse DICOM, move file, index DB:

```java
@Component
public class IndexingConsumer {

    @Autowired private RedisStreamIngestQueue queue;  // direct dep for per-tenant consume
    @Autowired private DicomParser dicomParser;
    @Autowired private StorageService storageService;
    @Autowired private BulkInsertRepository bulkInsertRepository;
    @Autowired private TenantManagementService tenantManagementService;
    @Autowired private TenantCacheService tenantCacheService;         // SPEC-21
    @Autowired private WadoRsCacheService wadoRsCacheService;         // SPEC-21
    @Autowired private CacheManager cacheManager;                     // SPEC-21

    @Value("${spax.ingest.batch-size:200}")
    private int batchSize;

    @Value("${spax.ingest.consumer-threads:4}")
    private int consumerThreads;

    private ExecutorService executor;
    private volatile boolean running = true;

    @PostConstruct
    public void start() {
        // Use virtual threads for IO-bound indexing
        executor = Executors.newVirtualThreadPerTaskExecutor();

        // Start consumer loops for each active tenant
        // In practice, get tenant list from DB
        // For simplicity, start 1 generic consumer that polls all active tenants
        for (int i = 0; i < consumerThreads; i++) {
            executor.submit(this::consumeLoop);
        }
    }

    @PreDestroy
    public void stop() {
        running = false;
        executor.shutdown();
    }

    private void consumeLoop() {
        while (running) {
            try {
                // Get active tenants from DB (cache this, refresh every 60s)
                List<String> tenants = getActiveTenants();
                for (String tenantCode : tenants) {
                    queue.consumeForTenant(tenantCode, batchSize, messages ->
                        processMessages(tenantCode, messages)
                    );
                }
            } catch (Exception e) {
                log.error("Consumer loop error, retrying in 5s", e);
                sleepSafe(5000);
            }
        }
    }

    private void processMessages(String tenantCode, List<IngestMessage> messages) {
        // Set tenant context for DB operations
        TenantContext.setTenantCode(tenantCode);
        try {
            // Collect results per DICOM hierarchy
            List<DicomMetadata> parsed = new ArrayList<>();

            for (IngestMessage msg : messages) {
                try {
                    DicomMetadata metadata = parseDicomFile(msg.filePath());

                    // Move file from temp dir to permanent storage
                    byte[] bytes = Files.readAllBytes(Path.of(msg.filePath()));
                    StoreResult stored = storageService.store(tenantCode, bytes, metadata.rawAttributes());

                    // Attach storage info to metadata for DB insert
                    parsed.add(metadata.withStorage(stored.volumeId(), stored.storagePath(), stored.fileSize()));

                    // Delete temp file
                    Files.deleteIfExists(Path.of(msg.filePath()));

                } catch (Exception e) {
                    log.error("Failed to process {}, moving to error dir", msg.filePath(), e);
                    moveToError(msg.filePath(), tenantCode);
                }
            }

            // Batch upsert all parsed metadata in 1 transaction
            if (!parsed.isEmpty()) {
                bulkInsertRepository.batchUpsert(parsed);

                // Invalidate caches cho affected series/studies (SPEC-21)
                // Khi resend images vào series đã có → cache cũ thiếu instances mới
                Set<String> affectedSeriesUids = parsed.stream()
                    .map(item -> item.dicom().seriesInstanceUid())
                    .collect(Collectors.toSet());
                Set<String> affectedStudyUids = parsed.stream()
                    .map(item -> item.dicom().studyInstanceUid())
                    .collect(Collectors.toSet());

                wadoRsCacheService.evictInstanceLocations(cacheManager, tenantCode, affectedSeriesUids);
                for (String seriesUid : affectedSeriesUids) {
                    wadoRsCacheService.evictSeriesMetadataInfo(tenantCode, seriesUid);
                }
                for (String studyUid : affectedStudyUids) {
                    wadoRsCacheService.evictSeriesByStudy(tenantCode, studyUid);
                }
            }

        } finally {
            TenantContext.clear();
        }
    }

    private DicomMetadata parseDicomFile(String filePath) throws IOException {
        try (InputStream in = new BufferedInputStream(new FileInputStream(filePath))) {
            return dicomParser.parse(in);
        }
    }

    private void moveToError(String filePath, String tenantCode) {
        try {
            Path src = Path.of(filePath);
            Path errorDir = src.getParent().resolveSibling("error").resolve(tenantCode);
            Files.createDirectories(errorDir);
            Files.move(src, errorDir.resolve(src.getFileName()), StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            log.error("Failed to move file to error dir: {}", filePath, e);
        }
    }

    private List<String> getActiveTenants() {
        // Cached via TenantCacheService (SPEC-21) — TTL 60s, avoids DB hit every loop iteration
        return tenantCacheService.getActiveTenantCodes();
    }

    private void sleepSafe(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

**Note về `DicomMetadata.withStorage()`**: Cần thêm method hoặc dùng builder pattern để thêm storage info. Hoặc tạo `IndexedDicomMetadata` extends thêm `volumeId`, `storagePath`, `fileSize`.

### Cần thêm: `IndexedDicomMetadata.java`

```java
public record IndexedDicomMetadata(
    DicomMetadata dicom,
    int volumeId,
    String storagePath,
    long fileSize
) {}
```

`BulkInsertRepository.batchUpsert(List<IndexedDicomMetadata> items)` — xem SPEC-09.

## Lưu ý quan trọng
- Virtual threads phù hợp vì indexing là IO-bound (disk read + network DB)
- Batch tối đa 200 messages rồi flush 1 lần → 1 DB transaction / 200 files
- Nếu parse thất bại → move sang `/error/` → KHÔNG block pipeline
- Nếu DB insert thất bại → handler throw exception → messages không ACKed → retry toàn bộ batch
- `TenantContext.clear()` trong finally — bắt buộc để không leak tenant sang thread khác
- File temp bị xóa SAU KHI lưu vào storage và index DB thành công

## Kiểm tra thành công
- Upload 1 DICOM file → file xuất hiện trong storage, DB có record patient/study/series/instance
- Upload 100 file cùng lúc → tất cả indexed, không mất file
- Upload file corrupt → chuyển sang `/error/`, pipeline tiếp tục
- Retry: kill process giữa chừng → restart → files được redelivered, không duplicate
