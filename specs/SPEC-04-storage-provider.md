# SPEC-04: Storage Provider SPI + Local + jclouds

## Mục tiêu
Implement tầng storage abstraction với 2 providers: Local filesystem (Java NIO) và Cloud (Apache jclouds). Đây là tầng thấp nhất — chỉ đọc/ghi bytes, không biết gì về DICOM hay DB.

## Dependencies
- SPEC-01 (project setup, pom.xml với jclouds)

## Files cần tạo

### 1. `src/main/java/com/spax/storage/StorageProvider.java`

Interface SPI cho từng loại storage:

```java
public interface StorageProvider {
    /**
     * Write data to storage.
     * @param relativePath path relative to this provider's base path
     * @param data input stream of bytes
     * @param size total bytes, -1 if unknown
     */
    void write(String relativePath, InputStream data, long size) throws IOException;

    /**
     * Read data from storage.
     * @return InputStream — caller must close
     */
    InputStream read(String relativePath) throws IOException;

    void delete(String relativePath) throws IOException;

    boolean exists(String relativePath) throws IOException;

    /**
     * Get file size in bytes.
     */
    long size(String relativePath) throws IOException;

    /**
     * Copy from another provider. Default: stream-based copy.
     * Override for server-side copy when both providers are same type/bucket.
     */
    default void copyFrom(StorageProvider source, String srcPath, String destPath) throws IOException {
        try (InputStream in = source.read(srcPath)) {
            write(destPath, in, source.size(srcPath));
        }
    }

    /**
     * Provider type identifier (matches storage_volume.provider_type)
     */
    String getProviderType();
}
```

### 2. `src/main/java/com/spax/storage/LocalStorageProvider.java`

Java NIO implementation:

```java
public class LocalStorageProvider implements StorageProvider {
    private final Path basePath;

    public LocalStorageProvider(String basePath) {
        this.basePath = Path.of(basePath);
        // Create base directory if not exists
        try { Files.createDirectories(this.basePath); }
        catch (IOException e) { throw new RuntimeException("Cannot create storage dir: " + basePath, e); }
    }

    @Override
    public void write(String relativePath, InputStream data, long size) throws IOException {
        Path target = resolve(relativePath);
        Files.createDirectories(target.getParent());
        Files.copy(data, target, StandardCopyOption.REPLACE_EXISTING);
    }

    @Override
    public InputStream read(String relativePath) throws IOException {
        return new BufferedInputStream(Files.newInputStream(resolve(relativePath)));
    }

    @Override
    public void delete(String relativePath) throws IOException {
        Files.deleteIfExists(resolve(relativePath));
    }

    @Override
    public boolean exists(String relativePath) throws IOException {
        return Files.exists(resolve(relativePath));
    }

    @Override
    public long size(String relativePath) throws IOException {
        return Files.size(resolve(relativePath));
    }

    @Override
    public String getProviderType() { return "LOCAL"; }

    private Path resolve(String relativePath) {
        // Security: prevent path traversal
        Path resolved = basePath.resolve(relativePath).normalize();
        if (!resolved.startsWith(basePath)) {
            throw new SecurityException("Path traversal detected: " + relativePath);
        }
        return resolved;
    }

    /**
     * Get available disk space in bytes (for DiskSpaceMonitor)
     */
    public long getAvailableBytes() {
        return basePath.toFile().getFreeSpace();
    }

    public long getTotalBytes() {
        return basePath.toFile().getTotalSpace();
    }
}
```

### 3. `src/main/java/com/spax/storage/JCloudStorageProvider.java`

Apache jclouds implementation — 1 class cho ALL cloud providers (S3, GCS, Azure, MinIO):

```java
public class JCloudStorageProvider implements StorageProvider, AutoCloseable {
    private final BlobStore blobStore;
    private final String bucket;
    private final String basePath;   // prefix in bucket
    private final String providerType;
    private final BlobStoreContext context;

    /**
     * @param providerType  jclouds provider: "aws-s3", "google-cloud-storage", "azureblob", "s3"
     * @param identity      access key / service account
     * @param credential    secret key / JSON key
     * @param bucket        bucket/container name
     * @param basePath      prefix path within bucket (can be empty string)
     * @param endpoint      custom endpoint for MinIO/S3-compatible (nullable)
     * @param region        cloud region (nullable)
     */
    public JCloudStorageProvider(String providerType, String identity, String credential,
                                  String bucket, String basePath, String endpoint, String region) {
        this.providerType = providerType;
        this.bucket = bucket;
        this.basePath = basePath;

        Properties props = new Properties();
        if (endpoint != null) {
            props.setProperty(S3Constants.PROPERTY_S3_VIRTUAL_HOST_BUCKETS, "false");
        }
        if (region != null) {
            props.setProperty(LocationConstants.PROPERTY_REGION, region);
        }

        ContextBuilder builder = ContextBuilder.newBuilder(providerType)
            .credentials(identity, credential)
            .overrides(props);

        if (endpoint != null) {
            builder.endpoint(endpoint);
        }

        this.context = builder.buildView(BlobStoreContext.class);
        this.blobStore = context.getBlobStore();

        // Create container if not exists
        if (!blobStore.containerExists(bucket)) {
            blobStore.createContainerInLocation(null, bucket);
        }
    }

    @Override
    public void write(String relativePath, InputStream data, long size) {
        String key = toKey(relativePath);
        Blob blob = blobStore.blobBuilder(key)
            .payload(data)
            .contentLength(size >= 0 ? size : null)
            .build();
        blobStore.putBlob(bucket, blob);
    }

    @Override
    public InputStream read(String relativePath) {
        Blob blob = blobStore.getBlob(bucket, toKey(relativePath));
        if (blob == null) throw new NoSuchFileException(relativePath);
        return blob.getPayload().openStream();
    }

    @Override
    public void delete(String relativePath) {
        blobStore.removeBlob(bucket, toKey(relativePath));
    }

    @Override
    public boolean exists(String relativePath) {
        return blobStore.blobExists(bucket, toKey(relativePath));
    }

    @Override
    public long size(String relativePath) {
        BlobMetadata meta = blobStore.blobMetadata(bucket, toKey(relativePath));
        return meta != null ? meta.getContentMetadata().getContentLength() : -1;
    }

    @Override
    public String getProviderType() { return providerType; }

    @Override
    public void close() { context.close(); }

    private String toKey(String relativePath) {
        return basePath.isEmpty() ? relativePath : basePath + "/" + relativePath;
    }
}
```

### 4. `src/main/java/com/spax/storage/StoreResult.java`

Value object trả về sau khi store:

```java
public record StoreResult(
    int volumeId,
    String storagePath,  // relative path within volume
    long fileSize        // bytes written
) {}
```

### 5. `src/main/java/com/spax/storage/StorageService.java`

Facade interface (higher-level, biết về volumeId):

```java
public interface StorageService {
    /**
     * Store bytes into appropriate volume.
     * VolumeManager chọn volume, StoragePathResolver tạo path.
     */
    StoreResult store(String tenantCode, byte[] data, org.dcm4che3.data.Attributes dicomAttrs);

    /**
     * Read file from specific volume.
     */
    InputStream retrieve(int volumeId, String storagePath);

    /**
     * Delete file from specific volume.
     */
    void delete(int volumeId, String storagePath);

    /**
     * Copy file between volumes (for lifecycle migration).
     */
    void migrate(int sourceVolumeId, String sourcePath, int targetVolumeId, String targetPath);
}
```

**Note**: `StorageService` implementation sẽ được làm ở SPEC-05 (cần VolumeManager).

## Lưu ý quan trọng
- `LocalStorageProvider` dùng `Path.normalize()` + kiểm tra prefix để prevent path traversal
- `JCloudStorageProvider` là stateful (giữ connection pool) — nên tạo 1 lần per volume và cache
- Với MinIO: dùng `providerType="s3"` + `endpoint="http://minio:9000"`
- `write()` dùng `REPLACE_EXISTING` — ingest lại cùng file thì ghi đè (idempotent)
- IOException phải được propagate (không nuốt exception ở tầng này)

## Kiểm tra thành công
- Unit test `LocalStorageProvider`:
  - write → read → xác nhận content đúng
  - delete → exists() trả false
  - path traversal `../` → SecurityException
- `JCloudStorageProvider` với MinIO (nếu có): tương tự test trên với real S3-compatible endpoint
