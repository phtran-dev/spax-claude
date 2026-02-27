# SPEC-13: Admin — DICOM Correction + Audit Log

## Mục tiêu
Implement DICOM correction module: admin sửa patient/study metadata → update DB ngay lập tức + async sửa file headers + ghi audit log.

## Dependencies
- SPEC-02 (schema — audit_log, file_correction_task tables)
- SPEC-03 (TenantContext)
- SPEC-05 (StorageService — đọc/ghi file)
- SPEC-07 (DicomParser — parse file)

## Files cần tạo

### 1. `src/main/java/com/spax/admin/AuditService.java`

```java
@Service
public class AuditService {

    @Autowired private JdbcTemplate jdbcTemplate;

    /**
     * Record a change to the audit log.
     */
    public void log(String entityType, long entityId, String fieldName,
                    String oldValue, String newValue, String changedBy, String reason) {
        jdbcTemplate.update("""
            INSERT INTO audit_log (entity_type, entity_id, field_name, old_value, new_value, changed_by, reason)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            """,
            entityType, entityId, fieldName, oldValue, newValue, changedBy, reason
        );
    }

    /**
     * Get audit history for an entity.
     */
    public List<AuditEntry> getHistory(String entityType, long entityId) {
        return jdbcTemplate.query("""
            SELECT * FROM audit_log
            WHERE entity_type = ? AND entity_id = ?
            ORDER BY changed_at DESC
            LIMIT 100
            """,
            (rs, rowNum) -> new AuditEntry(
                rs.getLong("id"),
                rs.getString("entity_type"),
                rs.getLong("entity_id"),
                rs.getString("field_name"),
                rs.getString("old_value"),
                rs.getString("new_value"),
                rs.getString("changed_by"),
                rs.getTimestamp("changed_at").toInstant(),
                rs.getString("reason")
            ),
            entityType, entityId
        );
    }

    public record AuditEntry(long id, String entityType, long entityId,
                              String fieldName, String oldValue, String newValue,
                              String changedBy, Instant changedAt, String reason) {}
}
```

### 2. `src/main/java/com/spax/admin/CorrectionService.java`

```java
@Service
public class CorrectionService {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private AuditService auditService;
    @Autowired private ApplicationEventPublisher eventPublisher;

    /**
     * Correct patient information.
     * Updates DB immediately. Queues background job to fix DICOM file headers.
     *
     * @param patientId  DB id (not DICOM patient ID)
     * @param changes    map of field → new value
     * @param changedBy  username of admin
     * @param reason     reason for change (required)
     * @param version    current version for optimistic locking
     */
    @Transactional
    public void correctPatient(long patientId, Map<String, String> changes,
                                String changedBy, String reason, int version) {
        // 1. Load current values for audit
        Map<String, Object> current = jdbcTemplate.queryForMap(
            "SELECT * FROM patient WHERE id = ?", patientId
        );

        // 2. Build UPDATE SET clause
        List<Object> params = new ArrayList<>();
        StringBuilder sql = new StringBuilder("UPDATE patient SET updated_at = now(), version = version + 1");

        if (changes.containsKey("patientName")) {
            sql.append(", patient_name = ?");
            params.add(changes.get("patientName"));
            auditService.log("PATIENT", patientId, "patient_name",
                (String) current.get("patient_name"), changes.get("patientName"), changedBy, reason);
        }

        if (changes.containsKey("birthDate")) {
            sql.append(", birth_date = ?");
            params.add(changes.get("birthDate"));
            auditService.log("PATIENT", patientId, "birth_date",
                String.valueOf(current.get("birth_date")), changes.get("birthDate"), changedBy, reason);
        }

        if (changes.containsKey("sex")) {
            sql.append(", sex = ?");
            params.add(changes.get("sex"));
            auditService.log("PATIENT", patientId, "sex",
                (String) current.get("sex"), changes.get("sex"), changedBy, reason);
        }

        if (changes.containsKey("patientId")) {
            // PID change: also recalculate public_id + background update of study public_ids
            String newPid = changes.get("patientId");
            String newPublicId = sha1(newPid);
            sql.append(", patient_id = ?, public_id = ?");
            params.add(newPid);
            params.add(newPublicId);
            auditService.log("PATIENT", patientId, "patient_id",
                (String) current.get("patient_id"), newPid, changedBy, reason);

            // Queue background job to update study public_ids
            eventPublisher.publishEvent(new PidCorrectionEvent(patientId, (String) current.get("patient_id"), newPid));
        }

        sql.append(" WHERE id = ? AND version = ?");
        params.add(patientId);
        params.add(version);

        int updated = jdbcTemplate.update(sql.toString(), params.toArray());
        if (updated == 0) {
            throw new OptimisticLockingFailureException("Patient was modified by another user. Please refresh.");
        }

        // 3. Queue file correction job
        List<String> affectedStudyUids = jdbcTemplate.queryForList(
            "SELECT study_instance_uid FROM study WHERE patient_fk = ?", String.class, patientId
        );

        for (String studyUid : affectedStudyUids) {
            createFileCorrectionTask(studyUid, changes);
        }
    }

    /**
     * Correct study information.
     */
    @Transactional
    public void correctStudy(long studyFk, Map<String, String> changes,
                              String changedBy, String reason, int version) {
        Map<String, Object> current = jdbcTemplate.queryForMap(
            "SELECT * FROM study WHERE id = ?", studyFk
        );

        List<Object> params = new ArrayList<>();
        StringBuilder sql = new StringBuilder("UPDATE study SET updated_at = now(), version = version + 1");

        Map<String, String> fieldMappings = Map.of(
            "studyDescription", "study_description",
            "accessionNumber", "accession_number",
            "referringPhysician", "referring_physician"
        );

        for (Map.Entry<String, String> change : changes.entrySet()) {
            String colName = fieldMappings.get(change.getKey());
            if (colName != null) {
                sql.append(", ").append(colName).append(" = ?");
                params.add(change.getValue());
                auditService.log("STUDY", studyFk, change.getKey(),
                    (String) current.get(colName), change.getValue(), changedBy, reason);
            }
        }

        sql.append(" WHERE id = ? AND version = ?");
        params.add(studyFk);
        params.add(version);

        int updated = jdbcTemplate.update(sql.toString(), params.toArray());
        if (updated == 0) {
            throw new OptimisticLockingFailureException("Study was modified. Please refresh.");
        }

        // Queue file correction
        String studyUid = (String) current.get("study_instance_uid");
        createFileCorrectionTask(studyUid, changes);
    }

    private void createFileCorrectionTask(String studyUid, Map<String, String> changes) {
        int totalFiles = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM instance WHERE study_instance_uid = ?", Integer.class, studyUid
        );

        jdbcTemplate.update("""
            INSERT INTO file_correction_task (study_instance_uid, correction_type, changes, total_files)
            VALUES (?, 'METADATA_CORRECTION', ?::jsonb, ?)
            """,
            studyUid, toJson(changes), totalFiles
        );
    }

    private String sha1(String input) { /* same impl as BulkInsertRepository */ }
    private String toJson(Map<String, String> map) { /* JSON serialization */ }
}
```

### 3. `src/main/java/com/spax/admin/FileCorrectionJob.java`

Background worker thực sự sửa DICOM file headers:

```java
@Component
public class FileCorrectionJob {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private StorageService storageService;

    @Scheduled(fixedDelay = 30_000)  // Check every 30s
    public void processPendingTasks() {
        List<Long> pendingTaskIds = jdbcTemplate.queryForList(
            "SELECT id FROM file_correction_task WHERE status = 'PENDING' LIMIT 5",
            Long.class
        );

        for (Long taskId : pendingTaskIds) {
            processTask(taskId);
        }
    }

    private void processTask(long taskId) {
        // Mark IN_PROGRESS
        jdbcTemplate.update(
            "UPDATE file_correction_task SET status = 'IN_PROGRESS' WHERE id = ?", taskId
        );

        Map<String, Object> task = jdbcTemplate.queryForMap(
            "SELECT * FROM file_correction_task WHERE id = ?", taskId
        );

        String studyUid = (String) task.get("study_instance_uid");
        Map<String, String> changes = parseJson((String) task.get("changes"));

        // Get all instances of study
        List<Map<String, Object>> instances = jdbcTemplate.queryForList(
            "SELECT volume_id, storage_path FROM instance WHERE study_instance_uid = ?", studyUid
        );

        int processed = 0;
        for (Map<String, Object> instance : instances) {
            int volumeId = (int) instance.get("volume_id");
            String storagePath = (String) instance.get("storage_path");

            try {
                // 1. Read DICOM file
                byte[] dicomBytes;
                try (InputStream in = storageService.retrieve(volumeId, storagePath)) {
                    dicomBytes = in.readAllBytes();
                }

                // 2. Parse + modify tags
                byte[] corrected = applyCorrections(dicomBytes, changes);

                // 3. Write back (overwrite)
                storageService.delete(volumeId, storagePath);
                try (InputStream in = new ByteArrayInputStream(corrected)) {
                    // Need a way to write directly — StorageService.store() creates new path
                    // Use VolumeManager directly for in-place write
                    volumeManager.getProvider(volumeId).write(storagePath, in, corrected.length);
                }

                processed++;
            } catch (Exception e) {
                log.error("Failed to correct file {}/{}: {}", volumeId, storagePath, e.getMessage());
            }
        }

        // Update task
        jdbcTemplate.update("""
            UPDATE file_correction_task
            SET status = CASE WHEN ? = (SELECT total_files FROM file_correction_task WHERE id = ?)
                         THEN 'COMPLETED' ELSE 'FAILED' END,
                processed_files = ?,
                completed_at = now()
            WHERE id = ?
            """, processed, taskId, processed, taskId
        );

        // Invalidate + rebuild series metadata cache cho toàn bộ series trong study
        // (DICOM file headers đã thay đổi → cache cũ không còn hợp lệ)
        rebuildSeriesMetadata(studyUid, tenantCode);
    }

    private void rebuildSeriesMetadata(String studyUid, String tenantCode) {
        List<String> seriesUids = jdbcTemplate.queryForList(
            "SELECT series_instance_uid FROM series WHERE study_instance_uid = ?",
            String.class, studyUid
        );

        TenantContext.setTenantCode(tenantCode);
        try {
            for (String seriesUid : seriesUids) {
                seriesMetadataBuilder.invalidate(seriesUid, tenantCode);
                // Rebuild async — không block correction task completion
                executor.submit(() -> {
                    TenantContext.setTenantCode(tenantCode);
                    try {
                        seriesMetadataBuilder.buildAndSave(seriesUid, tenantCode);
                    } catch (Exception e) {
                        log.warn("Failed to rebuild series metadata after correction: {}", seriesUid, e);
                    } finally {
                        TenantContext.clear();
                    }
                });
            }
        } finally {
            TenantContext.clear();
        }
    }

    @Autowired
    @Qualifier("seriesMetadataBuilder")
    private DicomMetadataBuilder seriesMetadataBuilder;

    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    private byte[] applyCorrections(byte[] dicomBytes, Map<String, String> changes) throws IOException {
        DicomInputStream dis = new DicomInputStream(new ByteArrayInputStream(dicomBytes));
        dis.setIncludeBulkData(DicomInputStream.IncludeBulkData.URI);  // keep pixel data reference
        Attributes attrs = dis.readDataset();
        Attributes fmi = dis.readFileMetaInformation();

        // Apply changes to DICOM tags
        if (changes.containsKey("patientName")) {
            attrs.setString(Tag.PatientName, VR.PN, changes.get("patientName"));
        }
        if (changes.containsKey("patientId")) {
            attrs.setString(Tag.PatientID, VR.LO, changes.get("patientId"));
        }
        if (changes.containsKey("birthDate")) {
            attrs.setString(Tag.PatientBirthDate, VR.DA, changes.get("birthDate"));
        }
        if (changes.containsKey("accessionNumber")) {
            attrs.setString(Tag.AccessionNumber, VR.SH, changes.get("accessionNumber"));
        }
        // Add more as needed

        // Write back to bytes
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        DicomOutputStream dos = new DicomOutputStream(baos, dis.getTransferSyntax());
        dos.writeDataset(fmi, attrs);
        dos.close();

        return baos.toByteArray();
    }
}
```

### 4. `src/main/java/com/spax/admin/CorrectionController.java`

```java
@RestController
@RequestMapping("/api/v1/{tenant}/admin")
public class CorrectionController {

    @Autowired private CorrectionService correctionService;
    @Autowired private AuditService auditService;
    @Autowired private JdbcTemplate jdbcTemplate;

    @PutMapping("/patients/{patientId}")
    public ResponseEntity<Void> correctPatient(
            @PathVariable long patientId,
            @RequestBody CorrectionRequest request) {
        correctionService.correctPatient(patientId, request.changes(), request.changedBy(),
                                          request.reason(), request.version());
        return ResponseEntity.ok().build();
    }

    @PutMapping("/studies/{studyId}")
    public ResponseEntity<Void> correctStudy(
            @PathVariable long studyId,
            @RequestBody CorrectionRequest request) {
        correctionService.correctStudy(studyId, request.changes(), request.changedBy(),
                                        request.reason(), request.version());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/audit")
    public ResponseEntity<List<AuditService.AuditEntry>> getAuditLog(
            @RequestParam String entityType,
            @RequestParam long entityId) {
        return ResponseEntity.ok(auditService.getHistory(entityType, entityId));
    }

    @GetMapping("/corrections")
    public ResponseEntity<List<Map<String, Object>>> getCorrectionTasks(
            @RequestParam(defaultValue = "10") int limit) {
        return ResponseEntity.ok(jdbcTemplate.queryForList(
            "SELECT * FROM file_correction_task ORDER BY created_at DESC LIMIT ?", limit
        ));
    }

    public record CorrectionRequest(Map<String, String> changes, String changedBy, String reason, int version) {}
}
```

## Lưu ý quan trọng
- Optimistic locking: `version` column tăng mỗi lần update → 2 admin sửa cùng lúc → người sau nhận lỗi
- Sửa PID: phải recalculate `public_id` của patient (và async recalculate study public_ids)
- `FileCorrectionJob` dùng `VolumeManager.getProvider()` trực tiếp cho in-place write (StorageService.store() tạo path mới)
- DICOM file correction dùng `IncludeBulkData.URI` để keep pixel data by reference, không load vào RAM
- Audit log ghi từng field riêng (không ghi toàn bộ object) để dễ diff sau này

## Kiểm tra thành công
- Sửa PatientName → DB update ngay → QIDO-RS trả tên mới
- Audit log có entry với old value + new value
- file_correction_task được tạo với total_files đúng
- Sau 30s → FileCorrectionJob chạy → file headers được sửa
- Sửa cùng lúc từ 2 requests → 1 thành công, 1 nhận OptimisticLockingFailureException
