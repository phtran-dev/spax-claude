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

Convert DB rows → DICOM JSON format (PS3.18 F.2).

Dùng dcm4che `Attributes` + `JSONWriter` thay vì hand-roll JSON:
- Build `Attributes` object từ DB columns bằng `setString(Tag.X, VR.Y, value)`
- Serialize bằng `JSONWriter.write(attrs)` → output chuẩn PS3.18 tự động
- PersonName format `{"Alphabetic": "DOE^JOHN"}` được `JSONWriter` xử lý đúng
- Null/empty values → `JSONWriter` tự output `{"vr":"XX"}` (không có `Value` key)
- Không cần custom helpers (addString, addPersonName, addInt) — dcm4che lo hết

**Tham khảo**: dcm4chee-arc-light `QidoRS.java` + `StudyQuery.java` — build `Attributes` từ DB tuple rồi serialize bằng `JSONWriter`.

```java
@Component
public class DicomJsonBuilder {

    /**
     * Build Attributes cho 1 study row, rồi serialize → DICOM JSON.
     *
     * Approach giống dcm4chee-arc-light:
     * - DB columns → dcm4che Attributes (setString/setInt với Tag constants + VR)
     * - Attributes → JSON via JSONWriter (streaming, chuẩn PS3.18 F.2)
     *
     * JSONWriter tự xử lý:
     * - Tag format: uppercase 8-hex (e.g., "00100020")
     * - VR + Value array wrapper
     * - PersonName → {"Alphabetic": "LAST^FIRST"}
     * - Null values → {"vr": "XX"} (no Value key)
     * - IS (Integer String) → number trong JSON (không phải string)
     */
    public void writeStudyJson(Study study, JsonGenerator gen) {
        Attributes attrs = new Attributes();

        attrs.setString(Tag.PatientID, VR.LO, study.getPatientId());
        attrs.setString(Tag.PatientName, VR.PN, study.getPatientName());
        attrs.setString(Tag.StudyInstanceUID, VR.UI, study.getStudyInstanceUid());
        attrs.setString(Tag.StudyDate, VR.DA, study.getStudyDate());
        attrs.setString(Tag.StudyTime, VR.TM, study.getStudyTime());
        attrs.setString(Tag.StudyDescription, VR.LO, study.getStudyDescription());
        attrs.setString(Tag.AccessionNumber, VR.SH, study.getAccessionNumber());
        attrs.setString(Tag.ReferringPhysicianName, VR.PN, study.getReferringPhysician());
        attrs.setString(Tag.ModalitiesInStudy, VR.CS, study.getModalitiesInStudy());

        if (study.getNumSeries() != null)
            attrs.setInt(Tag.NumberOfStudyRelatedSeries, VR.IS, study.getNumSeries());
        if (study.getNumInstances() != null)
            attrs.setInt(Tag.NumberOfStudyRelatedInstances, VR.IS, study.getNumInstances());

        new JSONWriter(gen).write(attrs);
    }

    public void writeSeriesJson(Series series, JsonGenerator gen) {
        Attributes attrs = new Attributes();

        attrs.setString(Tag.SeriesInstanceUID, VR.UI, series.getSeriesInstanceUid());
        attrs.setString(Tag.StudyInstanceUID, VR.UI, series.getStudyInstanceUid());
        attrs.setString(Tag.Modality, VR.CS, series.getModality());
        attrs.setString(Tag.SeriesDescription, VR.LO, series.getSeriesDescription());
        attrs.setString(Tag.BodyPartExamined, VR.CS, series.getBodyPart());

        if (series.getSeriesNumber() != null)
            attrs.setInt(Tag.SeriesNumber, VR.IS, series.getSeriesNumber());
        if (series.getNumInstances() != null)
            attrs.setInt(Tag.NumberOfSeriesRelatedInstances, VR.IS, series.getNumInstances());

        new JSONWriter(gen).write(attrs);
    }

    public void writeInstanceJson(Instance instance, JsonGenerator gen) {
        Attributes attrs = new Attributes();

        attrs.setString(Tag.SOPInstanceUID, VR.UI, instance.getSopInstanceUid());
        attrs.setString(Tag.SOPClassUID, VR.UI, instance.getSopClassUid());
        attrs.setString(Tag.TransferSyntaxUID, VR.UI, instance.getTransferSyntaxUid());

        if (instance.getInstanceNumber() != null)
            attrs.setInt(Tag.InstanceNumber, VR.IS, instance.getInstanceNumber());
        if (instance.getNumFrames() != null)
            attrs.setInt(Tag.NumberOfFrames, VR.IS, instance.getNumFrames());

        new JSONWriter(gen).write(attrs);
    }
}
```

**Imports cần thiết** (dcm4che-core + dcm4che-json):
```java
import org.dcm4che3.data.Attributes;
import org.dcm4che3.data.Tag;
import org.dcm4che3.data.VR;
import org.dcm4che3.json.JSONWriter;
import jakarta.json.stream.JsonGenerator;
```

**So sánh với bản cũ (hand-rolled)**:

| Aspect | Hand-rolled (cũ) | dcm4che JSONWriter (mới) |
|--------|-------------------|--------------------------|
| PersonName format | Phải tự code `{"Alphabetic": "..."}` | `JSONWriter` tự xử lý |
| VR encoding | Tự map string tag → VR | Dùng `Tag` constants + `VR` enum |
| Null handling | 3 if/else per type | `Attributes.setString(null)` = skip, JSONWriter auto |
| PS3.18 compliance | Phải tự kiểm tra | Guaranteed bởi dcm4che |
| IS (Integer String) | Tự decide int vs string | JSONWriter output theo chuẩn |
| Thêm field mới | Copy-paste addString/addInt | 1 dòng `attrs.setString(Tag.X, VR.Y, val)` |
| Output type | `Map<String, Object>` | Streaming `JsonGenerator` (không buffer) |

### 3. `src/main/java/com/spax/dicomweb/QidoController.java`

Dùng `StreamingResponseBody` + `JsonGenerator` để stream DICOM JSON array trực tiếp ra response,
không buffer toàn bộ JSON vào RAM. Phù hợp cho study list lớn (100+ studies).

```java
@RestController
@RequestMapping("/dicomweb/{tenant}")
public class QidoController {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private QueryBuilder queryBuilder;
    @Autowired private DicomJsonBuilder jsonBuilder;

    /**
     * GET /dicomweb/{tenant}/studies
     * Query params: PatientName, PatientID, StudyDate, AccessionNumber, StudyDescription, StudyInstanceUID
     *               limit, offset
     * Response: application/dicom+json
     */
    @GetMapping(value = "/studies", produces = "application/dicom+json")
    public ResponseEntity<StreamingResponseBody> searchStudies(
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

        StreamingResponseBody body = outputStream -> {
            JsonGenerator gen = Json.createGenerator(outputStream);
            gen.writeStartArray();
            for (Study study : studies) {
                jsonBuilder.writeStudyJson(study, gen);
            }
            gen.writeEnd();
            gen.flush();
        };

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom+json"))
            .body(body);
    }

    /**
     * GET /dicomweb/{tenant}/studies/{studyUid}/series
     */
    @GetMapping(value = "/studies/{studyUid}/series", produces = "application/dicom+json")
    public ResponseEntity<StreamingResponseBody> searchSeries(
            @PathVariable String tenant,
            @PathVariable String studyUid,
            @RequestParam(defaultValue = "100") int limit,
            @RequestParam(defaultValue = "0") int offset) {

        List<Series> seriesList = jdbcTemplate.query(
            "SELECT * FROM series WHERE study_instance_uid = ? ORDER BY series_number LIMIT ? OFFSET ?",
            new SeriesRowMapper(),
            studyUid, limit, offset
        );

        StreamingResponseBody body = outputStream -> {
            JsonGenerator gen = Json.createGenerator(outputStream);
            gen.writeStartArray();
            for (Series series : seriesList) {
                jsonBuilder.writeSeriesJson(series, gen);
            }
            gen.writeEnd();
            gen.flush();
        };

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom+json"))
            .body(body);
    }

    /**
     * GET /dicomweb/{tenant}/studies/{studyUid}/series/{seriesUid}/instances
     */
    @GetMapping(value = "/studies/{studyUid}/series/{seriesUid}/instances", produces = "application/dicom+json")
    public ResponseEntity<StreamingResponseBody> searchInstances(
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

        StreamingResponseBody body = outputStream -> {
            JsonGenerator gen = Json.createGenerator(outputStream);
            gen.writeStartArray();
            for (Instance instance : instances) {
                jsonBuilder.writeInstanceJson(instance, gen);
            }
            gen.writeEnd();
            gen.flush();
        };

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("application/dicom+json"))
            .body(body);
    }
}
```

**Note**: Cần implement `StudyRowMapper`, `SeriesRowMapper`, `InstanceRowMapper` — các `RowMapper<T>` class map SQL ResultSet → JPA entity.

## Maven Dependencies

dcm4che-json (cho `JSONWriter`):
```xml
<dependency>
    <groupId>org.dcm4che</groupId>
    <artifactId>dcm4che-json</artifactId>
    <version>${dcm4che.version}</version>
</dependency>
```

> `dcm4che-core` (cho `Attributes`, `Tag`, `VR`) — đã có từ SPEC-07 (DicomParser).

> Jakarta JSON API (`jakarta.json.stream.JsonGenerator`) — dcm4che-json dùng Jakarta JSON-P, cần thêm implementation nếu chưa có:
> ```xml
> <dependency>
>     <groupId>org.glassfish</groupId>
>     <artifactId>jakarta.json</artifactId>
>     <version>2.0.1</version>
> </dependency>
> ```

## Lưu ý quan trọng
- Content-Type response: `application/dicom+json` (không phải `application/json`)
- **dcm4che JSONWriter** tự đảm bảo PS3.18 F.2 compliance:
  - Tag format: uppercase 8-hex chars (e.g., `"00100020"`)
  - Value luôn là array: `{"vr":"LO","Value":["DOE^JOHN"]}`
  - PersonName: `{"vr":"PN","Value":[{"Alphabetic":"DOE^JOHN"}]}`
  - Null/empty → `{"vr":"LO"}` (no Value key)
  - IS (Integer String) → output đúng kiểu number
- **Streaming**: `JsonGenerator` ghi trực tiếp ra `OutputStream` — không buffer JSON vào RAM
- Wildcard: `*` → `%`, `?` → `_` trong SQL LIKE
- `last_accessed_at` update: async fire-and-forget OK, không cần transaction với query
- Limit cap tại 1000 — tránh OOM với response quá lớn
- QIDO không trả pixel data — chỉ metadata
- **Khác với `SeriesMetadataBuilder`** (SPEC-11): `DicomJsonBuilder` build JSON từ DB columns (indexed metadata), còn `SeriesMetadataBuilder` đọc full DICOM file → build complete instance metadata JSON
- **UID không unique** (xem SPEC-09): `study_instance_uid`, `series_instance_uid` không phải unique key. QIDO search query (`searchStudies`, `searchSeries`) trả `List` nên không bị ảnh hưởng — kết quả đã bao gồm tất cả rows trùng UID. WADO-RS endpoints (`/studies/{uid}/series`, `/studies/{uid}/series/{uid}/instances`) cũng query bằng `WHERE study_instance_uid = ?` trả List — nếu có collision thì kết quả gộp chung (chấp nhận được vì OHIF navigate từ QIDO worklist nên context đúng)

## Kiểm tra thành công
- `GET /dicomweb/test/studies` → list studies dạng DICOM JSON
- `GET /dicomweb/test/studies?PatientName=DOE*` → filter theo wildcard
- `GET /dicomweb/test/studies?StudyDate=20240101-20240131` → filter theo date range
- OHIF Viewer configured to use SPAX → studies hiện thị trong worklist
