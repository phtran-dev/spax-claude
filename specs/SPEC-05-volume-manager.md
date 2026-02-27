# SPEC-05: VolumeManager + StoragePathResolver + StorageServiceImpl

## Mục tiêu
Implement tầng volume management: chọn volume để ghi, resolve storage path từ DICOM attributes, và StorageService facade. Đây là "brain" của storage layer.

## Dependencies
- SPEC-02 (database schema — `storage_volume` table)
- SPEC-04 (StorageProvider SPI, LocalStorageProvider, JCloudStorageProvider)

## Files cần tạo

### 1. `src/main/java/com/spax/storage/StorageVolume.java`

JPA entity map với `public.storage_volume` table:

```java
@Entity
@Table(name = "storage_volume", schema = "public")
public class StorageVolume {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    private String code;
    private String providerType;
    private String basePath;
    private String tier;          // HOT, WARM, COLD
    private String status;        // ACTIVE, READ_ONLY, OFFLINE
    private Integer priority;
    private Long totalBytes;
    private Long usedBytes;
    private String pathTemplate;
    private String cloudEndpoint;
    private String cloudRegion;
    private String cloudBucket;
    private String cloudIdentity;
    private String cloudCredential;
    // getters/setters or use Lombok @Data
}
```

### 2. `src/main/java/com/spax/storage/VolumeManager.java`

Core component — cache volumes, chọn volume để ghi, expose provider instances:

```java
@Component
public class VolumeManager {
    @Autowired private StorageVolumeRepository volumeRepository;

    // Cache: volumeId → StorageProvider instance
    private final Map<Integer, StorageProvider> providerCache = new ConcurrentHashMap<>();

    // Cache: list of volumes by tier (sorted by priority DESC)
    private volatile Map<String, List<StorageVolume>> volumesByTier = new HashMap<>();

    @PostConstruct
    public void init() {
        reload();
    }

    /**
     * Reload volume config từ DB (gọi sau Admin API thay đổi).
     * Thread-safe: replace volatile reference atomically.
     */
    public synchronized void reload() {
        List<StorageVolume> all = volumeRepository.findAll();
        Map<String, List<StorageVolume>> byTier = all.stream()
            .collect(Collectors.groupingBy(StorageVolume::getTier));

        // Sort by priority DESC within each tier
        byTier.values().forEach(list -> list.sort(
            Comparator.comparingInt(StorageVolume::getPriority).reversed()
        ));

        this.volumesByTier = byTier;

        // Initialize providers for new volumes
        all.forEach(v -> providerCache.computeIfAbsent(v.getId(), id -> createProvider(v)));
    }

    /**
     * Get best write volume for given tier.
     * Returns ACTIVE volume with highest priority that has sufficient free space (LOCAL only).
     */
    public StorageVolume getActiveWriteVolume(String tier) {
        List<StorageVolume> volumes = volumesByTier.getOrDefault(tier, List.of());
        return volumes.stream()
            .filter(v -> "ACTIVE".equals(v.getStatus()))
            .filter(v -> hasSpace(v))
            .findFirst()
            .orElseThrow(() -> new StorageException("No active write volume for tier: " + tier));
    }

    /**
     * Get StorageProvider for a specific volume.
     */
    public StorageProvider getProvider(int volumeId) {
        StorageProvider provider = providerCache.get(volumeId);
        if (provider == null) throw new StorageException("Unknown volume: " + volumeId);
        return provider;
    }

    /**
     * Get StorageVolume entity by id.
     */
    public StorageVolume getVolume(int volumeId) {
        return volumeRepository.findById(volumeId)
            .orElseThrow(() -> new StorageException("Volume not found: " + volumeId));
    }

    private boolean hasSpace(StorageVolume volume) {
        if (!"LOCAL".equals(volume.getProviderType())) return true; // cloud = unlimited
        StorageProvider provider = providerCache.get(volume.getId());
        if (provider instanceof LocalStorageProvider local) {
            long freeBytes = local.getAvailableBytes();
            long threshold = 1024L * 1024 * 1024; // 1GB minimum
            return freeBytes > threshold;
        }
        return true;
    }

    private StorageProvider createProvider(StorageVolume v) {
        return switch (v.getProviderType()) {
            case "LOCAL" -> new LocalStorageProvider(v.getBasePath());
            case "aws-s3", "google-cloud-storage", "azureblob", "s3", "b2" ->
                new JCloudStorageProvider(
                    v.getProviderType(), v.getCloudIdentity(), v.getCloudCredential(),
                    v.getCloudBucket(), v.getBasePath(),
                    v.getCloudEndpoint(), v.getCloudRegion()
                );
            default -> throw new StorageException("Unknown provider type: " + v.getProviderType());
        };
    }
}
```

### 3. `src/main/java/com/spax/storage/StoragePathResolver.java`

Wrapper quanh dcm4che `AttributesFormat` để resolve path template:

```java
@Service
public class StoragePathResolver {

    @Value("${spax.storage.default-path-template}")
    private String defaultTemplate;

    /**
     * Resolve storage path for a DICOM instance.
     *
     * @param volume         the target storage volume (có thể có custom path_template)
     * @param dicomAttrs     DICOM Attributes từ DicomParser
     * @param tenantCode     tenant code (prepended to path)
     * @return relative path within volume, e.g. "hospital_a/2026/02/26/a1b2/c3d4/e5f6"
     */
    public String resolve(StorageVolume volume, Attributes dicomAttrs, String tenantCode) {
        String template = volume.getPathTemplate();
        if (template == null || template.isBlank()) {
            template = defaultTemplate;
        }

        // dcm4che AttributesFormat: formats template using DICOM tag values
        AttributesFormat formatter = new AttributesFormat(template);
        String relativePath = formatter.format(dicomAttrs);

        // Prepend tenant code
        return tenantCode + "/" + relativePath;
    }

    /**
     * Validate template contains SOP UID reference (uniqueness guarantee).
     * Called when creating/updating a storage volume.
     */
    public void validateTemplate(String template) {
        if (template == null) return; // null = use default
        // Must contain {00080018} (SOP Instance UID) or a hash of it
        if (!template.contains("{00080018")) {
            throw new IllegalArgumentException(
                "Path template must contain {00080018} (SOP Instance UID) to ensure uniqueness"
            );
        }
    }
}
```

**dcm4che `AttributesFormat` template syntax** (reference):
- `{00080018}` → SOP Instance UID (raw)
- `{0020000D}` → Study Instance UID
- `{0020000E}` → Series Instance UID
- `{00080018,hash}` → Java hashCode của SOP UID (hex int)
- `{now,date,yyyy/MM/dd}` → current date
- `{00080018,md5}` → MD5 base32 hash

Default template: `{now,date,yyyy/MM/dd}/{0020000D,hash}/{0020000E,hash}/{00080018,hash}`

Result example: `hospital_a/2026/02/26/a1b2c3d4/e5f6a7b8/c9d0e1f2`

### 4. `src/main/java/com/spax/storage/StorageServiceImpl.java`

Implement `StorageService` interface (defined in SPEC-04):

```java
@Service
public class StorageServiceImpl implements StorageService {
    @Autowired private VolumeManager volumeManager;
    @Autowired private StoragePathResolver pathResolver;

    @Override
    public StoreResult store(String tenantCode, byte[] data, Attributes dicomAttrs) {
        // Get HOT tier volume (default write tier)
        StorageVolume volume = volumeManager.getActiveWriteVolume("HOT");
        StorageProvider provider = volumeManager.getProvider(volume.getId());

        String path = pathResolver.resolve(volume, dicomAttrs, tenantCode);

        try (InputStream in = new ByteArrayInputStream(data)) {
            provider.write(path, in, data.length);
        }

        return new StoreResult(volume.getId(), path, data.length);
    }

    @Override
    public InputStream retrieve(int volumeId, String storagePath) {
        return volumeManager.getProvider(volumeId).read(storagePath);
    }

    @Override
    public void delete(int volumeId, String storagePath) {
        volumeManager.getProvider(volumeId).delete(storagePath);
    }

    @Override
    public void migrate(int sourceVolumeId, String sourcePath, int targetVolumeId, String targetPath) {
        StorageProvider source = volumeManager.getProvider(sourceVolumeId);
        StorageProvider target = volumeManager.getProvider(targetVolumeId);
        target.copyFrom(source, sourcePath, targetPath);
    }
}
```

### 5. `src/main/java/com/spax/storage/StorageVolumeRepository.java`

```java
@Repository
public interface StorageVolumeRepository extends JpaRepository<StorageVolume, Integer> {
    List<StorageVolume> findByStatusAndTierOrderByPriorityDesc(String status, String tier);
    Optional<StorageVolume> findByCode(String code);
}
```

### 6. Exception class: `src/main/java/com/spax/storage/StorageException.java`

```java
public class StorageException extends RuntimeException {
    public StorageException(String message) { super(message); }
    public StorageException(String message, Throwable cause) { super(message, cause); }
}
```

## Lưu ý quan trọng
- `VolumeManager.reload()` phải thread-safe (synchronized + volatile)
- Provider instances được cache — không tạo lại connection pool mỗi request
- `getActiveWriteVolume()` chỉ trả tier HOT. Lifecycle service sẽ handle migration sang WARM/COLD
- `StoragePathResolver.resolve()` inject `Attributes` trực tiếp từ dcm4che (không cần convert)
- Path traversal đã được handle ở `LocalStorageProvider` layer

## Kiểm tra thành công
- Thêm 1 LOCAL volume qua DB, call `VolumeManager.getActiveWriteVolume("HOT")` → trả volume đó
- `StorageServiceImpl.store()` → file xuất hiện trên disk
- `StorageServiceImpl.retrieve()` → đọc được file vừa write
- `StoragePathResolver` với default template → path format đúng `{tenantCode}/yyyy/MM/dd/...`
