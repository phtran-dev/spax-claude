# SPEC-09: BulkInsertRepository (Application-level Dedup + Batch Upsert)

## Mục tiêu
Implement raw JDBC batch upsert cho ingest pipeline. Không dùng JPA/Hibernate cho path này (performance critical). Handle deduplication ở application level cho instance table (PostgreSQL không cho UNIQUE index trên partitioned table không có partition key).

## Dependencies
- SPEC-02 (database schema — patient/study/series/instance structure)
- SPEC-03 (TenantContext — biết schema hiện tại)
- SPEC-07 (DicomMetadata)

## Files cần tạo

### 1. `src/main/java/com/spax/repository/BulkInsertRepository.java`

```java
@Repository
public class BulkInsertRepository {

    @Autowired private JdbcTemplate jdbcTemplate;

    /**
     * Batch upsert a list of parsed+stored DICOM files.
     * All operations in single transaction.
     * Handles dedup at application level for instances.
     *
     * @param items list of IndexedDicomMetadata (parsed metadata + storage info)
     */
    @Transactional
    public void batchUpsert(List<IndexedDicomMetadata> items) {
        // Group by hierarchy (patient → study → series → instances)
        // Use LinkedHashMap to preserve insertion order

        // 1. Upsert patients (get/create patient IDs)
        Map<String, Long> patientIds = upsertPatients(items);

        // 2. Upsert studies (get/create study IDs)
        Map<String, Long> studyIds = upsertStudies(items, patientIds);

        // 3. Upsert series (get/create series IDs)
        Map<String, Long> seriesIds = upsertSeries(items, studyIds);

        // 4. Dedup + insert instances
        insertInstances(items, seriesIds);

        // 5. Update counters (num_instances, num_series, num_studies, sizes)
        updateCounters(items, studyIds, seriesIds);
    }

    // ─── PATIENT ─────────────────────────────────────────────────────────────

    private Map<String, Long> upsertPatients(List<IndexedDicomMetadata> items) {
        // Deduplicate by public_id = SHA1(patient_id)
        Map<String, IndexedDicomMetadata> byPublicId = new LinkedHashMap<>();
        for (IndexedDicomMetadata item : items) {
            String publicId = sha1(item.dicom().patientId());
            byPublicId.putIfAbsent(publicId, item);
        }

        Map<String, Long> result = new HashMap<>();

        for (Map.Entry<String, IndexedDicomMetadata> entry : byPublicId.entrySet()) {
            String publicId = entry.getKey();
            DicomMetadata d = entry.getValue().dicom();

            // Determine is_provisional: PID is synthetic OR name mismatch with existing
            boolean isProvisional = d.patientId().startsWith("NOPID_");

            Long id = jdbcTemplate.queryForObject("""
                INSERT INTO patient (public_id, patient_id, patient_name, birth_date, sex, is_provisional)
                VALUES (?, ?, ?, ?, ?, ?)
                ON CONFLICT (public_id) DO UPDATE SET
                    patient_name = COALESCE(EXCLUDED.patient_name, patient.patient_name),
                    updated_at = now()
                RETURNING id
                """,
                Long.class,
                publicId, d.patientId(), d.patientName(),
                parseBirthDate(d.birthDate()), d.sex(), isProvisional
            );

            result.put(publicId, id);
        }

        return result;
    }

    // ─── STUDY ────────────────────────────────────────────────────────────────

    private Map<String, Long> upsertStudies(List<IndexedDicomMetadata> items, Map<String, Long> patientIds) {
        Map<String, IndexedDicomMetadata> byPublicId = new LinkedHashMap<>();
        for (IndexedDicomMetadata item : items) {
            String pid = item.dicom().patientId();
            String studyPublicId = sha1(pid + "|" + item.dicom().studyInstanceUid());
            byPublicId.putIfAbsent(studyPublicId, item);
        }

        Map<String, Long> result = new HashMap<>();

        for (Map.Entry<String, IndexedDicomMetadata> entry : byPublicId.entrySet()) {
            String publicId = entry.getKey();
            DicomMetadata d = entry.getValue().dicom();
            long patientFk = patientIds.get(sha1(d.patientId()));

            Long id = jdbcTemplate.queryForObject("""
                INSERT INTO study
                  (public_id, study_instance_uid, study_date, study_time, study_description,
                   accession_number, referring_physician, patient_fk, patient_id)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT (public_id) DO UPDATE SET
                    study_description = COALESCE(EXCLUDED.study_description, study.study_description),
                    accession_number  = COALESCE(EXCLUDED.accession_number, study.accession_number),
                    updated_at = now()
                RETURNING id
                """,
                Long.class,
                publicId, d.studyInstanceUid(), d.studyDate(), d.studyTime(), d.studyDescription(),
                d.accessionNumber(), d.referringPhysician(), patientFk, d.patientId()
            );

            result.put(publicId, id);
        }

        return result;
    }

    // ─── SERIES ───────────────────────────────────────────────────────────────

    private Map<String, Long> upsertSeries(List<IndexedDicomMetadata> items, Map<String, Long> studyIds) {
        // Key for series dedup: (studyPublicId, seriesUid)
        Map<String, IndexedDicomMetadata> bySeries = new LinkedHashMap<>();
        for (IndexedDicomMetadata item : items) {
            DicomMetadata d = item.dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            String seriesKey = studyPublicId + "|" + d.seriesInstanceUid();
            bySeries.putIfAbsent(seriesKey, item);
        }

        Map<String, Long> result = new HashMap<>();

        for (Map.Entry<String, IndexedDicomMetadata> entry : bySeries.entrySet()) {
            DicomMetadata d = entry.getValue().dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            long studyFk = studyIds.get(studyPublicId);

            Long id = jdbcTemplate.queryForObject("""
                INSERT INTO series
                  (series_instance_uid, modality, series_number, series_description,
                   body_part, institution, station_name, sending_aet, study_fk, study_instance_uid)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT (study_fk, series_instance_uid) DO UPDATE SET
                    modality = EXCLUDED.modality,
                    series_description = COALESCE(EXCLUDED.series_description, series.series_description)
                RETURNING id
                """,
                Long.class,
                d.seriesInstanceUid(), d.modality(), d.seriesNumber(), d.seriesDescription(),
                d.bodyPart(), d.institution(), d.stationName(), d.sendingAet(),
                studyFk, d.studyInstanceUid()
            );

            result.put(entry.getKey(), id);
        }

        return result;
    }

    // ─── INSTANCE (application-level dedup) ───────────────────────────────────

    private void insertInstances(List<IndexedDicomMetadata> items, Map<String, Long> seriesIds) {
        // Application-level dedup: query existing SOPUIDs for each series
        // Cannot use DB-level UNIQUE because PostgreSQL partitioned table requires partition key in UNIQUE index

        // Group items by series
        Map<Long, List<IndexedDicomMetadata>> bySeriesFk = new LinkedHashMap<>();
        for (IndexedDicomMetadata item : items) {
            DicomMetadata d = item.dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            String seriesKey = studyPublicId + "|" + d.seriesInstanceUid();
            long seriesFk = seriesIds.get(seriesKey);
            bySeriesFk.computeIfAbsent(seriesFk, k -> new ArrayList<>()).add(item);
        }

        for (Map.Entry<Long, List<IndexedDicomMetadata>> entry : bySeriesFk.entrySet()) {
            long seriesFk = entry.getKey();
            List<IndexedDicomMetadata> seriesItems = entry.getValue();

            // Query existing SOPUIDs for this series
            List<String> existingSops = jdbcTemplate.queryForList(
                "SELECT sop_instance_uid FROM instance WHERE series_fk = ?",
                String.class,
                seriesFk
            );
            Set<String> existingSet = new HashSet<>(existingSops);

            // Filter out duplicates
            List<IndexedDicomMetadata> toInsert = seriesItems.stream()
                .filter(item -> !existingSet.contains(item.dicom().sopInstanceUid()))
                .toList();

            if (toInsert.isEmpty()) continue;

            // Batch insert
            jdbcTemplate.batchUpdate("""
                INSERT INTO instance
                  (sop_instance_uid, sop_class_uid, instance_number, transfer_syntax_uid,
                   num_frames, file_size, volume_id, storage_path,
                   series_fk, series_instance_uid, study_instance_uid, created_date)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, CURRENT_DATE)
                """,
                toInsert,
                toInsert.size(),
                (ps, item) -> {
                    DicomMetadata d = item.dicom();
                    ps.setString(1, d.sopInstanceUid());
                    ps.setString(2, d.sopClassUid());
                    ps.setInt(3, d.instanceNumber());
                    ps.setString(4, d.transferSyntaxUid());
                    ps.setInt(5, d.numFrames());
                    ps.setLong(6, item.fileSize());
                    ps.setInt(7, item.volumeId());
                    ps.setString(8, item.storagePath());
                    ps.setLong(9, seriesFk);
                    ps.setString(10, d.seriesInstanceUid());
                    ps.setString(11, d.studyInstanceUid());
                }
            );
        }
    }

    // ─── COUNTERS ─────────────────────────────────────────────────────────────

    private void updateCounters(List<IndexedDicomMetadata> items,
                                 Map<String, Long> studyIds, Map<String, Long> seriesIds) {
        // Update series counts
        Set<Long> affectedSeriesFks = new HashSet<>();
        Set<Long> affectedStudyFks = new HashSet<>();

        for (IndexedDicomMetadata item : items) {
            DicomMetadata d = item.dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            String seriesKey = studyPublicId + "|" + d.seriesInstanceUid();
            affectedSeriesFks.add(seriesIds.get(seriesKey));
            affectedStudyFks.add(studyIds.get(studyPublicId));
        }

        // Update series: num_instances + series_size
        for (Long seriesFk : affectedSeriesFks) {
            jdbcTemplate.update("""
                UPDATE series SET
                    num_instances = (SELECT COUNT(*) FROM instance WHERE series_fk = ?),
                    series_size   = (SELECT COALESCE(SUM(file_size), 0) FROM instance WHERE series_fk = ?)
                WHERE id = ?
                """, seriesFk, seriesFk, seriesFk);
        }

        // Update study: num_instances + num_series + study_size
        for (Long studyFk : affectedStudyFks) {
            jdbcTemplate.update("""
                UPDATE study SET
                    num_instances = (SELECT COUNT(*) FROM instance WHERE study_instance_uid =
                        (SELECT study_instance_uid FROM study WHERE id = ?)),
                    num_series    = (SELECT COUNT(*) FROM series WHERE study_fk = ?),
                    study_size    = (SELECT COALESCE(SUM(file_size), 0) FROM instance WHERE study_instance_uid =
                        (SELECT study_instance_uid FROM study WHERE id = ?)),
                    updated_at = now()
                WHERE id = ?
                """, studyFk, studyFk, studyFk, studyFk);
        }

        // Update patient: num_studies
        // (Less critical, can be eventually consistent)
        // TODO: Update patient.num_studies similarly
    }

    // ─── METADATA CACHE TRIGGER ──────────────────────────────────────────────

    /**
     * Trigger async rebuild của series metadata cache cho các series vừa được upsert.
     * Gọi sau batchUpsert() hoàn tất (không ở trong transaction).
     *
     * @param seriesUids   tập hợp seriesUid đã được xử lý trong batch
     * @param tenantCode   để biết đang ở tenant nào
     */
    public void triggerMetadataRebuild(Set<String> seriesUids, String tenantCode) {
        for (String seriesUid : seriesUids) {
            executor.submit(() -> {
                TenantContext.setTenantCode(tenantCode);
                try {
                    seriesMetadataBuilder.buildAndSave(seriesUid, tenantCode);
                } catch (Exception e) {
                    log.warn("Failed to rebuild series metadata for {}: {}", seriesUid, e.getMessage());
                    // Non-fatal: WADO-RS will fallback to reading DICOM files
                } finally {
                    TenantContext.clear();
                }
            });
        }
    }

    @Autowired
    @Qualifier("seriesMetadataBuilder")
    private DicomMetadataBuilder seriesMetadataBuilder;

    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    // ─── UTILS ───────────────────────────────────────────────────────────────

    private String sha1(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-1");
            byte[] hash = md.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder sb = new StringBuilder();
            for (byte b : hash) sb.append(String.format("%02x", b));
            return sb.toString();  // 40 hex chars = CHAR(40)
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);  // SHA-1 always available
        }
    }

    private java.sql.Date parseBirthDate(String dicomDate) {
        if (dicomDate == null || dicomDate.isBlank()) return null;
        try {
            LocalDate d = LocalDate.parse(dicomDate, DateTimeFormatter.ofPattern("yyyyMMdd"));
            return java.sql.Date.valueOf(d);
        } catch (Exception e) {
            return null;  // Invalid date format
        }
    }
}
```

### 2. Các JPA Repositories cơ bản (dùng cho non-ingest paths)

`src/main/java/com/spax/repository/PatientRepository.java`:
```java
@Repository
public interface PatientRepository extends JpaRepository<Patient, Long> {
    Optional<Patient> findByPublicId(String publicId);
    List<Patient> findByPatientId(String patientId);
}
```

`src/main/java/com/spax/repository/StudyRepository.java`:
```java
@Repository
public interface StudyRepository extends JpaRepository<Study, Long> {
    Optional<Study> findByPublicId(String publicId);
    List<Study> findByPatientFk(Long patientFk);
    Optional<Study> findByStudyInstanceUid(String uid);
}
```

`src/main/java/com/spax/repository/SeriesRepository.java`:
```java
@Repository
public interface SeriesRepository extends JpaRepository<Series, Long> {
    List<Series> findByStudyFk(Long studyFk);
    Optional<Series> findByStudyFkAndSeriesInstanceUid(Long studyFk, String seriesUid);
}
```

`src/main/java/com/spax/repository/InstanceRepository.java`:
```java
@Repository
public interface InstanceRepository extends JpaRepository<Instance, Long> {
    List<Instance> findByStudyInstanceUid(String studyUid);
    List<Instance> findBySeriesFk(Long seriesFk);
    Optional<Instance> findBySopInstanceUid(String sopUid);
}
```

## Lưu ý quan trọng
- SHA-1 formula: `patient.public_id = SHA1(patientId)`, `study.public_id = SHA1(patientId + "|" + studyUid)`
- `"|"` là separator (pipe) — theo Orthanc convention
- Application-level dedup cho instance: query `idx_instance_dedup` index trước khi INSERT
- `batchUpsert()` là `@Transactional` — rollback toàn bộ batch nếu có lỗi DB
- `triggerMetadataRebuild()` gọi **sau** `batchUpsert()` (ngoài transaction), async — không ảnh hưởng ingest throughput
- Counter update dùng `SELECT COUNT(*) / SUM()` subquery để recalculate chính xác thay vì increment
- `RETURNING id` trong INSERT/UPSERT dùng để lấy generated ID ngay lập tức (tránh query thêm)
- Metadata rebuild fail = non-fatal (log warn, WADO-RS sẽ fallback)

## Kiểm tra thành công
- Insert 200 instances → 1 DB roundtrip cho batch (verify via query logs)
- Resend cùng file → không tạo duplicate instance
- 2 bệnh nhân khác nhau cùng StudyUID → 2 studies riêng (khác public_id)
- num_instances, study_size được update chính xác
- Sau ingest → `series.metadata_path` được populate (async, có thể delay vài giây)
