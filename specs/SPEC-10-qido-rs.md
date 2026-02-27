# SPEC-10: DICOMWeb QIDO-RS (Query)

## Mục tiêu
Implement QIDO-RS endpoints cho OHIF Viewer. OHIF dùng QIDO để list studies, series, instances. Response là DICOM JSON format (PS3.18).

## Dependencies
- SPEC-02 (database schema)
- SPEC-03 (TenantContext)
- SPEC-09 (JPA Repositories — StudyRepository, SeriesRepository, InstanceRepository)

## Files cần tạo

### 1. `src/main/java/com/spax/dicomweb/QueryBuilder.java`

Dynamic SQL builder từ QIDO query params:

```java
@Component
public class QueryBuilder {

    /**
     * Build WHERE clause and params for QIDO-RS study search.
     *
     * QIDO params → SQL columns mapping:
     *   PatientName      → patient_name (via JOIN or denormalized)
     *   PatientID        → patient_id
     *   StudyDate        → study_date (format: YYYYMMDD or YYYYMMDD-YYYYMMDD for range)
     *   StudyDescription → study_description
     *   AccessionNumber  → accession_number
     *   Modality         → modalities_in_study (series level)
     *   StudyInstanceUID → study_instance_uid
     */
    public QueryResult buildStudyQuery(Map<String, String> params) {
        StringBuilder sql = new StringBuilder("SELECT s.* FROM study s WHERE 1=1");
        List<Object> args = new ArrayList<>();

        String patientName = params.get("PatientName");
        if (patientName != null && !patientName.isBlank()) {
            // Wildcard: DOE* → LIKE 'DOE%'
            sql.append(" AND s.patient_id IN (SELECT p.id FROM patient p WHERE p.patient_name ILIKE ?)");
            args.add(wildcardToSql(patientName));
        }

        String patientId = params.get("PatientID");
        if (patientId != null && !patientId.isBlank()) {
            sql.append(" AND s.patient_id ILIKE ?");
            args.add(wildcardToSql(patientId));
        }

        String studyDate = params.get("StudyDate");
        if (studyDate != null && !studyDate.isBlank()) {
            if (studyDate.contains("-")) {
                // Range: 20240101-20240131
                String[] parts = studyDate.split("-");
                sql.append(" AND s.study_date BETWEEN ? AND ?");
                args.add(parts[0]);
                args.add(parts[1]);
            } else {
                sql.append(" AND s.study_date = ?");
                args.add(studyDate);
            }
        }

        String accession = params.get("AccessionNumber");
        if (accession != null && !accession.isBlank()) {
            sql.append(" AND s.accession_number ILIKE ?");
            args.add(wildcardToSql(accession));
        }

        String studyDesc = params.get("StudyDescription");
        if (studyDesc != null && !studyDesc.isBlank()) {
            sql.append(" AND s.study_description ILIKE ?");
            args.add(wildcardToSql(studyDesc));
        }

        String studyUid = params.get("StudyInstanceUID");
        if (studyUid != null && !studyUid.isBlank()) {
            sql.append(" AND s.study_instance_uid = ?");
            args.add(studyUid);
        }

        // Pagination
        int limit = Integer.parseInt(params.getOrDefault("limit", "100"));
        int offset = Integer.parseInt(params.getOrDefault("offset", "0"));
        sql.append(" ORDER BY s.created_at DESC LIMIT ? OFFSET ?");
        args.add(Math.min(limit, 1000));  // cap at 1000
        args.add(offset);

        return new QueryResult(sql.toString(), args.toArray());
    }

    private String wildcardToSql(String value) {
        return value.replace("*", "%").replace("?", "_");
    }

    public record QueryResult(String sql, Object[] args) {}
}
```

### 2. `src/main/java/com/spax/dicomweb/DicomJsonBuilder.java`

Convert DB rows → DICOM JSON format (PS3.18):

```java
@Component
public class DicomJsonBuilder {

    /**
     * Build DICOM JSON for a study row.
     * OHIF expects format: { "XXXXXXXX": { "vr": "XX", "Value": [...] } }
     */
    public Map<String, Object> studyToJson(Study study) {
        Map<String, Object> json = new LinkedHashMap<>();

        // PatientID (0010,0020)
        addString(json, "00100020", "LO", study.getPatientId());

        // PatientName (0010,0010)
        addPersonName(json, "00100010", study.getPatientName());

        // StudyInstanceUID (0020,000D)
        addString(json, "0020000D", "UI", study.getStudyInstanceUid());

        // StudyDate (0008,0020)
        addString(json, "00080020", "DA", study.getStudyDate());

        // StudyTime (0008,0030)
        addString(json, "00080030", "TM", study.getStudyTime());

        // StudyDescription (0008,1030)
        addString(json, "00081030", "LO", study.getStudyDescription());

        // AccessionNumber (0008,0050)
        addString(json, "00080050", "SH", study.getAccessionNumber());

        // ReferringPhysicianName (0008,0090)
        addPersonName(json, "00080090", study.getReferringPhysician());

        // ModalitiesInStudy (0008,0061)
        addString(json, "00080061", "CS", study.getModalitiesInStudy());

        // NumberOfStudyRelatedSeries (0020,1206)
        addInt(json, "00201206", "IS", study.getNumSeries());

        // NumberOfStudyRelatedInstances (0020,1208)
        addInt(json, "00201208", "IS", study.getNumInstances());

        return json;
    }

    public Map<String, Object> seriesToJson(Series series) {
        Map<String, Object> json = new LinkedHashMap<>();
        addString(json, "0020000E", "UI", series.getSeriesInstanceUid());
        addString(json, "0020000D", "UI", series.getStudyInstanceUid());
        addString(json, "00080060", "CS", series.getModality());
        addInt(json, "00200011", "IS", series.getSeriesNumber());
        addString(json, "0008103E", "LO", series.getSeriesDescription());
        addString(json, "00180015", "CS", series.getBodyPart());
        addInt(json, "00201209", "IS", series.getNumInstances());
        return json;
    }

    public Map<String, Object> instanceToJson(Instance instance) {
        Map<String, Object> json = new LinkedHashMap<>();
        addString(json, "00080018", "UI", instance.getSopInstanceUid());
        addString(json, "00080016", "UI", instance.getSopClassUid());
        addInt(json, "00200013", "IS", instance.getInstanceNumber());
        addString(json, "00020010", "UI", instance.getTransferSyntaxUid());
        addInt(json, "00280008", "IS", instance.getNumFrames());
        return json;
    }

    // ─── Helpers ─────────────────────────────────────────────────────────────

    private void addString(Map<String, Object> json, String tag, String vr, String value) {
        if (value == null || value.isBlank()) {
            json.put(tag, Map.of("vr", vr));
        } else {
            json.put(tag, Map.of("vr", vr, "Value", List.of(value)));
        }
    }

    private void addPersonName(Map<String, Object> json, String tag, String name) {
        if (name == null || name.isBlank()) {
            json.put(tag, Map.of("vr", "PN"));
        } else {
            // DICOM PN format: "DOE^JOHN" → {"Alphabetic": "DOE^JOHN"}
            json.put(tag, Map.of("vr", "PN", "Value", List.of(Map.of("Alphabetic", name))));
        }
    }

    private void addInt(Map<String, Object> json, String tag, String vr, Integer value) {
        if (value == null) {
            json.put(tag, Map.of("vr", vr));
        } else {
            json.put(tag, Map.of("vr", vr, "Value", List.of(value)));
        }
    }
}
```

### 3. `src/main/java/com/spax/dicomweb/QidoController.java`

```java
@RestController
@RequestMapping("/dicomweb/{tenant}")
public class QidoController {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private QueryBuilder queryBuilder;
    @Autowired private DicomJsonBuilder jsonBuilder;
    @Autowired private SeriesRepository seriesRepository;
    @Autowired private InstanceRepository instanceRepository;

    /**
     * GET /dicomweb/{tenant}/studies
     * Query params: PatientName, PatientID, StudyDate, AccessionNumber, StudyDescription, StudyInstanceUID
     *               limit, offset
     * Response: application/dicom+json
     */
    @GetMapping(value = "/studies", produces = "application/dicom+json")
    public ResponseEntity<List<Map<String, Object>>> searchStudies(
            @PathVariable String tenant,
            @RequestParam Map<String, String> params) {

        QueryBuilder.QueryResult query = queryBuilder.buildStudyQuery(params);

        List<Study> studies = jdbcTemplate.query(
            query.sql(), query.args(),
            new StudyRowMapper()
        );

        // Update last_accessed_at for LAST_ACCESS_DAYS lifecycle condition
        if (!studies.isEmpty()) {
            List<Long> ids = studies.stream().map(Study::getId).toList();
            jdbcTemplate.update(
                "UPDATE study SET last_accessed_at = now() WHERE id = ANY(?)",
                (Object) ids.toArray(Long[]::new)
            );
        }

        List<Map<String, Object>> response = studies.stream()
            .map(jsonBuilder::studyToJson)
            .toList();

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom+json"))
            .body(response);
    }

    /**
     * GET /dicomweb/{tenant}/studies/{studyUid}/series
     */
    @GetMapping(value = "/studies/{studyUid}/series", produces = "application/dicom+json")
    public ResponseEntity<List<Map<String, Object>>> searchSeries(
            @PathVariable String tenant,
            @PathVariable String studyUid,
            @RequestParam(defaultValue = "100") int limit,
            @RequestParam(defaultValue = "0") int offset) {

        List<Series> series = jdbcTemplate.query(
            "SELECT * FROM series WHERE study_instance_uid = ? ORDER BY series_number LIMIT ? OFFSET ?",
            new SeriesRowMapper(),
            studyUid, limit, offset
        );

        List<Map<String, Object>> response = series.stream()
            .map(jsonBuilder::seriesToJson)
            .toList();

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom+json"))
            .body(response);
    }

    /**
     * GET /dicomweb/{tenant}/studies/{studyUid}/series/{seriesUid}/instances
     */
    @GetMapping(value = "/studies/{studyUid}/series/{seriesUid}/instances", produces = "application/dicom+json")
    public ResponseEntity<List<Map<String, Object>>> searchInstances(
            @PathVariable String tenant,
            @PathVariable String studyUid,
            @PathVariable String seriesUid,
            @RequestParam(defaultValue = "1000") int limit,
            @RequestParam(defaultValue = "0") int offset) {

        List<Instance> instances = jdbcTemplate.query(
            """
            SELECT * FROM instance
            WHERE study_instance_uid = ? AND series_instance_uid = ?
            ORDER BY instance_number
            LIMIT ? OFFSET ?
            """,
            new InstanceRowMapper(),
            studyUid, seriesUid, limit, offset
        );

        List<Map<String, Object>> response = instances.stream()
            .map(jsonBuilder::instanceToJson)
            .toList();

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom+json"))
            .body(response);
    }
}
```

**Note**: Cần implement `StudyRowMapper`, `SeriesRowMapper`, `InstanceRowMapper` — các `RowMapper<T>` class map SQL ResultSet → JPA entity.

## Lưu ý quan trọng
- Content-Type response: `application/dicom+json` (không phải `application/json`)
- OHIF cần exact format: tag phải uppercase 8 hex chars, Value là array
- PersonName format: `{"Alphabetic": "LAST^FIRST"}` (không phải plain string)
- Wildcard: `*` → `%`, `?` → `_` trong SQL LIKE
- `last_accessed_at` update: async fire-and-forget OK, không cần transaction với query
- Limit cap tại 1000 — tránh OOM với response quá lớn
- QIDO không trả pixel data — chỉ metadata

## Kiểm tra thành công
- `GET /dicomweb/test/studies` → list studies dạng DICOM JSON
- `GET /dicomweb/test/studies?PatientName=DOE*` → filter theo wildcard
- `GET /dicomweb/test/studies?StudyDate=20240101-20240131` → filter theo date range
- OHIF Viewer configured to use SPAX → studies hiện thị trong worklist
