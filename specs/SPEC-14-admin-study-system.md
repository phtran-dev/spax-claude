# SPEC-14: Admin — Study Management + System Info

## Mục tiêu
Implement study deletion, patient merge, và system info endpoints.

## Dependencies
- SPEC-02 (schema)
- SPEC-03 (TenantContext)
- SPEC-05 (StorageService — xóa files)

## Files cần tạo

### 1. `src/main/java/com/spax/admin/StudyAdminController.java`

```java
@RestController
@RequestMapping("/api/v1/{tenant}/admin")
public class StudyAdminController {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private StorageService storageService;
    @Autowired private VolumeManager volumeManager;

    // ─── DELETE STUDY ─────────────────────────────────────────────────────

    /**
     * DELETE /api/v1/{tenant}/admin/studies/{studyUid}
     *
     * Deletes:
     * 1. All DICOM files from storage
     * 2. All instances from DB
     * 3. All series from DB
     * 4. Study from DB
     * 5. Patient if no more studies (optional, query param: deletePatientIfEmpty=true)
     * 6. Ghi audit log
     *
     * Returns 204 No Content on success.
     */
    @DeleteMapping("/studies/{studyUid}")
    @Transactional
    public ResponseEntity<Map<String, Object>> deleteStudy(
            @PathVariable String studyUid,
            @RequestParam(defaultValue = "false") boolean deletePatientIfEmpty,
            @RequestParam String deletedBy,
            @RequestParam(required = false) String reason) {

        // 1. Get all instances (need volume_id + storage_path)
        List<Map<String, Object>> instances = jdbcTemplate.queryForList(
            "SELECT id, volume_id, storage_path FROM instance WHERE study_instance_uid = ?",
            studyUid
        );

        // 2. Delete files from storage
        int filesDeleted = 0;
        int filesFailed = 0;
        for (Map<String, Object> inst : instances) {
            try {
                storageService.delete((Integer) inst.get("volume_id"), (String) inst.get("storage_path"));
                filesDeleted++;
            } catch (Exception e) {
                log.warn("Failed to delete file for instance {}: {}", inst.get("id"), e.getMessage());
                filesFailed++;
            }
        }

        // 3. Delete from DB (cascade: series → instances auto-deleted? No, instance is partitioned)
        // Must delete instances explicitly
        jdbcTemplate.update("DELETE FROM instance WHERE study_instance_uid = ?", studyUid);
        jdbcTemplate.update("DELETE FROM series WHERE study_instance_uid = ?", studyUid);

        long studyFk = jdbcTemplate.queryForObject(
            "SELECT id FROM study WHERE study_instance_uid = ?", Long.class, studyUid
        );
        Long patientFk = jdbcTemplate.queryForObject(
            "SELECT patient_fk FROM study WHERE id = ?", Long.class, studyFk
        );

        jdbcTemplate.update("DELETE FROM study WHERE study_instance_uid = ?", studyUid);

        // 4. Optionally delete patient if no more studies
        if (deletePatientIfEmpty && patientFk != null) {
            int remainingStudies = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM study WHERE patient_fk = ?", Integer.class, patientFk
            );
            if (remainingStudies == 0) {
                jdbcTemplate.update("DELETE FROM patient WHERE id = ?", patientFk);
            }
        } else if (patientFk != null) {
            // Update patient.num_studies counter
            jdbcTemplate.update("""
                UPDATE patient SET
                    num_studies = (SELECT COUNT(*) FROM study WHERE patient_fk = ?)
                WHERE id = ?
                """, patientFk, patientFk);
        }

        // 5. Audit log
        jdbcTemplate.update("""
            INSERT INTO audit_log (entity_type, entity_id, field_name, old_value, new_value, changed_by, reason)
            VALUES ('STUDY', ?, 'DELETED', ?, null, ?, ?)
            """, studyFk, studyUid, deletedBy, reason);

        return ResponseEntity.ok(Map.of(
            "studyUid", studyUid,
            "filesDeleted", filesDeleted,
            "filesFailed", filesFailed,
            "dbRecordsDeleted", instances.size()
        ));
    }

    // ─── MERGE PATIENTS ───────────────────────────────────────────────────

    /**
     * POST /api/v1/{tenant}/admin/patients/merge
     * Body: { "sourcePatientId": 123, "targetPatientId": 456, "mergedBy": "admin", "reason": "..." }
     *
     * Moves all studies from sourcePatient → targetPatient.
     * Deletes sourcePatient.
     * Updates study public_ids (since patient_id changes).
     */
    @PostMapping("/patients/merge")
    @Transactional
    public ResponseEntity<Map<String, Object>> mergePatients(
            @RequestBody MergeRequest request) {

        // 1. Verify both patients exist
        Map<String, Object> source = jdbcTemplate.queryForMap(
            "SELECT * FROM patient WHERE id = ?", request.sourcePatientId()
        );
        Map<String, Object> target = jdbcTemplate.queryForMap(
            "SELECT * FROM patient WHERE id = ?", request.targetPatientId()
        );

        String targetPid = (String) target.get("patient_id");

        // 2. Move all studies from source → target
        List<Map<String, Object>> studies = jdbcTemplate.queryForList(
            "SELECT id, study_instance_uid FROM study WHERE patient_fk = ?", request.sourcePatientId()
        );

        for (Map<String, Object> study : studies) {
            String studyUid = (String) study.get("study_instance_uid");
            String newPublicId = sha1(targetPid + "|" + studyUid);

            jdbcTemplate.update("""
                UPDATE study SET
                    patient_fk = ?,
                    patient_id = ?,
                    public_id = ?,
                    updated_at = now()
                WHERE id = ?
                """, request.targetPatientId(), targetPid, newPublicId, study.get("id"));
        }

        // 3. Update target patient counters
        jdbcTemplate.update("""
            UPDATE patient SET
                num_studies = (SELECT COUNT(*) FROM study WHERE patient_fk = ?)
            WHERE id = ?
            """, request.targetPatientId(), request.targetPatientId());

        // 4. Delete source patient
        jdbcTemplate.update("DELETE FROM patient WHERE id = ?", request.sourcePatientId());

        // 5. Audit
        jdbcTemplate.update("""
            INSERT INTO audit_log (entity_type, entity_id, field_name, old_value, new_value, changed_by, reason)
            VALUES ('PATIENT', ?, 'MERGED_INTO', ?, ?, ?, ?)
            """, request.sourcePatientId(),
                String.valueOf(request.sourcePatientId()),
                String.valueOf(request.targetPatientId()),
                request.mergedBy(), request.reason());

        return ResponseEntity.ok(Map.of(
            "studiesMoved", studies.size(),
            "sourcePatientDeleted", request.sourcePatientId(),
            "targetPatient", request.targetPatientId()
        ));
    }

    // ─── STATS ────────────────────────────────────────────────────────────

    /**
     * GET /api/v1/{tenant}/admin/stats
     * Returns tenant statistics: study count, instance count, disk usage, etc.
     */
    @GetMapping("/stats")
    public ResponseEntity<Map<String, Object>> getStats() {
        Map<String, Object> stats = new LinkedHashMap<>();

        stats.put("patients", jdbcTemplate.queryForObject("SELECT COUNT(*) FROM patient", Long.class));
        stats.put("studies", jdbcTemplate.queryForObject("SELECT COUNT(*) FROM study", Long.class));
        stats.put("series", jdbcTemplate.queryForObject("SELECT COUNT(*) FROM series", Long.class));
        stats.put("instances", jdbcTemplate.queryForObject("SELECT COUNT(*) FROM instance", Long.class));
        stats.put("totalStorageBytes", jdbcTemplate.queryForObject("SELECT COALESCE(SUM(study_size), 0) FROM study", Long.class));
        stats.put("provisionalPatients", jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM patient WHERE is_provisional = true", Long.class
        ));
        stats.put("pendingCorrections", jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM file_correction_task WHERE status != 'COMPLETED'", Long.class
        ));

        return ResponseEntity.ok(stats);
    }

    private String sha1(String input) { /* same as BulkInsertRepository */ }

    public record MergeRequest(long sourcePatientId, long targetPatientId, String mergedBy, String reason) {}
}
```

### 2. `src/main/java/com/spax/admin/SystemInfoController.java`

```java
@RestController
@RequestMapping("/api/v1/admin/system")
public class SystemInfoController {

    @Autowired private VolumeManager volumeManager;
    @Autowired private JdbcTemplate jdbcTemplate;

    /**
     * GET /api/v1/admin/system/info
     * Returns global system info (not tenant-specific).
     */
    @GetMapping("/info")
    public ResponseEntity<Map<String, Object>> getSystemInfo() {
        Map<String, Object> info = new LinkedHashMap<>();

        // Active tenants
        info.put("tenants", jdbcTemplate.queryForList(
            "SELECT code, name, active FROM public.tenant ORDER BY code",
            Map.class
        ));

        // Storage volumes with usage
        List<Map<String, Object>> volumes = jdbcTemplate.queryForList(
            "SELECT id, code, provider_type, tier, status, priority, total_bytes, used_bytes FROM public.storage_volume"
        );

        // Enrich LOCAL volumes with real disk usage
        for (Map<String, Object> vol : volumes) {
            if ("LOCAL".equals(vol.get("provider_type"))) {
                try {
                    int volId = (int) vol.get("id");
                    StorageVolume volume = volumeManager.getVolume(volId);
                    if (volumeManager.getProvider(volId) instanceof LocalStorageProvider local) {
                        vol.put("availableBytes", local.getAvailableBytes());
                        vol.put("totalDiskBytes", local.getTotalBytes());
                    }
                } catch (Exception e) { /* skip */ }
            }
        }
        info.put("volumes", volumes);

        // Pending migrations
        info.put("pendingMigrations", jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM public.migration_task WHERE status IN ('PENDING','IN_PROGRESS')", Long.class
        ));

        // DB size
        info.put("dbVersion", jdbcTemplate.queryForObject("SELECT version()", String.class));

        return ResponseEntity.ok(info);
    }
}
```

## Lưu ý quan trọng
- Delete study: files TRƯỚC, DB SAU — nếu storage fail, file vẫn còn, có thể retry
- Nếu DB delete thành công nhưng storage delete fail → "orphan files" — acceptable, log warning
- Merge patients: phải recalculate study `public_id` vì công thức là `SHA1(patient_id | study_uid)`
- `@Transactional` trên delete/merge — nếu có lỗi giữa chừng, DB rollback
- Không dùng CASCADE DELETE trong DB schema (instance là partitioned table) — phải delete explicitly

## Kiểm tra thành công
- `DELETE /admin/studies/{uid}` → files bị xóa khỏi disk, DB không còn records
- `POST /admin/patients/merge` → studies chuyển sang target patient, source bị xóa
- `GET /admin/stats` → counts chính xác
- `GET /admin/system/info` → volumes với disk usage
