# SPEC-11: DICOMWeb WADO-RS (Retrieve) + DicomMetadataBuilder

## Mục tiêu
Implement WADO-RS endpoints để OHIF Viewer tải ảnh DICOM. Trọng tâm là endpoint **series metadata** — OHIF gọi đây để lấy full metadata của tất cả instances trước khi render. Dùng pre-built cache file để tránh đọc N DICOM files mỗi request.

## 3 URL chính OHIF gọi
1. `GET /studies/{uid}/series?...` → **QIDO** (SPEC-10, đã xử lý bằng DB columns)
2. `GET /studies/{uid}/series/{uid}/metadata` → **WADO-RS metadata** — **bottleneck, cần cache**
3. `GET /studies/{uid}/series/{uid}/instances/{sop}/frames/{frame}` → **WADO-RS frames** — đọc pixel data từ file

## Dependencies
- SPEC-02 (series.metadata_path, series.metadata_volume_id)
- SPEC-03 (TenantContext)
- SPEC-05 (StorageService, VolumeManager)
- SPEC-07 (DicomParser — dùng trong SeriesMetadataBuilder fallback)
- SPEC-21 (WadoRsCacheService — cached instance location lookup)

## Files cần tạo

### 1. `src/main/java/com/spax/dicomweb/DicomMetadataBuilder.java`

Interface chung — không bó cứng level (series, study, ...) vào tên:

```java
public interface DicomMetadataBuilder {

    /**
     * Build metadata JSON từ tất cả DICOM files của entity (series hoặc study),
     * ghi cache file lên storage, update DB.
     *
     * @param uid        seriesInstanceUid hoặc studyInstanceUid tùy impl
     * @param tenantCode tenant hiện tại
     * @return storage path của cache file vừa tạo (relative path within volume)
     */
    String buildAndSave(String uid, String tenantCode) throws IOException;

    /**
     * Serve metadata: đọc từ cache nếu có, không thì build (sync hoặc async tùy provider).
     *
     * @return InputStream của DICOM JSON array (caller phải đóng)
     */
    InputStream getOrBuild(String uid, String tenantCode) throws IOException;

    /**
     * Xóa cache path trong DB (không xóa file — để GC sau).
     * Gọi trước khi rebuild (FileCorrectionJob, compression, ...).
     */
    void invalidate(String uid, String tenantCode);
}
```

### 2. `src/main/java/com/spax/dicomweb/SeriesMetadataBuilder.java`

Implement `DicomMetadataBuilder` cho series level:

```java
@Component("seriesMetadataBuilder")
public class SeriesMetadataBuilder implements DicomMetadataBuilder {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private VolumeManager volumeManager;
    @Autowired private ObjectMapper objectMapper;  // Jackson

    /**
     * Build DICOM JSON array cho toàn bộ instances của 1 series.
     * Ghi file JSON lên cùng volume với DICOM files.
     * Update series.metadata_path + series.metadata_volume_id.
     */
    @Override
    public String buildAndSave(String seriesUid, String tenantCode) throws IOException {
        // 1. Query tất cả instances (volume_id + storage_path, theo thứ tự instance_number)
        List<Map<String, Object>> instances = jdbcTemplate.queryForList("""
            SELECT volume_id, storage_path
            FROM instance
            WHERE series_instance_uid = ?
            ORDER BY instance_number
            """, seriesUid
        );

        if (instances.isEmpty()) return null;

        int volumeId = (int) instances.get(0).get("volume_id");
        StorageProvider provider = volumeManager.getProvider(volumeId);

        // 2. Parse từng DICOM file → DICOM JSON (skip pixel data)
        List<Map<String, Object>> dicomJsonArray = new ArrayList<>();
        for (Map<String, Object> inst : instances) {
            try (InputStream in = provider.read((String) inst.get("storage_path"))) {
                DicomInputStream dis = new DicomInputStream(in);
                dis.setIncludeBulkData(DicomInputStream.IncludeBulkData.NO);
                Attributes attrs = dis.readDataset();
                dicomJsonArray.add(attributesToDicomJson(attrs));
            }
        }

        // 3. Serialize → JSON bytes
        byte[] jsonBytes = objectMapper.writeValueAsBytes(dicomJsonArray);

        // 4. Write cache file lên cùng volume
        // Convention path: {tenantCode}/series-meta/{uid[0:2]}/{uid[2:4]}/{seriesUid}.json
        String metadataPath = tenantCode + "/series-meta/"
            + seriesUid.substring(0, 2) + "/"
            + seriesUid.substring(2, 4) + "/"
            + seriesUid + ".json";

        try (InputStream in = new ByteArrayInputStream(jsonBytes)) {
            provider.write(metadataPath, in, jsonBytes.length);
        }

        // 5. Update DB
        jdbcTemplate.update("""
            UPDATE series
            SET metadata_volume_id = ?, metadata_path = ?
            WHERE series_instance_uid = ?
            """, volumeId, metadataPath, seriesUid
        );

        log.debug("Built series metadata cache for {}: {} instances, {} bytes",
            seriesUid, instances.size(), jsonBytes.length);

        return metadataPath;
    }

    /**
     * Serve metadata từ cache (fast path), hoặc build nếu chưa có.
     *
     * Fallback strategy khi metadata_path = null:
     * - LOCAL provider → đọc DICOM files trực tiếp (1 request OK) + async rebuild cache
     * - CLOUD provider → buộc build synchronously trước khi trả về
     *   (tránh N API calls tốn tiền mỗi lần request)
     */
    @Override
    public InputStream getOrBuild(String seriesUid, String tenantCode) throws IOException {
        Map<String, Object> series = jdbcTemplate.queryForMap("""
            SELECT metadata_volume_id, metadata_path,
                   (SELECT provider_type FROM public.storage_volume
                    WHERE id = (SELECT volume_id FROM instance
                                WHERE series_instance_uid = ? LIMIT 1)) AS provider_type
            FROM series WHERE series_instance_uid = ?
            """, seriesUid, seriesUid
        );

        String metadataPath = (String) series.get("metadata_path");

        // ── Fast path: cache file đã có ──────────────────────────────────
        if (metadataPath != null) {
            int volId = (int) series.get("metadata_volume_id");
            return volumeManager.getProvider(volId).read(metadataPath);
        }

        // ── Fallback: chưa có cache ───────────────────────────────────────
        String providerType = (String) series.get("provider_type");
        boolean isCloud = providerType != null && !"LOCAL".equals(providerType);

        if (isCloud) {
            // Cloud: sync build (không thể đọc N files mỗi request, tốn tiền)
            String newPath = buildAndSave(seriesUid, tenantCode);
            Integer volId = jdbcTemplate.queryForObject(
                "SELECT metadata_volume_id FROM series WHERE series_instance_uid = ?",
                Integer.class, seriesUid
            );
            return volumeManager.getProvider(volId).read(newPath);
        } else {
            // Local: đọc DICOM files trực tiếp lần này + async rebuild cache
            executor.submit(() -> {
                try { buildAndSave(seriesUid, tenantCode); }
                catch (Exception e) { log.warn("Async metadata rebuild failed for {}", seriesUid, e); }
            });
            return buildFromDicomFiles(seriesUid);
        }
    }

    @Override
    public void invalidate(String seriesUid, String tenantCode) {
        // Chỉ xóa path trong DB, không xóa file ngay (file sẽ bị overwrite lần rebuild sau)
        jdbcTemplate.update(
            "UPDATE series SET metadata_path = NULL, metadata_volume_id = NULL WHERE series_instance_uid = ?",
            seriesUid
        );
    }

    // ─── build trực tiếp từ DICOM files (không cache) ─────────────────────

    private InputStream buildFromDicomFiles(String seriesUid) throws IOException {
        List<Map<String, Object>> instances = jdbcTemplate.queryForList("""
            SELECT volume_id, storage_path FROM instance
            WHERE series_instance_uid = ? ORDER BY instance_number
            """, seriesUid
        );

        List<Map<String, Object>> dicomJsonArray = new ArrayList<>();
        for (Map<String, Object> inst : instances) {
            int volId = (int) inst.get("volume_id");
            try (InputStream in = volumeManager.getProvider(volId).read((String) inst.get("storage_path"))) {
                DicomInputStream dis = new DicomInputStream(in);
                dis.setIncludeBulkData(DicomInputStream.IncludeBulkData.NO);
                dicomJsonArray.add(attributesToDicomJson(dis.readDataset()));
            }
        }

        byte[] jsonBytes = objectMapper.writeValueAsBytes(dicomJsonArray);
        return new ByteArrayInputStream(jsonBytes);
    }

    // ─── Convert dcm4che Attributes → DICOM JSON Map ──────────────────────

    private Map<String, Object> attributesToDicomJson(Attributes attrs) throws IOException {
        // Dùng dcm4che DicomJSON writer
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        JsonGenerator gen = Json.createGenerator(baos);
        new DicomJSON(gen).write(attrs);
        gen.flush();
        // Parse back as Map (Jackson sẽ serialize về đúng format)
        return objectMapper.readValue(baos.toByteArray(), Map.class);
    }

    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
}
```

### 3. `src/main/java/com/spax/dicomweb/WadoController.java`

Dùng `WadoRsCacheService` (SPEC-21) thay vì query DB trực tiếp. Instance location được batch-load
theo series và cache lại — 1000 frame requests chỉ cần 1 DB query thay vì 1000.

```java
@RestController
@RequestMapping("/dicomweb/{tenant}")
public class WadoController {

    @Autowired private StorageService storageService;
    @Autowired private WadoRsCacheService wadoRsCacheService;
    @Autowired private CacheManager cacheManager;
    @Autowired private FrameRetrievalService frameRetrievalService;

    @Autowired
    @Qualifier("seriesMetadataBuilder")
    private DicomMetadataBuilder seriesMetadataBuilder;

    // ─── SERIES METADATA (key endpoint for OHIF) ──────────────────────────

    /**
     * GET /dicomweb/{tenant}/studies/{studyUid}/series/{seriesUid}/metadata
     *
     * OHIF gọi endpoint này để load full metadata của series trước khi render.
     * Serve từ pre-built cache file nếu có (fast), fallback tùy provider type.
     *
     * Returns: application/dicom+json — JSON array của tất cả instances
     */
    @GetMapping(
        value = "/studies/{studyUid}/series/{seriesUid}/metadata",
        produces = "application/dicom+json"
    )
    public ResponseEntity<StreamingResponseBody> seriesMetadata(
            @PathVariable String tenant,
            @PathVariable String seriesUid) {

        StreamingResponseBody body = outputStream -> {
            try (InputStream in = seriesMetadataBuilder.getOrBuild(seriesUid, tenant)) {
                in.transferTo(outputStream);
            }
        };

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom+json"))
            .body(body);
    }

    // ─── RETRIEVE SINGLE INSTANCE ──────────────────────────────────────────

    /**
     * GET /dicomweb/{tenant}/studies/{studyUid}/series/{seriesUid}/instances/{sopUid}
     * Returns: raw DICOM file bytes
     *
     * Dùng WadoRsCacheService: batch load toàn bộ series vào cache khi miss.
     * seriesUid từ URL được dùng làm cache key — implicit validation.
     */
    @GetMapping(
        value = "/studies/{studyUid}/series/{seriesUid}/instances/{sopUid}",
        produces = "application/dicom"
    )
    public ResponseEntity<StreamingResponseBody> retrieveInstance(
            @PathVariable String tenant,
            @PathVariable String seriesUid,
            @PathVariable String sopUid) {

        WadoRsCacheService.InstanceLocation loc =
            wadoRsCacheService.findInstance(cacheManager, tenant, seriesUid, sopUid);
        if (loc == null) return ResponseEntity.notFound().build();

        StreamingResponseBody body = outputStream -> {
            try (InputStream in = storageService.retrieve(loc.volumeId(), loc.storagePath())) {
                in.transferTo(outputStream);
            }
        };

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom"))
            .body(body);
    }

    // ─── FRAMES (pixel data) — SPEC-22 ────────────────────────────────────

    /**
     * GET /dicomweb/{tenant}/.../instances/{sopUid}/frames/{frameList}
     *
     * OHIF gọi để lấy pixel data frames.
     * frameList: "1" hoặc "1,3,5" (1-based, comma-separated).
     *
     * Returns: multipart/related — mỗi frame = 1 MIME part.
     * - Uncompressed: Content-Type: application/octet-stream
     * - Compressed: Content-Type: application/octet-stream; transfer-syntax={tsuid}
     *
     * Single-pass: 1 file open, 1 header parse cho tất cả frames (SPEC-22).
     * Cache: batch load toàn bộ series → 1000 frame requests chỉ 1 DB query.
     */
    @GetMapping(value = "/studies/{studyUid}/series/{seriesUid}/instances/{sopUid}/frames/{frameList}")
    public ResponseEntity<StreamingResponseBody> retrieveFrames(
            @PathVariable String tenant,
            @PathVariable String seriesUid,
            @PathVariable String sopUid,
            @PathVariable String frameList) {

        // 1. Cache lookup (batch load series on miss)
        WadoRsCacheService.InstanceLocation loc =
            wadoRsCacheService.findInstance(cacheManager, tenant, seriesUid, sopUid);
        if (loc == null) return ResponseEntity.notFound().build();

        // 2. Parse + sort frame list: "5,1,3" → [1, 3, 5]
        List<Integer> frameNumbers;
        try {
            frameNumbers = Arrays.stream(frameList.split(","))
                .map(String::trim)
                .map(Integer::parseInt)
                .sorted()
                .toList();
        } catch (NumberFormatException e) {
            return ResponseEntity.badRequest().build();
        }
        if (frameNumbers.isEmpty() || frameNumbers.getFirst() < 1) {
            return ResponseEntity.badRequest().build();
        }

        // 3. Validate frame range
        if (frameNumbers.getLast() > loc.numFrames()) {
            return ResponseEntity.badRequest().build();
        }

        // 4. Determine outer Content-Type header
        String boundary = "frames_" + UUID.randomUUID().toString().replace("-", "");
        FrameType frameType = FrameType.classify(loc.transferSyntaxUid(), loc.numFrames());
        String contentType;
        if (frameType.isCompressed()) {
            contentType = "multipart/related; type=\"application/octet-stream; transfer-syntax="
                + loc.transferSyntaxUid() + "\"; boundary=" + boundary;
        } else {
            contentType = "multipart/related; type=\"application/octet-stream\"; boundary=" + boundary;
        }

        // 5. Stream response (non-buffered, single-pass frame extraction)
        StreamingResponseBody body = outputStream -> {
            frameRetrievalService.writeFrames(loc, frameNumbers, outputStream, boundary);
        };

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(contentType))
            .body(body);
    }

    // ─── RETRIEVE STUDY / SERIES (multipart/related) ──────────────────────

    @GetMapping(value = "/studies/{studyUid}")
    public ResponseEntity<StreamingResponseBody> retrieveStudy(
            @PathVariable String tenant,
            @PathVariable String studyUid) {
        // Study-level retrieve: query tất cả series → batch load instance locations
        // TODO: optimize bằng study-level cache nếu cần
        List<WadoRsCacheService.InstanceLocation> instances =
            wadoRsCacheService.findAllInStudy(cacheManager, tenant, studyUid);
        if (instances.isEmpty()) return ResponseEntity.notFound().build();
        return buildMultipartResponse(instances);
    }

    @GetMapping(value = "/studies/{studyUid}/series/{seriesUid}")
    public ResponseEntity<StreamingResponseBody> retrieveSeries(
            @PathVariable String tenant,
            @PathVariable String seriesUid) {
        List<WadoRsCacheService.InstanceLocation> instances =
            wadoRsCacheService.findAllInSeries(cacheManager, tenant, seriesUid);
        if (instances.isEmpty()) return ResponseEntity.notFound().build();
        return buildMultipartResponse(instances);
    }

    // ─── Helpers ──────────────────────────────────────────────────────────

    private ResponseEntity<StreamingResponseBody> buildMultipartResponse(
            List<WadoRsCacheService.InstanceLocation> instances) {
        String boundary = "DICOMweb_" + UUID.randomUUID().toString().replace("-", "");
        StreamingResponseBody body = outputStream -> {
            PrintWriter writer = new PrintWriter(new OutputStreamWriter(outputStream, StandardCharsets.UTF_8));
            for (WadoRsCacheService.InstanceLocation loc : instances) {
                writer.print("\r\n--" + boundary + "\r\n");
                writer.print("Content-Type: application/dicom\r\n\r\n");
                writer.flush();
                try (InputStream in = storageService.retrieve(loc.volumeId(), loc.storagePath())) {
                    in.transferTo(outputStream);
                    outputStream.flush();
                }
            }
            writer.print("\r\n--" + boundary + "--\r\n");
            writer.flush();
        };
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(
                "multipart/related; type=\"application/dicom\"; boundary=" + boundary
            ))
            .body(body);
    }
}
```

**Thay đổi so với bản cũ:**
- Bỏ `JdbcTemplate` inject trực tiếp — mọi DB query đi qua `WadoRsCacheService`
- Bỏ private helper methods `findInstance()`, `findInstancesByStudy()`, `findInstancesBySeries()` — thay bằng cached service calls
- Bỏ inner record `InstanceLocation` — dùng `WadoRsCacheService.InstanceLocation`
- `retrieveInstance()` và `retrieveFrames()` thêm `@PathVariable tenant, seriesUid` — cần cho cache key
- `retrieveStudy()` và `retrieveSeries()` thêm `@PathVariable tenant` — cần cho cache key
- URL validation: loose — `studyUid` không validate, `seriesUid` implicit validate qua cache lookup
```

## Lưu ý quan trọng
- `StreamingResponseBody` bắt buộc — không buffer toàn bộ file vào RAM
- `SeriesMetadataBuilder.getOrBuild()` handle cả 2 cases: cache hit (fast) và fallback theo provider type
- **Local fallback**: đọc trực tiếp + async rebuild. User sẽ thấy chậm lần đầu, nhanh từ lần 2.
- **Cloud fallback**: sync build bắt buộc — không thể đọc N DICOM files từ cloud per request
- Frame extraction: proper implementation cho tất cả 4 loại ảnh (SPEC-22). Single-pass sequential qua requested frames.
- `DicomMetadataBuilder` là interface → sau này có thể thêm `StudyMetadataBuilder` không cần sửa WadoController
- **Caching** (SPEC-21): `WadoRsCacheService` batch-load instance locations theo series. 1000 frame requests cho 1 CT series → 1 DB query (2-step: series FK lookup + instance batch) thay vì 1000 queries scan 12+ partitions mỗi cái.

## Kiểm tra thành công
- Sau ingest → `series.metadata_path` populated
- `GET /dicomweb/test/studies/{uid}/series/{uid}/metadata` → trả DICOM JSON array ngay lập tức (từ cache file)
- Xóa `metadata_path` trong DB → request tiếp theo fallback đọc DICOM files → rebuild cache → request sau dùng cache
- OHIF Viewer mở được 512-slice CT series mà không timeout
- Cloud volume: metadata_path = null → first request builds synchronously → cached cho các request sau
- **Cache test**: request frame 1 → cache miss (batch load) → request frame 2-1000 → cache hit (0 DB queries)
