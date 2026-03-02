# SPEC-22: WADO-RS Frame Retrieval — Xử lý ảnh nén, multiframe

## Mục tiêu
Implement proper WADO-RS frame extraction cho endpoint `GET /dicomweb/{tenant}/.../frames/{frameList}`. Thay thế placeholder hiện tại (trả toàn bộ DICOM file) bằng logic trích xuất đúng pixel data frames. Hỗ trợ đầy đủ 4 loại ảnh DICOM:
- **UncompressedSingleFrame**: Implicit/Explicit VR, 1 frame (CT, DR, CR)
- **CompressedSingleFrame**: JPEG/JPEG-LS/JPEG2000/RLE, 1 frame
- **UncompressedMultiFrame**: Uncompressed, N frames (US cine, MR dynamic)
- **CompressedMultiFrame**: Compressed encapsulated, N frames

## Dependencies
- SPEC-02 (instance table: `transfer_syntax_uid`, `num_frames` — đã có)
- SPEC-04/05 (StorageService — `retrieve()` trả InputStream)
- SPEC-07 (DicomParser — đã extract numFrames + transferSyntax khi ingest)
- SPEC-11 (WadoController — sửa `retrieveFrames()`)
- SPEC-21 (WadoRsCacheService — mở rộng `InstanceLocation` record)

## Thiết kế

### Per-frame InputStream (V1)
Mỗi frame mở 1 InputStream riêng: open → parse header → skip tới frame → đọc → close. Đơn giản, stateless, dễ debug. Local storage: nhanh nhờ OS page cache. Cloud: N GET requests cho N frames — chấp nhận được vì OHIF thường gọi 1 frame/HTTP request.

```
Request: GET /frames/1,3,5

Frame 1: openStream() → skip header → read frame 1 → close
Frame 3: openStream() → skip header → skip 2 frames → read frame 3 → close
Frame 5: openStream() → skip header → skip 4 frames → read frame 5 → close
```

### Upgrade path → Single-pass (V2, sau)
Khi cần tối ưu cloud storage: chuyển sang single-pass (1 InputStream cho tất cả frames). Chỉ thay đổi nội bộ `FrameExtractor` + `FrameRetrievalService` — không ảnh hưởng `WadoController`, `FrameType`, `MultipartFrameWriter`.

### Không transcoding
Server trả pixel data ở native transfer syntax. OHIF's cornerstonejs tự decompress client-side (JPEG, JPEG-LS, JPEG2000, RLE đều có codec). Đơn giản, đúng SPAX principle.

## Files cần tạo

### 1. `src/main/java/com/spax/dicomweb/frame/FrameType.java`

Phân loại ảnh DICOM dựa trên transfer syntax UID + number of frames. Thông tin lấy từ cache (không cần đọc file).

```java
package com.spax.dicomweb.frame;

import org.dcm4che3.data.UID;
import java.util.Set;

/**
 * Classifies DICOM images for frame extraction.
 * Uses transfer syntax UID (from instance table) and number of frames
 * to determine the pixel data structure and extraction strategy.
 *
 * Uncompressed: pixel data is a contiguous byte block. Frame N starts at (N-1)*frameLength.
 * Compressed: pixel data is encapsulated as DICOM items (fragments). Each frame = 1 item.
 */
public enum FrameType {

    UNCOMPRESSED_SINGLE,   // Native pixel data, 1 frame
    UNCOMPRESSED_MULTI,    // Native pixel data, N frames
    COMPRESSED_SINGLE,     // Encapsulated pixel data, 1 frame
    COMPRESSED_MULTI,      // Encapsulated pixel data, N frames
    VIDEO;                 // MPEG2/MPEG4/HEVC — treat as single compressed blob

    private static final Set<String> UNCOMPRESSED_SYNTAXES = Set.of(
        UID.ImplicitVRLittleEndian,    // 1.2.840.10008.1.2
        UID.ExplicitVRLittleEndian,    // 1.2.840.10008.1.2.1
        UID.ExplicitVRBigEndian        // 1.2.840.10008.1.2.2 (retired)
    );

    private static final Set<String> VIDEO_SYNTAXES = Set.of(
        UID.MPEG2MPML, UID.MPEG2MPMLF,
        UID.MPEG2MPHL, UID.MPEG2MPHLF,
        UID.MPEG4HP41, UID.MPEG4HP41F,
        UID.MPEG4HP41BD, UID.MPEG4HP41BDF,
        UID.MPEG4HP422D, UID.MPEG4HP422DF,
        UID.MPEG4HP423D, UID.MPEG4HP423DF,
        UID.MPEG4HP42STEREO, UID.MPEG4HP42STEREOF,
        UID.HEVCMP51, UID.HEVCM10P51
    );

    /**
     * Classify a DICOM instance for frame extraction.
     *
     * @param transferSyntaxUid from instance.transfer_syntax_uid (cached in InstanceLocation)
     * @param numFrames         from instance.num_frames (cached in InstanceLocation)
     */
    public static FrameType classify(String transferSyntaxUid, int numFrames) {
        if (transferSyntaxUid == null) {
            // Defensive: no TS info → treat as uncompressed
            return numFrames > 1 ? UNCOMPRESSED_MULTI : UNCOMPRESSED_SINGLE;
        }
        if (VIDEO_SYNTAXES.contains(transferSyntaxUid)) {
            return VIDEO;
        }
        if (UNCOMPRESSED_SYNTAXES.contains(transferSyntaxUid)) {
            return numFrames > 1 ? UNCOMPRESSED_MULTI : UNCOMPRESSED_SINGLE;
        }
        return numFrames > 1 ? COMPRESSED_MULTI : COMPRESSED_SINGLE;
    }

    /**
     * Whether this type uses encapsulated (compressed) pixel data format.
     * Determines Content-Type header: compressed includes transfer-syntax parameter.
     */
    public boolean isCompressed() {
        return this == COMPRESSED_SINGLE || this == COMPRESSED_MULTI || this == VIDEO;
    }
}
```

### 2. `src/main/java/com/spax/dicomweb/frame/FrameExtractor.java`

Core frame extraction logic. Per-frame: mỗi lần gọi extract 1 frame từ 1 fresh InputStream.

```java
package com.spax.dicomweb.frame;

import org.dcm4che3.data.Attributes;
import org.dcm4che3.data.Tag;
import org.dcm4che3.imageio.codec.ImageDescriptor;
import org.dcm4che3.io.DicomInputStream;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;

/**
 * Extracts a single pixel data frame from a DICOM file stream.
 *
 * Design: per-frame — each call receives a fresh InputStream positioned at file start.
 * Stateless, simple, easy to debug.
 *
 * Works with any InputStream (local file, cloud storage GET response).
 * Does NOT buffer entire file in RAM — reads header (~few KB), skips to frame, streams pixel data.
 *
 * Reference implementations:
 * - store-server-2x: UncompressedMultiFrameInputStream, CompressedMultiFrameInputStream
 * - dcm4chee-arc-light: UncompressedFramesOutput, CompressedFramesOutput
 *
 * Upgrade path: khi cần tối ưu cloud (1 GET thay vì N), chuyển sang single-pass
 * với extractFrames(stream, List<frameNumbers>, callback). Chỉ thay đổi nội bộ,
 * không ảnh hưởng caller.
 */
@Component
public class FrameExtractor {

    private static final Logger log = LoggerFactory.getLogger(FrameExtractor.class);
    private static final int BUFFER_SIZE = 8192;

    /**
     * Extract 1 frame from a DICOM stream, write pixel bytes to output.
     *
     * @param dicomStream  fresh DICOM file InputStream (positioned at file start).
     *                     Caller owns lifecycle (closes after return).
     * @param frameNumber  1-based frame number (per DICOMWeb spec)
     * @param frameType    pre-classified image type (from FrameType.classify)
     * @param output       target OutputStream for pixel bytes
     * @throws IOException              if stream read/write fails or pixel data missing
     * @throws IllegalArgumentException if frameNumber out of range
     */
    public void extractFrame(InputStream dicomStream, int frameNumber,
                             FrameType frameType, OutputStream output) throws IOException {
        switch (frameType) {
            case UNCOMPRESSED_SINGLE ->
                extractUncompressedSingle(dicomStream, output);
            case COMPRESSED_SINGLE, VIDEO ->
                extractCompressedSingle(dicomStream, output);
            case UNCOMPRESSED_MULTI ->
                extractUncompressedMulti(dicomStream, frameNumber, output);
            case COMPRESSED_MULTI ->
                extractCompressedMulti(dicomStream, frameNumber, output);
        }
    }

    // ─── Uncompressed Single Frame ────────────────────────────────────

    /**
     * Uncompressed single frame: pixel data is a contiguous byte block after the header.
     * Read header → stream all pixel bytes.
     */
    private void extractUncompressedSingle(InputStream rawStream,
                                            OutputStream output) throws IOException {
        try (DicomInputStream dis = new DicomInputStream(rawStream)) {
            dis.readDataset(-1, Tag.PixelData);

            if (dis.tag() != Tag.PixelData) {
                throw new IOException("No pixel data found in DICOM file");
            }

            int pixelDataLength = dis.length();
            log.debug("Uncompressed single frame: {} bytes pixel data", pixelDataLength);

            copyExactly(dis, output, pixelDataLength);
        }
    }

    // ─── Compressed Single Frame ──────────────────────────────────────

    /**
     * Compressed single frame: encapsulated pixel data format.
     *
     * Structure per DICOM PS3.5 A.4:
     *   PixelData element (tag=7FE0,0010, length=-1)
     *     Item: Basic Offset Table (may be empty, length=0)
     *     Item: fragment 1 (compressed frame data)
     *     [Item: fragment 2 ...]  ← single frame may span multiple fragments
     *     Sequence Delimiter (FFFE,E00D)
     *
     * Strategy: skip BOT, concatenate all remaining fragments until delimiter.
     * Also used for VIDEO transfer syntaxes (entire pixel blob).
     */
    private void extractCompressedSingle(InputStream rawStream,
                                          OutputStream output) throws IOException {
        try (DicomInputStream dis = new DicomInputStream(rawStream)) {
            dis.readDataset(-1, Tag.PixelData);

            if (dis.tag() != Tag.PixelData || dis.length() != -1) {
                throw new IOException("No encapsulated pixel data found");
            }

            // Skip Basic Offset Table (first item)
            dis.readHeader();
            dis.skipFully(dis.length());

            // Read all remaining fragments until Sequence Delimiter
            byte[] buf = new byte[BUFFER_SIZE];
            while (dis.readHeader()) {
                if (dis.tag() == Tag.SequenceDelimitationItem) {
                    break;
                }
                int itemLength = dis.length();
                copyExactly(dis, output, itemLength, buf);
            }

            log.debug("Compressed single frame extracted");
        }
    }

    // ─── Uncompressed Multi Frame ─────────────────────────────────────

    /**
     * Uncompressed multi-frame: pixel data is a contiguous block.
     * Frame N occupies bytes [(N-1)*frameLength .. N*frameLength).
     * frameLength = rows × columns × (bitsAllocated/8) × samplesPerPixel.
     *
     * Skip to the requested frame, read exactly frameLength bytes.
     *
     * Ref: store-server-2x UncompressedMultiFrameInputStream
     */
    private void extractUncompressedMulti(InputStream rawStream, int frameNumber,
                                           OutputStream output) throws IOException {
        try (DicomInputStream dis = new DicomInputStream(rawStream)) {
            Attributes attrs = dis.readDataset(-1, Tag.PixelData);

            if (dis.tag() != Tag.PixelData && dis.tag() != Tag.FloatPixelData
                    && dis.tag() != Tag.DoubleFloatPixelData) {
                throw new IOException("No pixel data found in DICOM file");
            }

            int totalFrames = attrs.getInt(Tag.NumberOfFrames, 1);
            int frameLength = new ImageDescriptor(attrs).getFrameLength();
            log.debug("Uncompressed multi-frame: {} frames, {} bytes/frame", totalFrames, frameLength);

            if (frameNumber < 1 || frameNumber > totalFrames) {
                throw new IllegalArgumentException(
                    "Frame " + frameNumber + " out of range [1.." + totalFrames + "]");
            }

            // Skip to requested frame
            long bytesToSkip = (long) (frameNumber - 1) * frameLength;
            if (bytesToSkip > 0) {
                skipExactly(dis, bytesToSkip);
            }

            // Read exactly 1 frame
            byte[] buf = new byte[BUFFER_SIZE];
            copyExactly(dis, output, frameLength, buf);
        }
    }

    // ─── Compressed Multi Frame ───────────────────────────────────────

    /**
     * Compressed multi-frame: encapsulated format with 1 item (fragment) per frame.
     *
     * Structure:
     *   PixelData (length=-1)
     *     Item: Basic Offset Table
     *     Item: frame 1 compressed data
     *     Item: frame 2 compressed data
     *     ...
     *     Sequence Delimiter
     *
     * Iterate through item headers, skip (frameNumber-1) items, read the target item.
     *
     * Note: per DICOM PS3.5 A.4, frames MAY span multiple fragments.
     * This implementation assumes 1 fragment = 1 frame, which covers
     * the vast majority of real-world DICOM data.
     *
     * Ref: store-server-2x CompressedMultiFrameInputStream
     */
    private void extractCompressedMulti(InputStream rawStream, int frameNumber,
                                         OutputStream output) throws IOException {
        try (DicomInputStream dis = new DicomInputStream(rawStream)) {
            dis.readDataset(-1, Tag.PixelData);

            if (dis.tag() != Tag.PixelData || dis.length() != -1) {
                throw new IOException("No encapsulated pixel data found");
            }

            // Skip Basic Offset Table
            dis.readHeader();
            dis.skipFully(dis.length());

            // Skip items until we reach the requested frame
            for (int i = 1; i < frameNumber; i++) {
                if (!dis.readHeader() || dis.tag() == Tag.SequenceDelimitationItem) {
                    throw new IllegalArgumentException(
                        "Frame " + frameNumber + " exceeds available frames (hit end at frame " + i + ")");
                }
                dis.skipFully(dis.length());
            }

            // Read the requested frame's item
            if (!dis.readHeader() || dis.tag() == Tag.SequenceDelimitationItem) {
                throw new IllegalArgumentException(
                    "Frame " + frameNumber + " exceeds available frames");
            }

            int itemLength = dis.length();
            byte[] buf = new byte[BUFFER_SIZE];
            copyExactly(dis, output, itemLength, buf);

            log.debug("Compressed multi-frame: extracted frame {} ({} bytes)", frameNumber, itemLength);
        }
    }

    // ─── Stream helpers ───────────────────────────────────────────────

    private void copyExactly(InputStream in, OutputStream out, int length) throws IOException {
        copyExactly(in, out, length, new byte[BUFFER_SIZE]);
    }

    private void copyExactly(InputStream in, OutputStream out, int length, byte[] buf)
            throws IOException {
        int remaining = length;
        while (remaining > 0) {
            int toRead = Math.min(buf.length, remaining);
            int read = in.read(buf, 0, toRead);
            if (read < 0) {
                throw new IOException("Unexpected end of stream, needed " + remaining + " more bytes");
            }
            out.write(buf, 0, read);
            remaining -= read;
        }
    }

    private void skipExactly(InputStream in, long n) throws IOException {
        long remaining = n;
        while (remaining > 0) {
            long skipped = in.skip(remaining);
            if (skipped <= 0) {
                // Some InputStreams return 0 from skip; fall back to reading
                int read = in.read();
                if (read < 0) {
                    throw new IOException("Unexpected end of stream, needed to skip " + remaining + " more bytes");
                }
                remaining--;
            } else {
                remaining -= skipped;
            }
        }
    }
}
```

### 3. `src/main/java/com/spax/dicomweb/frame/MultipartFrameWriter.java`

Builds multipart/related response. Mỗi frame = 1 MIME part với Content-Type header.

```java
package com.spax.dicomweb.frame;

import java.io.IOException;
import java.io.OutputStream;
import java.nio.charset.StandardCharsets;

/**
 * Writes MIME multipart/related response for WADO-RS frame retrieval.
 *
 * Each frame becomes one MIME part:
 *   \r\n--{boundary}\r\n
 *   Content-Type: application/octet-stream[; transfer-syntax={tsuid}]\r\n
 *   \r\n
 *   [frame pixel bytes]
 *
 * Per DICOMWeb PS3.18 8.2.4:
 * - Uncompressed frames: Content-Type: application/octet-stream
 * - Compressed frames: Content-Type: application/octet-stream; transfer-syntax={tsuid}
 */
public class MultipartFrameWriter {

    private final OutputStream output;
    private final String boundary;
    private final String partContentType;

    /**
     * @param output             response output stream
     * @param boundary           multipart boundary string
     * @param transferSyntaxUid  transfer syntax UID (used in Content-Type for compressed)
     * @param compressed         true if pixel data is compressed/encapsulated
     */
    public MultipartFrameWriter(OutputStream output, String boundary,
                                 String transferSyntaxUid, boolean compressed) {
        this.output = output;
        this.boundary = boundary;

        // Pre-compute Content-Type string (same for all parts within one instance)
        if (compressed) {
            this.partContentType = "Content-Type: application/octet-stream; transfer-syntax="
                + transferSyntaxUid + "\r\n";
        } else {
            this.partContentType = "Content-Type: application/octet-stream\r\n";
        }
    }

    /**
     * Write the MIME part header for one frame.
     * After calling this, write frame pixel bytes directly to getOutputStream().
     */
    public void writePartHeader(int frameNumber) throws IOException {
        StringBuilder sb = new StringBuilder();
        sb.append("\r\n--").append(boundary).append("\r\n");
        sb.append(partContentType);
        sb.append("\r\n"); // blank line separating headers from body

        output.write(sb.toString().getBytes(StandardCharsets.UTF_8));
    }

    /**
     * The output stream for writing frame pixel bytes.
     * Same stream for all parts (multipart is written sequentially).
     */
    public OutputStream getOutputStream() {
        return output;
    }

    /**
     * Write the closing multipart boundary. Must be called after all parts are written.
     */
    public void writeEnd() throws IOException {
        output.write(("\r\n--" + boundary + "--\r\n").getBytes(StandardCharsets.UTF_8));
        output.flush();
    }
}
```

### 4. `src/main/java/com/spax/dicomweb/frame/FrameRetrievalService.java`

Orchestrator: kết nối cache → storage → extraction → multipart output. Loop per frame.

```java
package com.spax.dicomweb.frame;

import com.spax.dicomweb.WadoRsCacheService;
import com.spax.storage.StorageService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.util.List;

/**
 * Orchestrates WADO-RS frame retrieval.
 *
 * Flow (per-frame):
 * 1. Validate frame numbers against cached numFrames
 * 2. Classify image type from cached transferSyntaxUid + numFrames
 * 3. For each frame:
 *    a. Write MIME part header
 *    b. Open DICOM file stream from storage
 *    c. Extract frame via FrameExtractor
 *    d. Close stream
 * 4. Write multipart end boundary
 *
 * Upgrade path: khi cần tối ưu → open 1 stream, extract tất cả frames single-pass.
 * Chỉ cần thay đổi class này + FrameExtractor. Caller (WadoController) không đổi.
 */
@Service
public class FrameRetrievalService {

    private static final Logger log = LoggerFactory.getLogger(FrameRetrievalService.class);

    @Autowired
    private StorageService storageService;

    @Autowired
    private FrameExtractor frameExtractor;

    /**
     * Extract requested frames and write as multipart/related response.
     *
     * @param loc          cached instance location (includes TS UID and numFrames)
     * @param frameNumbers 1-based, sorted ascending, validated by caller
     * @param output       response output stream
     * @param boundary     multipart boundary string
     * @throws IOException              on stream errors
     * @throws IllegalArgumentException on invalid frame numbers
     */
    public void writeFrames(WadoRsCacheService.InstanceLocation loc,
                            List<Integer> frameNumbers,
                            OutputStream output,
                            String boundary) throws IOException {

        // 1. Validate frame range
        int maxFrame = loc.numFrames();
        for (int frame : frameNumbers) {
            if (frame < 1 || frame > maxFrame) {
                throw new IllegalArgumentException(
                    "Frame " + frame + " out of range [1.." + maxFrame + "]");
            }
        }

        // 2. Classify image type
        FrameType frameType = FrameType.classify(loc.transferSyntaxUid(), loc.numFrames());

        // 3. Create multipart writer
        MultipartFrameWriter writer = new MultipartFrameWriter(
            output, boundary, loc.transferSyntaxUid(), frameType.isCompressed()
        );

        // 4. Extract each frame (per-frame InputStream open)
        for (int frameNumber : frameNumbers) {
            writer.writePartHeader(frameNumber);

            try (InputStream dicomStream = storageService.retrieve(
                    loc.volumeId(), loc.storagePath())) {
                frameExtractor.extractFrame(dicomStream, frameNumber, frameType, output);
            }
        }

        // 5. Close multipart
        writer.writeEnd();

        log.debug("Wrote {} frames for {} (type={})",
            frameNumbers.size(), loc.storagePath(), frameType);
    }
}
```

## Files cần sửa

### 5. Sửa `WadoRsCacheService` (SPEC-21)

#### Thay đổi `InstanceLocation` record

```java
// ── TRƯỚC ──────────────────────────────────────────────────────────
public record InstanceLocation(int volumeId, String storagePath) implements Serializable {}

// ── SAU ────────────────────────────────────────────────────────────
public record InstanceLocation(int volumeId, String storagePath,
                                String transferSyntaxUid, int numFrames) implements Serializable {}
```

#### Thay đổi `loadSeriesLocations()` query + mapping

```java
private SeriesInstanceLocations loadSeriesLocations(String seriesUid) {
    List<Long> seriesIds = jdbcTemplate.queryForList(
        "SELECT id FROM series WHERE series_instance_uid = ?",
        Long.class, seriesUid
    );
    if (seriesIds.isEmpty()) return null;

    List<Map<String, Object>> rows = jdbcTemplate.queryForList("""
        SELECT sop_instance_uid, volume_id, storage_path,
               transfer_syntax_uid, num_frames
        FROM instance WHERE series_fk = ?
        """, seriesIds.get(0)
    );
    if (rows.isEmpty()) return null;

    Map<String, InstanceLocation> map = HashMap.newHashMap(rows.size());
    for (Map<String, Object> row : rows) {
        map.put(
            (String) row.get("sop_instance_uid"),
            new InstanceLocation(
                ((Number) row.get("volume_id")).intValue(),
                (String) row.get("storage_path"),
                (String) row.get("transfer_syntax_uid"),
                row.get("num_frames") != null ? ((Number) row.get("num_frames")).intValue() : 1
            )
        );
    }

    log.debug("Loaded {} instance locations for series {}", map.size(), seriesUid);
    return new SeriesInstanceLocations(Collections.unmodifiableMap(map));
}
```

**Impact lên caller cũ**: Không ảnh hưởng. `retrieveInstance()`, `buildMultipartResponse()` chỉ dùng `.volumeId()` và `.storagePath()`. 2 field mới thêm vào record nhưng caller cũ không gọi chúng.

**Cache memory impact**: Thêm ~80 bytes/instance (1 String TS UID ~64 chars + 1 int). Với 300 instances/series × 500 series cache: ~12 MB thêm — chấp nhận được.

### 6. Sửa `WadoController.retrieveFrames()` (SPEC-11)

Thay thế method `retrieveFrames()` hiện tại trong WadoController:

```java
@Autowired
private FrameRetrievalService frameRetrievalService;

/**
 * GET /dicomweb/{tenant}/.../instances/{sopUid}/frames/{frameList}
 *
 * OHIF gọi để lấy pixel data frames.
 * frameList: "1" hoặc "1,2,3" (1-based, comma-separated).
 *
 * Returns: multipart/related response.
 * - Mỗi frame = 1 MIME part
 * - Uncompressed: Content-Type: application/octet-stream
 * - Compressed: Content-Type: application/octet-stream; transfer-syntax={tsuid}
 *
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

    // 5. Stream response (non-buffered)
    StreamingResponseBody body = outputStream -> {
        frameRetrievalService.writeFrames(loc, frameNumbers, outputStream, boundary);
    };

    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType(contentType))
        .body(body);
}
```

## Lưu ý quan trọng

### dcm4che DicomInputStream API

Sau `dis.readDataset(-1, Tag.PixelData)`:
- `dis.tag()` = `Tag.PixelData` (0x7FE00010) nếu có pixel data
- `dis.length()` = pixel data length:
  - **Uncompressed**: actual byte count (e.g., 512×512×2 = 524288)
  - **Compressed**: -1 (undefined length — encapsulated format)
- `dis.readHeader()` đọc tag + VR + length tiếp theo (dùng cho item headers trong encapsulated data)
- `dis.skipFully(n)` skip n bytes

### ImageDescriptor.getFrameLength()

Tính bytes per frame từ DICOM attributes:
```
frameLength = rows × columns × (bitsAllocated / 8) × samplesPerPixel
```
Nếu `PlanarConfiguration = 1` (color by plane): khác calculation. `ImageDescriptor` handle tự động.

### Encapsulated Pixel Data Structure (PS3.5 A.4)

```
(7FE0,0010) PixelData, length=-1
  (FFFE,E000) Item: Basic Offset Table [length=0 or 4*N]
  (FFFE,E000) Item: fragment 1 [compressed frame 1]
  (FFFE,E000) Item: fragment 2 [compressed frame 2]
  ...
  (FFFE,E00D) Sequence Delimiter [length=0]
```

- Basic Offset Table: optional, often empty (length=0)
- Cho multi-frame: mỗi frame = 1 fragment (item)
- Cho single-frame: frame có thể span multiple fragments → đọc tất cả tới delimiter

### Content-Type headers

**Outer response header** (HTTP Content-Type):
```
multipart/related; type="application/octet-stream"; boundary={boundary}
// hoặc cho compressed:
multipart/related; type="application/octet-stream; transfer-syntax={tsuid}"; boundary={boundary}
```

**Per-part header** (trong multipart body):
```
Content-Type: application/octet-stream
// hoặc cho compressed:
Content-Type: application/octet-stream; transfer-syntax={tsuid}
```

### Upgrade V1 → V2 (single-pass)

Khi cần tối ưu (cloud storage N GET → 1 GET), chỉ thay đổi 2 files:

1. **`FrameExtractor`**: thêm method `extractFrames(stream, List<frameNumbers>, callback)` — giữ stream open, navigate qua tất cả frames ascending
2. **`FrameRetrievalService`**: thay loop `for (frame: open → extract → close)` bằng single open + delegate to `extractFrames`

Không thay đổi: `FrameType`, `MultipartFrameWriter`, `WadoController`, `WadoRsCacheService`.

## Kiểm tra thành công

1. **Uncompressed single-frame CT**: `GET /frames/1` → 200, pixel data size = rows × cols × (bitsAllocated/8) × samplesPerPixel
2. **Compressed single-frame (JPEG-LS)**: `GET /frames/1` → 200, Content-Type includes `transfer-syntax=1.2.840.10008.1.2.4.80`
3. **Uncompressed multi-frame US (20 frames)**: `GET /frames/5` → 200, pixel data = 1 frame size. `GET /frames/1,5,10` → 3 MIME parts
4. **Compressed multi-frame**: `GET /frames/3` → 200, returns compressed fragment 3
5. **Out of range**: `GET /frames/0` → 400. `GET /frames/999` → 400
6. **Invalid**: `GET /frames/abc` → 400
7. **OHIF integration**: mở CT series → slices load. Mở US cine → cine plays. Mở compressed study → frames decode client-side
8. **Cache verify**: first request → cache miss (batch load) → DB query. Subsequent → cache hit → 0 DB queries
