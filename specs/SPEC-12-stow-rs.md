# SPEC-12: DICOMWeb STOW-RS (Store)

## Mục tiêu
Implement STOW-RS endpoint — DICOMWeb standard cho upload DICOM. Cho phép DICOMWeb clients (không chỉ Orthanc) push trực tiếp lên SPAX qua chuẩn PS3.18.

## Dependencies
- SPEC-08 (IngestService — reuse logic)

## Files cần tạo

### 1. `src/main/java/com/spax/dicomweb/StowController.java`

```java
@RestController
@RequestMapping("/dicomweb/{tenant}/studies")
public class StowController {

    @Autowired private IngestService ingestService;

    /**
     * POST /dicomweb/{tenant}/studies
     * Accept: multipart/related; type="application/dicom"
     *
     * Per DICOM PS3.18, response is XML or JSON listing which SOPs were stored.
     * Returns:
     *   200 OK:         All instances stored successfully
     *   202 Accepted:   Some stored, some failed
     *   409 Conflict:   None stored (all failed)
     *
     * Response body: application/dicom+xml or application/dicom+json
     */
    @PostMapping(
        consumes = {"multipart/related; type=\"application/dicom\"", "multipart/related"},
        produces = "application/dicom+json"
    )
    public ResponseEntity<Map<String, Object>> storeInstances(
            @PathVariable String tenant,
            HttpServletRequest request) throws IOException {

        List<String> successSops = new ArrayList<>();
        List<String> failedSops = new ArrayList<>();

        // Parse multipart/related body manually (Spring's @RequestParam doesn't handle it)
        String contentType = request.getContentType();
        String boundary = extractBoundary(contentType);

        if (boundary == null) {
            return ResponseEntity.badRequest().body(Map.of("error", "Missing boundary in Content-Type"));
        }

        // Use Apache Commons FileUpload or manual multipart parsing
        // Simple approach: read each MIME part
        List<byte[]> parts = parseMultipartRelated(request.getInputStream(), boundary);

        for (byte[] dicomBytes : parts) {
            try {
                // Quick validation: DICOM magic bytes at offset 128 = "DICM"
                if (!isDicom(dicomBytes)) {
                    failedSops.add("UNKNOWN_" + System.nanoTime());
                    continue;
                }

                // Extract SOP UID quickly for response
                String sopUid = extractSopUid(dicomBytes);

                // Delegate to IngestService (async — same pipeline as REST ingest)
                ingestService.acceptBytes(dicomBytes, tenant);

                successSops.add(sopUid);

            } catch (Exception e) {
                log.warn("Failed to process STOW-RS part: {}", e.getMessage());
                failedSops.add("FAILED_" + System.nanoTime());
            }
        }

        // Build DICOM JSON response per PS3.18
        Map<String, Object> response = buildStowResponse(successSops, failedSops);

        if (failedSops.isEmpty()) {
            return ResponseEntity.ok(response);
        } else if (successSops.isEmpty()) {
            return ResponseEntity.status(409).body(response);
        } else {
            return ResponseEntity.status(202).body(response);
        }
    }

    // ─── Helpers ──────────────────────────────────────────────────────────

    private String extractBoundary(String contentType) {
        if (contentType == null) return null;
        // Content-Type: multipart/related; type="application/dicom"; boundary=----BOUNDARY
        for (String part : contentType.split(";")) {
            part = part.trim();
            if (part.startsWith("boundary=")) {
                return part.substring("boundary=".length()).replace("\"", "");
            }
        }
        return null;
    }

    private List<byte[]> parseMultipartRelated(InputStream body, String boundary) throws IOException {
        // Read full body (STOW-RS typically not huge per request)
        byte[] fullBody = body.readAllBytes();
        List<byte[]> parts = new ArrayList<>();

        byte[] boundaryBytes = ("--" + boundary).getBytes(StandardCharsets.ISO_8859_1);
        // Simple split by boundary — handle DICOM binary data correctly
        // Use index-based scanning to avoid charset issues with binary data

        int start = findBytes(fullBody, boundaryBytes, 0);
        while (start != -1) {
            // Skip boundary line + headers
            int headerEnd = findBytes(fullBody, new byte[]{'\r','\n','\r','\n'}, start);
            if (headerEnd == -1) break;

            int dataStart = headerEnd + 4;
            int nextBoundary = findBytes(fullBody, boundaryBytes, dataStart);

            if (nextBoundary == -1) break;

            // Extract part bytes (exclude trailing \r\n before next boundary)
            int dataEnd = nextBoundary - 2; // -2 for \r\n
            if (dataEnd > dataStart) {
                byte[] part = Arrays.copyOfRange(fullBody, dataStart, dataEnd);
                parts.add(part);
            }

            start = nextBoundary;
        }

        return parts;
    }

    private boolean isDicom(byte[] bytes) {
        // DICOM files have "DICM" at offset 128
        if (bytes.length < 132) return false;
        return bytes[128] == 'D' && bytes[129] == 'I' && bytes[130] == 'C' && bytes[131] == 'M';
    }

    private String extractSopUid(byte[] dicomBytes) {
        // Quick extraction without full parse — read SOPInstanceUID (0008,0018) tag
        try (DicomInputStream dis = new DicomInputStream(new ByteArrayInputStream(dicomBytes))) {
            dis.setIncludeBulkData(DicomInputStream.IncludeBulkData.NO);
            Attributes attrs = dis.readDataset();
            return attrs.getString(Tag.SOPInstanceUID, "UNKNOWN");
        } catch (IOException e) {
            return "UNKNOWN";
        }
    }

    private Map<String, Object> buildStowResponse(List<String> successSops, List<String> failedSops) {
        // Per PS3.18, response contains ReferencedSOPSequence and FailedSOPSequence
        Map<String, Object> response = new LinkedHashMap<>();

        // ReferencedSOPSequence (0008,1199)
        if (!successSops.isEmpty()) {
            List<Map<String, Object>> referencedSops = successSops.stream()
                .map(sop -> Map.<String, Object>of(
                    "00081150", Map.of("vr", "UI"),  // ReferencedSOPClassUID (unknown at this point)
                    "00081155", Map.of("vr", "UI", "Value", List.of(sop))  // ReferencedSOPInstanceUID
                ))
                .toList();
            response.put("00081199", Map.of("vr", "SQ", "Value", referencedSops));
        }

        // FailedSOPSequence (0008,1198)
        if (!failedSops.isEmpty()) {
            List<Map<String, Object>> failedSeq = failedSops.stream()
                .map(sop -> Map.<String, Object>of(
                    "00081197", Map.of("vr", "US", "Value", List.of(272)),  // Failure reason
                    "00081155", Map.of("vr", "UI", "Value", List.of(sop))
                ))
                .toList();
            response.put("00081198", Map.of("vr", "SQ", "Value", failedSeq));
        }

        return response;
    }

    private int findBytes(byte[] data, byte[] pattern, int start) {
        outer:
        for (int i = start; i <= data.length - pattern.length; i++) {
            for (int j = 0; j < pattern.length; j++) {
                if (data[i + j] != pattern[j]) continue outer;
            }
            return i;
        }
        return -1;
    }
}
```

## Lưu ý quan trọng
- STOW-RS request body là `multipart/related; type="application/dicom"` — không phải standard `multipart/form-data`
- Phải parse multipart manually vì Spring `@RequestParam` không handle `multipart/related` correctly
- DICOM magic bytes check ở offset 128 (sau 128-byte preamble)
- Reuse `IngestService.acceptBytes()` — không duplicate indexing logic
- Response format per PS3.18 — OHIF/DICOMweb clients kiểm tra format này
- HTTP 200 = all success, 202 = partial, 409 = all failed

## Kiểm tra thành công
- `POST /dicomweb/test/studies` với valid DICOM file → 200 OK với ReferencedSOPSequence
- `POST` với mixed valid/invalid → 202 với both sequences
- File sau STOW-RS → queryable via QIDO-RS
- OHIF configured with STOW-RS → drag-and-drop upload works
