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

```java
@RestController
@RequestMapping("/dicomweb/{tenant}")
public class WadoController {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private StorageService storageService;
    @Autowired private VolumeManager volumeManager;

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
     */
    @GetMapping(
        value = "/studies/{studyUid}/series/{seriesUid}/instances/{sopUid}",
        produces = "application/dicom"
    )
    public ResponseEntity<StreamingResponseBody> retrieveInstance(
            @PathVariable String sopUid) {

        InstanceLocation loc = findInstance(sopUid);
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

    // ─── FRAMES (pixel data) ──────────────────────────────────────────────

    /**
     * GET /dicomweb/{tenant}/.../instances/{sopUid}/frames/{frameList}
     * OHIF gọi để lấy nội dung pixel data theo frame.
     * frameList: "1" hoặc "1,2,3"
     */
    @GetMapping(value = "/studies/{studyUid}/series/{seriesUid}/instances/{sopUid}/frames/{frameList}")
    public ResponseEntity<StreamingResponseBody> retrieveFrames(
            @PathVariable String sopUid,
            @PathVariable String frameList) {

        InstanceLocation loc = findInstance(sopUid);
        if (loc == null) return ResponseEntity.notFound().build();

        String boundary = "frames_" + UUID.randomUUID().toString().replace("-", "");

        StreamingResponseBody body = outputStream -> {
            // Simplified: trả toàn bộ file trong multipart wrapper
            // TODO: proper frame extraction với dcm4che Transcoder khi cần
            PrintWriter writer = new PrintWriter(new OutputStreamWriter(outputStream, StandardCharsets.UTF_8));
            writer.print("\r\n--" + boundary + "\r\n");
            writer.print("Content-Type: application/octet-stream\r\n\r\n");
            writer.flush();
            try (InputStream in = storageService.retrieve(loc.volumeId(), loc.storagePath())) {
                in.transferTo(outputStream);
            }
            writer.print("\r\n--" + boundary + "--\r\n");
            writer.flush();
        };

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(
                "multipart/related; type=\"application/octet-stream\"; boundary=" + boundary
            ))
            .body(body);
    }

    // ─── RETRIEVE STUDY / SERIES (multipart/related) ──────────────────────

    @GetMapping(value = "/studies/{studyUid}")
    public ResponseEntity<StreamingResponseBody> retrieveStudy(@PathVariable String studyUid) {
        List<InstanceLocation> instances = findInstancesByStudy(studyUid);
        if (instances.isEmpty()) return ResponseEntity.notFound().build();
        return buildMultipartResponse(instances);
    }

    @GetMapping(value = "/studies/{studyUid}/series/{seriesUid}")
    public ResponseEntity<StreamingResponseBody> retrieveSeries(@PathVariable String seriesUid) {
        List<InstanceLocation> instances = findInstancesBySeries(seriesUid);
        if (instances.isEmpty()) return ResponseEntity.notFound().build();
        return buildMultipartResponse(instances);
    }

    // ─── Helpers ──────────────────────────────────────────────────────────

    private ResponseEntity<StreamingResponseBody> buildMultipartResponse(List<InstanceLocation> instances) {
        String boundary = "DICOMweb_" + UUID.randomUUID().toString().replace("-", "");
        StreamingResponseBody body = outputStream -> {
            PrintWriter writer = new PrintWriter(new OutputStreamWriter(outputStream, StandardCharsets.UTF_8));
            for (InstanceLocation loc : instances) {
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

    private InstanceLocation findInstance(String sopUid) {
        List<InstanceLocation> r = jdbcTemplate.query(
            "SELECT volume_id, storage_path FROM instance WHERE sop_instance_uid = ? LIMIT 1",
            (rs, i) -> new InstanceLocation(rs.getInt("volume_id"), rs.getString("storage_path")),
            sopUid
        );
        return r.isEmpty() ? null : r.get(0);
    }

    private List<InstanceLocation> findInstancesByStudy(String studyUid) {
        return jdbcTemplate.query(
            "SELECT volume_id, storage_path FROM instance WHERE study_instance_uid = ? ORDER BY instance_number",
            (rs, i) -> new InstanceLocation(rs.getInt("volume_id"), rs.getString("storage_path")),
            studyUid
        );
    }

    private List<InstanceLocation> findInstancesBySeries(String seriesUid) {
        return jdbcTemplate.query(
            "SELECT volume_id, storage_path FROM instance WHERE series_instance_uid = ? ORDER BY instance_number",
            (rs, i) -> new InstanceLocation(rs.getInt("volume_id"), rs.getString("storage_path")),
            seriesUid
        );
    }

    record InstanceLocation(int volumeId, String storagePath) {}
}
```

## Lưu ý quan trọng
- `StreamingResponseBody` bắt buộc — không buffer toàn bộ file vào RAM
- `SeriesMetadataBuilder.getOrBuild()` handle cả 2 cases: cache hit (fast) và fallback theo provider type
- **Local fallback**: đọc trực tiếp + async rebuild. User sẽ thấy chậm lần đầu, nhanh từ lần 2.
- **Cloud fallback**: sync build bắt buộc — không thể đọc N DICOM files từ cloud per request
- Frame extraction hiện tại simplified (trả full file) — đủ cho OHIF single-frame images. Multi-frame cần implement proper extraction sau.
- `DicomMetadataBuilder` là interface → sau này có thể thêm `StudyMetadataBuilder` không cần sửa WadoController

## Kiểm tra thành công
- Sau ingest → `series.metadata_path` populated
- `GET /dicomweb/test/studies/{uid}/series/{uid}/metadata` → trả DICOM JSON array ngay lập tức (từ cache file)
- Xóa `metadata_path` trong DB → request tiếp theo fallback đọc DICOM files → rebuild cache → request sau dùng cache
- OHIF Viewer mở được 512-slice CT series mà không timeout
- Cloud volume: metadata_path = null → first request builds synchronously → cached cho các request sau
