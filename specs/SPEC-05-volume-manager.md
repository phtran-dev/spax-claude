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
    @Autowired private StoragePathResolver storagePathResolver;

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

        // Clear path format cache (volume templates may have changed)
        storagePathResolver.clearCache();
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

Wrapper quanh dcm4che `AttributesFormat` (từ `dcm4che-core`). `AttributesFormat` là **thread-safe và immutable** sau construction — cache instance để tránh parse template mỗi lần gọi.

```java
@Service
public class StoragePathResolver {

    @Value("${spax.storage.default-path-template}")
    private String defaultTemplate;

    /**
     * Cache: template string → compiled AttributesFormat.
     * AttributesFormat immutable sau construction, thread-safe cho concurrent format() calls.
     * ConcurrentHashMap đảm bảo mỗi template chỉ parse 1 lần.
     */
    private final Map<String, AttributesFormat> formatCache = new ConcurrentHashMap<>();

    private volatile AttributesFormat defaultFormat;

    @PostConstruct
    void init() {
        defaultFormat = new AttributesFormat(defaultTemplate);
        formatCache.put(defaultTemplate, defaultFormat);
    }

    /**
     * Resolve storage path for a DICOM instance.
     *
     * @param volume         the target storage volume (có thể có custom path_template)
     * @param dicomAttrs     DICOM Attributes từ DicomParser
     * @param tenantCode     tenant code (prepended to path)
     * @return relative path within volume, e.g. "hospital_a/2026/02/26/a1b2c3d4/e5f6a7b8/c9d0e1f2"
     */
    public String resolve(StorageVolume volume, Attributes dicomAttrs, String tenantCode) {
        AttributesFormat formatter = getFormat(volume);
        String relativePath = formatter.format(dicomAttrs);

        // Prepend tenant code — SPAX convention, dcm4chee không có
        return tenantCode + "/" + relativePath;
    }

    /**
     * Get or create cached AttributesFormat for volume's template.
     */
    private AttributesFormat getFormat(StorageVolume volume) {
        String template = volume.getPathTemplate();
        if (template == null || template.isBlank()) {
            return defaultFormat;
        }
        return formatCache.computeIfAbsent(template, AttributesFormat::new);
    }

    /**
     * Clear format cache khi admin thay đổi volume template.
     * Gọi từ VolumeController sau update volume.
     */
    public void clearCache() {
        formatCache.clear();
        defaultFormat = new AttributesFormat(defaultTemplate);
        formatCache.put(defaultTemplate, defaultFormat);
    }

    /**
     * Validate template contains SOP UID reference (uniqueness guarantee).
     * Called when creating/updating a storage volume.
     *
     * Cũng validate template hợp lệ bằng cách thử compile.
     */
    public void validateTemplate(String template) {
        if (template == null) return; // null = use default

        // Must contain {00080018} (SOP Instance UID) or a hash of it
        if (!template.contains("{00080018")) {
            throw new IllegalArgumentException(
                "Path template must contain {00080018} (SOP Instance UID) to ensure uniqueness"
            );
        }

        // Validate syntax: thử compile, throw IllegalArgumentException nếu sai
        try {
            new AttributesFormat(template);
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException("Invalid path template syntax: " + template, e);
        }
    }
}
```

### dcm4che `AttributesFormat` — Tham khảo đầy đủ

Source: `org.dcm4che3.util.AttributesFormat` (dcm4che-core 5.31.x)

**Cú pháp**: `{tagOrKeyword[,type[,options]]}`

#### Tag references
| Cú pháp | Mô tả | Ví dụ output |
|---------|--------|-------------|
| `{00080018}` | SOP Instance UID — raw string | `1.2.840.113619.2.55.1762` |
| `{0020000D}` | Study Instance UID — raw string | `1.2.840.113619.2.55.3` |
| `{0020000E}` | Series Instance UID — raw string | `1.2.840.113619.2.55.7` |
| `{00100020}` | Patient ID — raw string | `PAT001` |
| `{00080008[1]}` | ImageType value index 1 (multi-valued) | `PRIMARY` |

#### Type modifiers
| Type | Cú pháp | Output | Chi tiết |
|------|---------|--------|----------|
| `hash` | `{0020000D,hash}` | 8 hex chars | Java `String.hashCode()` → `TagUtils.toHexString()`. Nhanh, deterministic, nhưng collision possible |
| `md5` | `{00080018,md5}` | 26 chars base32 | MD5 digest → custom alphabet `0-9a-v`. Ít collision hơn hash |
| `upper` | `{00100010,upper}` | UPPERCASE | `toUpperCase()` |
| `slice` | `{00100020,slice,0,5}` | Substring | `slice,start[,end]` — hỗ trợ negative index |
| `number` | `{00200013,number}` | Double format | Kết hợp MessageFormat number pattern |
| `offset` | `{00200011,offset,100}` | Int + offset | `getInt() + 100` |
| `urlencoded` | `{00100010,urlencoded}` | URL encode | `URLEncoder.encode(s, "UTF-8")` |

#### Date/Time formatting
| Cú pháp | Mô tả | Output |
|---------|--------|--------|
| `{now,date,yyyy/MM/dd}` | Current date (ingest time) | `2026/02/28` |
| `{now,date,yyyy/MM}` | Year/month only | `2026/02` |
| `{now,time,HH}` | Current hour | `14` |
| `{00080020,date,yyyy/MM/dd}` | StudyDate (DICOM DA tag) | `2025/11/15` |
| `{now,date-P1M,yyyy/MM/dd}` | 1 month ago (ISO 8601 Period) | `2026/01/28` |
| `{now,time+PT5H,HH}` | 5 hours from now (Duration) | `19` |

#### Random/UUID generators
| Cú pháp | Output | Khi nào dùng |
|---------|--------|-------------|
| `{rnd}` | 8 hex chars random | Tránh collision thêm |
| `{rnd,uuid}` | UUID v4 | File name unique toàn cầu |
| `{rnd,uid}` | DICOM UID format | Tạo UID mới |

#### Missing value behavior
| Type | Tag missing → | Hệ quả |
|------|--------------|---------|
| `none`, `upper` | `""` (empty string) | Path có segment rỗng |
| `hash`, `md5`, `urlencoded` | `null` | Segment bị bỏ qua trong MessageFormat |
| `date`, `time` | `new Date()` (current) | Dùng thời gian hiện tại |
| `number`, `offset` | `0` / `0 + offset` | Giá trị mặc định |

### Default template và ví dụ

```yaml
# application.yml
spax:
  storage:
    default-path-template: "{now,date,yyyy/MM/dd}/{0020000D,hash}/{0020000E,hash}/{00080018,hash}"
```

**Ví dụ full path** (với tenant prepend):
```
hospital_a/2026/02/28/a1b2c3d4/e5f6a7b8/c9d0e1f2
│          │          │         │         └─ SOP UID hash (8 hex)
│          │          │         └─ Series UID hash (8 hex)
│          │          └─ Study UID hash (8 hex)
│          └─ Ingest date
└─ Tenant code (SPAX prepend, không phải dcm4che)
```

**Đặc điểm hash (Java hashCode)**:
- 8 hex chars = 32-bit signed int → ~4.3 tỉ giá trị → collision rate thấp cho UID strings
- Deterministic: cùng UID → cùng hash → file path ổn định qua restart
- dcm4chee dùng mặc định, production-proven trên hàng triệu studies

**Khi nào dùng `md5` thay `hash`?**
- `md5`: 26 chars, collision gần như impossible. Dùng khi cần absolute uniqueness (ví dụ: SOP UID làm filename cuối cùng, không có DB dedup)
- `hash`: 8 chars, gọn hơn. Đủ tốt khi kết hợp nhiều levels (`date/study/series/sop`) — collision ở 1 level được disambiguate bởi level khác

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
- `VolumeManager.reload()` phải thread-safe (synchronized + volatile). Gọi `storagePathResolver.clearCache()` để invalidate compiled templates
- Provider instances được cache — không tạo lại connection pool mỗi request
- `getActiveWriteVolume()` chỉ trả tier HOT. Lifecycle service sẽ handle migration sang WARM/COLD
- `StoragePathResolver`:
  - Cache `AttributesFormat` instances — `new AttributesFormat(template)` parse template, tốn CPU. Instance immutable + thread-safe → cache vĩnh viễn
  - `resolve()` inject `Attributes` trực tiếp từ dcm4che (không cần convert)
  - Prepend `tenantCode/` trước path — convention riêng SPAX, dcm4chee không có
  - `validateTemplate()` kiểm tra cả syntax (compile thử) lẫn logic (phải có `{00080018}`)
- Path traversal đã được handle ở `LocalStorageProvider` layer
- `AttributesFormat` dùng `MessageFormat` nội bộ → tự động sanitize special chars trong output

## Kiểm tra thành công
- Thêm 1 LOCAL volume qua DB, call `VolumeManager.getActiveWriteVolume("HOT")` → trả volume đó
- `StorageServiceImpl.store()` → file xuất hiện trên disk
- `StorageServiceImpl.retrieve()` → đọc được file vừa write
- `StoragePathResolver` với default template → path format đúng `{tenantCode}/yyyy/MM/dd/{hash}/{hash}/{hash}`
- Volume với custom template `{0020000D}/{0020000E}/{00080018,md5}` → path format khác, file lưu đúng chỗ
- `validateTemplate("{rnd}")` → throw exception (thiếu SOP UID)
- `validateTemplate("{00080018,invalid}")` → throw exception (syntax lỗi)
