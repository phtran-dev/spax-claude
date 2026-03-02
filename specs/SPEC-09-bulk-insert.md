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

        // 3. Upsert series (get/create series IDs + created_date for partition)
        Map<String, SeriesRef> seriesRefs = upsertSeries(items, studyIds);

        // 4. Dedup + insert instances (dùng series created_date làm partition key)
        insertInstances(items, seriesRefs);

        // 5. Update counters (num_instances, num_series, num_studies, sizes)
        updateCounters(items, studyIds, seriesRefs);
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

    /**
     * Trả SeriesRef (id + createdDate) thay vì chỉ id.
     * createdDate dùng làm partition key cho instance table:
     * - New series: created_at = now() → createdDate = today
     * - Existing series: created_at = ngày đầu tiên ingest series này
     * → Mọi instances cùng series nằm cùng 1 partition → partition pruning hoạt động.
     */
    private Map<String, SeriesRef> upsertSeries(List<IndexedDicomMetadata> items, Map<String, Long> studyIds) {
        // Key for series dedup: (studyPublicId, seriesUid)
        Map<String, IndexedDicomMetadata> bySeries = new LinkedHashMap<>();
        for (IndexedDicomMetadata item : items) {
            DicomMetadata d = item.dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            String seriesKey = studyPublicId + "|" + d.seriesInstanceUid();
            bySeries.putIfAbsent(seriesKey, item);
        }

        Map<String, SeriesRef> result = new HashMap<>();

        for (Map.Entry<String, IndexedDicomMetadata> entry : bySeries.entrySet()) {
            DicomMetadata d = entry.getValue().dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            long studyFk = studyIds.get(studyPublicId);

            // RETURNING id, created_at — lấy cả created_at cho partition key
            Map<String, Object> row = jdbcTemplate.queryForMap("""
                INSERT INTO series
                  (series_instance_uid, modality, series_number, series_description,
                   body_part, institution, station_name, sending_aet, study_fk, study_instance_uid)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT (study_fk, series_instance_uid) DO UPDATE SET
                    modality = EXCLUDED.modality,
                    series_description = COALESCE(EXCLUDED.series_description, series.series_description)
                RETURNING id, created_at::date AS created_date
                """,
                d.seriesInstanceUid(), d.modality(), d.seriesNumber(), d.seriesDescription(),
                d.bodyPart(), d.institution(), d.stationName(), d.sendingAet(),
                studyFk, d.studyInstanceUid()
            );

            result.put(entry.getKey(), new SeriesRef(
                ((Number) row.get("id")).longValue(),
                ((java.sql.Date) row.get("created_date")).toLocalDate()
            ));
        }

        return result;
    }

    private record SeriesRef(long id, LocalDate createdDate) {}

    // ─── INSTANCE (application-level dedup) ───────────────────────────────────

    private void insertInstances(List<IndexedDicomMetadata> items, Map<String, SeriesRef> seriesRefs) {
        // Application-level dedup: query existing SOPUIDs for each series
        // Cannot use DB-level UNIQUE because PostgreSQL partitioned table requires partition key in UNIQUE index

        // Group items by series key
        Map<String, List<IndexedDicomMetadata>> bySeriesKey = new LinkedHashMap<>();
        for (IndexedDicomMetadata item : items) {
            DicomMetadata d = item.dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            String seriesKey = studyPublicId + "|" + d.seriesInstanceUid();
            bySeriesKey.computeIfAbsent(seriesKey, k -> new ArrayList<>()).add(item);
        }

        for (Map.Entry<String, List<IndexedDicomMetadata>> entry : bySeriesKey.entrySet()) {
            SeriesRef ref = seriesRefs.get(entry.getKey());
            long seriesFk = ref.id();
            java.sql.Date createdDate = java.sql.Date.valueOf(ref.createdDate());
            List<IndexedDicomMetadata> seriesItems = entry.getValue();

            // Query existing SOPUIDs for this series
            // Dùng created_date để partition pruning — chỉ scan 1 partition
            List<String> existingSops = jdbcTemplate.queryForList(
                "SELECT sop_instance_uid FROM instance WHERE series_fk = ? AND created_date = ?",
                String.class,
                seriesFk, createdDate
            );
            Set<String> existingSet = new HashSet<>(existingSops);

            // Filter out duplicates
            List<IndexedDicomMetadata> toInsert = seriesItems.stream()
                .filter(item -> !existingSet.contains(item.dicom().sopInstanceUid()))
                .toList();

            if (toInsert.isEmpty()) continue;

            // Batch insert — created_date = series.created_at::date (không phải CURRENT_DATE)
            jdbcTemplate.batchUpdate("""
                INSERT INTO instance
                  (sop_instance_uid, sop_class_uid, instance_number, transfer_syntax_uid,
                   num_frames, file_size, volume_id, storage_path,
                   series_fk, series_instance_uid, study_instance_uid, created_date)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
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
                    ps.setDate(12, createdDate);  // partition key = series created date
                }
            );
        }
    }

    // ─── COUNTERS ─────────────────────────────────────────────────────────────

    private void updateCounters(List<IndexedDicomMetadata> items,
                                 Map<String, Long> studyIds, Map<String, SeriesRef> seriesRefs) {
        // Update series counts
        Set<Long> affectedSeriesFks = new HashSet<>();
        Set<Long> affectedStudyFks = new HashSet<>();

        for (IndexedDicomMetadata item : items) {
            DicomMetadata d = item.dicom();
            String studyPublicId = sha1(d.patientId() + "|" + d.studyInstanceUid());
            String seriesKey = studyPublicId + "|" + d.seriesInstanceUid();
            affectedSeriesFks.add(seriesRefs.get(seriesKey).id());
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
            return sb.toString();  // 40 hex chars (SHA-1) — column is VARCHAR(128) for future algorithm change
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

    /**
     * QUAN TRỌNG: trả List, không phải Optional.
     * study_instance_uid KHÔNG phải unique key trong SPAX.
     * Unique key là public_id = SHA1(patientId + "|" + studyUid).
     * Hai patient khác nhau có thể có cùng StudyInstanceUID
     * (máy clone, firmware lỗi, merge dataset).
     */
    List<Study> findByStudyInstanceUid(String uid);
}
```

`src/main/java/com/spax/repository/SeriesRepository.java`:
```java
@Repository
public interface SeriesRepository extends JpaRepository<Series, Long> {
    List<Series> findByStudyFk(Long studyFk);
    Optional<Series> findByStudyFkAndSeriesInstanceUid(Long studyFk, String seriesUid);

    /**
     * Trả List — series_instance_uid không unique toàn cục,
     * chỉ unique trong 1 study (UNIQUE(study_fk, series_instance_uid)).
     * Nhiều studies khác nhau có thể chứa series cùng UID.
     */
    List<Series> findBySeriesInstanceUid(String uid);
}
```

`src/main/java/com/spax/repository/InstanceRepository.java`:
```java
@Repository
public interface InstanceRepository extends JpaRepository<Instance, Long> {
    List<Instance> findByStudyInstanceUid(String studyUid);
    List<Instance> findBySeriesFk(Long seriesFk);

    /**
     * Trả List — sop_instance_uid không có DB-level UNIQUE constraint
     * (PostgreSQL partitioned table yêu cầu partition key trong unique index,
     * mà created_date + sop_uid không có ý nghĩa).
     * Dedup ở application level (BulkInsertRepository).
     * Trong thực tế thường chỉ có 1 row, nhưng không guarantee.
     */
    List<Instance> findBySopInstanceUid(String sopUid);
}
```

## Lưu ý quan trọng
- **UID không unique**: `study_instance_uid`, `series_instance_uid`, `sop_instance_uid` **không phải unique key** trong SPAX. Repository methods `findByXxxUid()` phải trả `List<T>`, không phải `Optional<T>`. Lý do: máy clone/firmware lỗi tạo UID trùng giữa patients. Unique key thực sự: `patient.public_id`, `study.public_id = SHA1(pid|studyUid)`, `series: UNIQUE(study_fk, series_uid)`, instance: app-level dedup.
- SHA-1 formula: `patient.public_id = SHA1(patientId)`, `study.public_id = SHA1(patientId + "|" + studyUid)`
- `"|"` là separator (pipe) — theo Orthanc convention
- **`instance.created_date` = `series.created_at::date`** (không dùng `CURRENT_DATE`). Mọi instances cùng series nằm cùng 1 partition → DICOMWeb query có thể partition prune bằng cách lấy `created_date` từ series table trước. Late arrival (re-send sau N tháng) vào partition cũ — chấp nhận được vì lifecycle quản lý theo study/series, không theo instance riêng lẻ.
- Application-level dedup cho instance: query `idx_instance_dedup` index trước khi INSERT, kèm `created_date` để partition prune
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
