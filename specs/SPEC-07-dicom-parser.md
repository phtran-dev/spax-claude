# SPEC-07: DICOM Parser

## Mục tiêu
Implement `DicomParser` để extract metadata từ DICOM file. Skip pixel data (không load vào RAM). Đây là component thuần kỹ thuật — input bytes, output metadata object.

## Dependencies
- SPEC-01 (project setup, dependency dcm4che-core)

## Files cần tạo

### 1. `src/main/java/com/spax/ingest/DicomParser.java`

```java
@Component
public class DicomParser {

    /**
     * Parse DICOM file metadata (skip pixel data).
     *
     * @param inputStream  raw DICOM file bytes
     * @return parsed metadata, or throws if not valid DICOM
     */
    public DicomMetadata parse(InputStream inputStream) throws IOException {
        DicomInputStream dis = new DicomInputStream(inputStream);
        dis.setIncludeBulkData(DicomInputStream.IncludeBulkData.NO);  // skip pixel data

        Attributes attrs = dis.readDataset();

        return extractMetadata(attrs, dis.getTransferSyntax());
    }

    /**
     * Parse from byte array.
     */
    public DicomMetadata parse(byte[] bytes) throws IOException {
        return parse(new ByteArrayInputStream(bytes));
    }

    private DicomMetadata extractMetadata(Attributes attrs, String transferSyntax) {
        // Patient level (tag references: PS3.6)
        String patientId   = attrs.getString(Tag.PatientID, "");
        String patientName = attrs.getString(Tag.PatientName, "");
        String birthDate   = attrs.getString(Tag.PatientBirthDate);
        String sex         = attrs.getString(Tag.PatientSex);

        // Study level
        String studyUid         = attrs.getString(Tag.StudyInstanceUID);
        String studyDate        = attrs.getString(Tag.StudyDate);
        String studyTime        = attrs.getString(Tag.StudyTime);
        String studyDesc        = attrs.getString(Tag.StudyDescription);
        String accessionNumber  = attrs.getString(Tag.AccessionNumber);
        String referringPhysician = attrs.getString(Tag.ReferringPhysicianName);

        // Series level
        String seriesUid    = attrs.getString(Tag.SeriesInstanceUID);
        String modality     = attrs.getString(Tag.Modality, "OT");  // OT = Other, fallback
        int    seriesNumber = attrs.getInt(Tag.SeriesNumber, 0);
        String seriesDesc   = attrs.getString(Tag.SeriesDescription);
        String bodyPart     = attrs.getString(Tag.BodyPartExamined);
        String institution  = attrs.getString(Tag.InstitutionName);
        String stationName  = attrs.getString(Tag.StationName);
        String sendingAet   = attrs.getString(Tag.SourceApplicationEntityTitle);

        // Instance level
        String sopUid       = attrs.getString(Tag.SOPInstanceUID);
        String sopClassUid  = attrs.getString(Tag.SOPClassUID);
        int    instanceNumber = attrs.getInt(Tag.InstanceNumber, 0);
        int    numFrames    = attrs.getInt(Tag.NumberOfFrames, 1);

        // Validate mandatory fields
        if (sopUid == null || sopUid.isBlank()) {
            throw new IllegalArgumentException("Missing SOPInstanceUID (0008,0018)");
        }
        if (studyUid == null || studyUid.isBlank()) {
            throw new IllegalArgumentException("Missing StudyInstanceUID (0020,000D)");
        }
        if (seriesUid == null || seriesUid.isBlank()) {
            throw new IllegalArgumentException("Missing SeriesInstanceUID (0020,000E)");
        }

        // Handle missing patient ID
        if (patientId == null || patientId.isBlank()) {
            patientId = "NOPID_" + studyUid.substring(0, Math.min(16, studyUid.length()));
        }

        return new DicomMetadata(
            patientId, patientName, birthDate, sex,
            studyUid, studyDate, studyTime, studyDesc, accessionNumber, referringPhysician,
            seriesUid, modality, seriesNumber, seriesDesc, bodyPart, institution, stationName, sendingAet,
            sopUid, sopClassUid, instanceNumber, numFrames,
            transferSyntax,
            attrs  // keep full Attributes for StoragePathResolver
        );
    }
}
```

### 2. `src/main/java/com/spax/ingest/DicomMetadata.java`

Value object chứa metadata đã parse:

```java
public record DicomMetadata(
    // Patient
    String patientId,
    String patientName,
    String birthDate,    // "YYYYMMDD" or null
    String sex,          // "M", "F", "O" or null

    // Study
    String studyInstanceUid,
    String studyDate,    // "YYYYMMDD" or null
    String studyTime,
    String studyDescription,
    String accessionNumber,
    String referringPhysician,

    // Series
    String seriesInstanceUid,
    String modality,
    int seriesNumber,
    String seriesDescription,
    String bodyPart,
    String institution,
    String stationName,
    String sendingAet,

    // Instance
    String sopInstanceUid,
    String sopClassUid,
    int instanceNumber,
    int numFrames,
    String transferSyntaxUid,

    // Full DICOM Attributes (for StoragePathResolver and WADO-RS)
    // Note: this is a reference — don't hold after InputStream is closed
    org.dcm4che3.data.Attributes rawAttributes
) {}
```

## Lưu ý quan trọng
- `IncludeBulkData.NO` là bắt buộc — không load pixel data vào RAM. DICOM file CT/MR có thể 100MB+
- `Tag.PatientID` là `0x00100020`, `Tag.StudyInstanceUID` là `0x0020000D`, etc. — dùng dcm4che `Tag` constants
- `transferSyntax` lấy từ `dis.getTransferSyntax()` sau khi read, KHÔNG phải từ dataset
- Missing PID → tạo synthetic ID `NOPID_{studyUid[0..16]}` — đây là quy tắc bắt buộc của SPAX
- `rawAttributes` ref được giữ trong `DicomMetadata` record. Caller phải dùng xong trước khi đóng stream.
- Modality fallback về `"OT"` (Other) khi thiếu — nhiều file legacy không có Modality tag

## Tag reference (dcm4che `Tag` constants)
```
PatientID              = 0x00100020
PatientName            = 0x00100010
PatientBirthDate       = 0x00100030
PatientSex             = 0x00100040
StudyInstanceUID       = 0x0020000D
StudyDate              = 0x00080020
StudyTime              = 0x00080030
StudyDescription       = 0x00081030
AccessionNumber        = 0x00080050
ReferringPhysicianName = 0x00080090
SeriesInstanceUID      = 0x0020000E
Modality               = 0x00080060
SeriesNumber           = 0x00200011
SeriesDescription      = 0x0008103E
BodyPartExamined       = 0x00180015
InstitutionName        = 0x00080080
StationName            = 0x00081010
SourceApplicationEntityTitle = 0x00020016
SOPInstanceUID         = 0x00080018
SOPClassUID            = 0x00080016
InstanceNumber         = 0x00200013
NumberOfFrames         = 0x00280008
```

## Kiểm tra thành công
- Parse test DICOM file → tất cả fields populated đúng
- Parse với missing PatientID → `patientId` = `"NOPID_<studyUid_prefix>"`
- Parse corrupt/non-DICOM file → throws IOException
- Verify memory: không load pixel data (file 50MB CT → thời gian parse < 100ms, RAM dùng < 5MB)
