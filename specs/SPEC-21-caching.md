# SPEC-21: Caching Layer — Spring Cache + EhCache / Redis

## Mục tiêu
Caching layer tổng quát cho SPAX, dùng **Spring Cache abstraction** (`@Cacheable`, `@CacheEvict`, `@CachePut`). Backend có thể **cấu hình** giữa EhCache (local, mặc định) hoặc Redis (distributed) tùy từng bệnh viện.

## Tại sao cần cache?
OHIF Viewer mở 1 ca CT 1000 ảnh → 1000 HTTP requests tới WADO-RS. Mỗi request query bảng `instance` (partitioned monthly, scan 12+ partitions). Không cache = 12,000 index scans/series. Với cache = 12 index scans (1 batch query duy nhất).

Ngoài WADO-RS, còn nhiều data khác cần cache: series metadata lookup, active tenants list, series by study, lifecycle rules.

## Dependencies
- SPEC-01 (project setup — thêm dependencies)
- SPEC-02 (database schema — tables + indexes)
- SPEC-03 (TenantContext — multi-tenant cache keys)
- SPEC-05 (VolumeManager — đã có manual cache, giữ nguyên)

## Thiết kế

### 1. Spring Cache Abstraction — Configurable Backend

```yaml
# application.yml
spax:
  cache:
    type: ehcache   # ehcache (default) | redis
```

- **EhCache** (default): local in-memory, zero network overhead. Phù hợp single-instance on-premise (đa số 1000 sites).
- **Redis**: distributed, shared across JVM instances. Phù hợp multi-instance deployments hoặc sites cần persistence qua restart.
- `VolumeManager` giữ nguyên `ConcurrentHashMap` — không đưa vào Spring Cache (lifecycle riêng: reload on admin action).

### 2. Cache Names & Config

| Cache Name | Mô tả | Key Pattern | TTL | Max Entries | Heap/Off-heap |
|------------|--------|-------------|-----|-------------|---------------|
| `instance-locations` | sopUid → (volumeId, storagePath). **Batch load theo series.** | `{tenantCode}:{seriesUid}` | 30 min (after access) | 500 | 200 MB heap |
| `series-metadata-lookup` | seriesUid → (metadataPath, metadataVolumeId) | `{tenantCode}:{seriesUid}` | 1 hour | 2,000 | 10 MB heap |
| `active-tenants` | Singleton list of active tenant codes | `all` | 60 sec | 1 | 1 KB |
| `series-by-study` | studyUid → List\<SeriesSummary\> | `{tenantCode}:{studyUid}` | 1 hour | 1,000 | 50 MB heap |
| `lifecycle-rules` | action_type → List\<LifecycleRule\> | `{actionType}` | 6 hours | 10 | 1 MB heap |

### 3. Multi-Tenant Cache Key Strategy

Mọi cache entry liên quan đến tenant data đều dùng **composite key** bắt đầu bằng `tenantCode`:

```java
/**
 * Tạo cache key có tenant prefix.
 * Dùng cho tất cả cache liên quan đến tenant-specific data.
 */
public static String tenantKey(String tenantCode, String id) {
    return tenantCode + ":" + id;
}
```

Lý do:
- Schema-per-tenant → cùng UID có thể tồn tại ở 2 tenants khác nhau (hiếm nhưng possible)
- Redis shared cache → bắt buộc phải phân biệt
- EhCache local → vẫn cần nếu 1 JVM serve nhiều tenants

Cache `active-tenants` và `lifecycle-rules` là global (bảng `public` schema) → không cần tenant prefix.

## Files cần tạo

### 1. `src/main/java/com/spax/config/CacheConfig.java`

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Value("${spax.cache.type:ehcache}")
    private String cacheType;

    /**
     * Chọn CacheManager dựa trên config.
     * EhCache: local, nhanh, không cần infrastructure.
     * Redis: distributed, cần Redis server (đã có cho IngestQueue).
     */
    @Bean
    public CacheManager cacheManager(
            @Autowired(required = false) RedisConnectionFactory redisConnectionFactory) {
        return switch (cacheType) {
            case "redis" -> buildRedisCacheManager(redisConnectionFactory);
            default -> buildEhCacheManager();
        };
    }

    private CacheManager buildEhCacheManager() {
        // Dùng Spring SimpleCacheManager + ConcurrentMapCache cho prototype
        // Production: dùng EhCacheCacheManager với ehcache.xml
        org.ehcache.CacheManager ehcacheManager = CacheManagerBuilder.newCacheManagerBuilder()
            .withCache("instance-locations",
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    String.class, Object.class,
                    ResourcePoolsBuilder.newResourcePoolsBuilder()
                        .heap(500, EntryUnit.ENTRIES))
                .withExpiry(ExpiryPolicyBuilder.timeToIdleExpiration(Duration.ofMinutes(30))))
            .withCache("series-metadata-lookup",
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    String.class, Object.class,
                    ResourcePoolsBuilder.newResourcePoolsBuilder()
                        .heap(2000, EntryUnit.ENTRIES))
                .withExpiry(ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofHours(1))))
            .withCache("active-tenants",
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    String.class, Object.class,
                    ResourcePoolsBuilder.newResourcePoolsBuilder()
                        .heap(1, EntryUnit.ENTRIES))
                .withExpiry(ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofSeconds(60))))
            .withCache("series-by-study",
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    String.class, Object.class,
                    ResourcePoolsBuilder.newResourcePoolsBuilder()
                        .heap(1000, EntryUnit.ENTRIES))
                .withExpiry(ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofHours(1))))
            .withCache("lifecycle-rules",
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    String.class, Object.class,
                    ResourcePoolsBuilder.newResourcePoolsBuilder()
                        .heap(10, EntryUnit.ENTRIES))
                .withExpiry(ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofHours(6))))
            .build(true);

        return new org.springframework.cache.jcache.JCacheCacheManager(
            // Wrap EhCache manager vào Spring CacheManager
            // Hoặc dùng EhCacheCacheManager adapter
        );
    }

    private CacheManager buildRedisCacheManager(RedisConnectionFactory factory) {
        if (factory == null) {
            throw new IllegalStateException("Redis cache requested but RedisConnectionFactory not available");
        }

        // Redis serializer: JSON cho readability + cross-language compatibility
        GenericJackson2JsonRedisSerializer serializer = new GenericJackson2JsonRedisSerializer();

        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(serializer))
            .prefixCacheNameWith("spax:cache:")
            .disableCachingNullValues();

        Map<String, RedisCacheConfiguration> perCacheConfig = Map.of(
            "instance-locations", defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "series-metadata-lookup", defaultConfig.entryTtl(Duration.ofHours(1)),
            "active-tenants", defaultConfig.entryTtl(Duration.ofSeconds(60)),
            "series-by-study", defaultConfig.entryTtl(Duration.ofHours(1)),
            "lifecycle-rules", defaultConfig.entryTtl(Duration.ofHours(6))
        );

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig.entryTtl(Duration.ofMinutes(30)))
            .withInitialCacheConfigurations(perCacheConfig)
            .build();
    }
}
```

### 2. `src/main/java/com/spax/dicomweb/WadoRsCacheService.java`

Service tách riêng cache logic khỏi controller. Controller gọi service, service dùng `@Cacheable`.

```java
@Service
public class WadoRsCacheService {

    private static final Logger log = LoggerFactory.getLogger(WadoRsCacheService.class);

    private final JdbcTemplate jdbcTemplate;

    public WadoRsCacheService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    // ─── Instance Location (batch load theo series) ─────────────────

    /**
     * Find file location cho 1 instance.
     * Cache key = "{tenantCode}:{seriesUid}" → batch load toàn bộ series.
     *
     * Không dùng @Cacheable trực tiếp vì cần:
     * 1. Batch load (load ALL instances, return 1)
     * 2. Cache key khác return value (key = series, return = 1 instance)
     * → Dùng CacheManager programmatically.
     */
    public InstanceLocation findInstance(CacheManager cacheManager,
                                         String tenantCode, String seriesUid, String sopUid) {
        String key = tenantCode + ":" + seriesUid;
        org.springframework.cache.Cache cache = cacheManager.getCache("instance-locations");

        SeriesInstanceLocations locations = cache.get(key, SeriesInstanceLocations.class);
        if (locations == null) {
            locations = loadSeriesLocations(seriesUid);
            if (locations != null) {
                cache.put(key, locations);
            }
        }

        return locations != null ? locations.instances().get(sopUid) : null;
    }

    /**
     * Lấy tất cả instance locations của 1 series.
     * Cũng populate cache cho subsequent frame requests.
     */
    public List<InstanceLocation> findAllInSeries(CacheManager cacheManager,
                                                   String tenantCode, String seriesUid) {
        String key = tenantCode + ":" + seriesUid;
        org.springframework.cache.Cache cache = cacheManager.getCache("instance-locations");

        SeriesInstanceLocations locations = cache.get(key, SeriesInstanceLocations.class);
        if (locations == null) {
            locations = loadSeriesLocations(seriesUid);
            if (locations != null) {
                cache.put(key, locations);
            }
        }

        return locations != null ? List.copyOf(locations.instances().values()) : List.of();
    }

    /**
     * 2-step batch load (with partition pruning):
     * 1. series_instance_uid → series.id + created_at::date (idx_series_uid)
     * 2. series.id + created_date → all instances (idx_instance_dedup, single partition)
     *
     * Tất cả instances cùng series có cùng created_date = series.created_at::date,
     * nên truyền created_date vào WHERE clause → PostgreSQL prune xuống đúng 1 partition
     * thay vì scan 60+ partitions.
     */
    private SeriesInstanceLocations loadSeriesLocations(String seriesUid) {
        // Step 1: lấy series.id + created_at::date
        List<Map<String, Object>> seriesRows = jdbcTemplate.queryForList(
            "SELECT id, created_at::date AS created_date FROM series WHERE series_instance_uid = ?",
            seriesUid
        );
        if (seriesRows.isEmpty()) return null;

        long seriesFk = ((Number) seriesRows.getFirst().get("id")).longValue();
        java.sql.Date createdDate = (java.sql.Date) seriesRows.getFirst().get("created_date");

        // Step 2: query instances với partition pruning
        List<Map<String, Object>> rows = jdbcTemplate.queryForList("""
            SELECT sop_instance_uid, volume_id, storage_path,
                   transfer_syntax_uid, num_frames
            FROM instance WHERE series_fk = ? AND created_date = ?
            """, seriesFk, createdDate
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
                    row.get("num_frames") != null ? ((Number) row.get("num_frames")).intValue() : 1)
            );
        }

        log.debug("Loaded {} instance locations for series {}", map.size(), seriesUid);
        return new SeriesInstanceLocations(Collections.unmodifiableMap(map));
    }

    // ─── Series Metadata Lookup ─────────────────────────────────────

    /**
     * Lookup metadata path cho series (WADO-RS metadata endpoint).
     * Cache: fast path khi metadata_path đã có.
     */
    @Cacheable(value = "series-metadata-lookup", key = "#tenantCode + ':' + #seriesUid")
    public SeriesMetadataInfo getSeriesMetadataInfo(String tenantCode, String seriesUid) {
        Map<String, Object> row = jdbcTemplate.queryForMap("""
            SELECT s.metadata_volume_id, s.metadata_path,
                   sv.provider_type
            FROM series s
            LEFT JOIN public.storage_volume sv ON sv.id = (
                SELECT i.volume_id FROM instance i
                WHERE i.series_instance_uid = ? LIMIT 1
            )
            WHERE s.series_instance_uid = ?
            """, seriesUid, seriesUid
        );

        return new SeriesMetadataInfo(
            (String) row.get("metadata_path"),
            row.get("metadata_volume_id") != null ? ((Number) row.get("metadata_volume_id")).intValue() : null,
            (String) row.get("provider_type")
        );
    }

    @CacheEvict(value = "series-metadata-lookup", key = "#tenantCode + ':' + #seriesUid")
    public void evictSeriesMetadataInfo(String tenantCode, String seriesUid) {
        // Spring Cache evicts on method call
    }

    // ─── Series by Study ────────────────────────────────────────────

    @Cacheable(value = "series-by-study", key = "#tenantCode + ':' + #studyUid")
    public List<SeriesSummary> getSeriesByStudy(String tenantCode, String studyUid) {
        return jdbcTemplate.query("""
            SELECT series_instance_uid, modality, series_number,
                   series_description, num_instances, body_part
            FROM series WHERE study_instance_uid = ?
            ORDER BY series_number
            """,
            (rs, i) -> new SeriesSummary(
                rs.getString("series_instance_uid"),
                rs.getString("modality"),
                rs.getObject("series_number", Integer.class),
                rs.getString("series_description"),
                rs.getInt("num_instances"),
                rs.getString("body_part")
            ),
            studyUid
        );
    }

    @CacheEvict(value = "series-by-study", key = "#tenantCode + ':' + #studyUid")
    public void evictSeriesByStudy(String tenantCode, String studyUid) {}

    // ─── Invalidation helpers ───────────────────────────────────────

    /**
     * Invalidate instance-locations cache cho 1 series.
     */
    public void evictInstanceLocations(CacheManager cacheManager,
                                        String tenantCode, String seriesUid) {
        org.springframework.cache.Cache cache = cacheManager.getCache("instance-locations");
        if (cache != null) {
            cache.evict(tenantCode + ":" + seriesUid);
        }
    }

    /**
     * Invalidate instance-locations cache cho nhiều series.
     */
    public void evictInstanceLocations(CacheManager cacheManager,
                                        String tenantCode, Collection<String> seriesUids) {
        org.springframework.cache.Cache cache = cacheManager.getCache("instance-locations");
        if (cache != null) {
            for (String uid : seriesUids) {
                cache.evict(tenantCode + ":" + uid);
            }
        }
    }

    // ─── Records ────────────────────────────────────────────────────

    public record InstanceLocation(int volumeId, String storagePath,
                                    String transferSyntaxUid, int numFrames) implements Serializable {}

    public record SeriesInstanceLocations(Map<String, InstanceLocation> instances) implements Serializable {}

    public record SeriesMetadataInfo(String metadataPath, Integer metadataVolumeId,
                                     String providerType) implements Serializable {}

    public record SeriesSummary(String seriesInstanceUid, String modality, Integer seriesNumber,
                                String seriesDescription, int numInstances,
                                String bodyPart) implements Serializable {}
}
```

### 3. `src/main/java/com/spax/tenant/TenantCacheService.java`

```java
@Service
public class TenantCacheService {

    private final JdbcTemplate jdbcTemplate;

    public TenantCacheService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Cacheable(value = "active-tenants", key = "'all'")
    public List<String> getActiveTenantCodes() {
        return jdbcTemplate.queryForList(
            "SELECT code FROM public.tenant WHERE active = true",
            String.class
        );
    }

    @CacheEvict(value = "active-tenants", allEntries = true)
    public void evictActiveTenants() {}
}
```

### 4. `src/main/java/com/spax/lifecycle/LifecycleRuleCacheService.java`

```java
@Service
public class LifecycleRuleCacheService {

    private final JdbcTemplate jdbcTemplate;

    public LifecycleRuleCacheService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Cacheable(value = "lifecycle-rules", key = "#actionType")
    public List<LifecycleRuleDto> getRulesByActionType(String actionType) {
        return jdbcTemplate.query("""
            SELECT id, name, action_type, source_tier, target_tier,
                   condition_type, condition_days, compression_type, enabled
            FROM public.lifecycle_rule
            WHERE enabled = true AND action_type = ?
            ORDER BY id
            """,
            (rs, i) -> new LifecycleRuleDto(
                rs.getLong("id"),
                rs.getString("name"),
                rs.getString("action_type"),
                rs.getString("source_tier"),
                rs.getString("target_tier"),
                rs.getString("condition_type"),
                rs.getInt("condition_days"),
                rs.getString("compression_type"),
                rs.getBoolean("enabled")
            ),
            actionType
        );
    }

    @CacheEvict(value = "lifecycle-rules", allEntries = true)
    public void evictAll() {}

    public record LifecycleRuleDto(
        long id, String name, String actionType,
        String sourceTier, String targetTier,
        String conditionType, int conditionDays,
        String compressionType, boolean enabled
    ) implements Serializable {}
}
```

## Invalidation Matrix

### Khi nào evict cache nào?

| Event | `instance-locations` | `series-metadata-lookup` | `series-by-study` | `active-tenants` | `lifecycle-rules` |
|-------|---------------------|--------------------------|--------------------|-------------------|-------------------|
| **Ingest batch** (IndexingConsumer) | evict affected series | evict affected series | evict affected study | - | - |
| **Lifecycle migration** | evict migrated series | - | - | - | - |
| **DICOM correction** | - | evict corrected series | evict affected study | - | - |
| **Compression** | - | evict compressed series | - | - | - |
| **Study deletion** | evict all series of study | evict all series of study | evict study | - | - |
| **Tenant CRUD** | - | - | - | evict all | - |
| **Lifecycle rule CRUD** | - | - | - | - | evict all |
| **Volume offline** | evict ALL¹ | - | - | - | - |

¹ Volume offline hiếm khi xảy ra → `cache.clear()` toàn bộ `instance-locations` là chấp nhận được.

### Integration points (files cần sửa)

**`IndexingConsumer`** (SPEC-08): sau `batchUpsert()`:
```java
Set<String> affectedSeriesUids = items.stream()
    .map(item -> item.dicom().seriesInstanceUid())
    .collect(Collectors.toSet());
String tenant = TenantContext.getTenantCode();
wadoRsCacheService.evictInstanceLocations(cacheManager, tenant, affectedSeriesUids);
for (String seriesUid : affectedSeriesUids) {
    wadoRsCacheService.evictSeriesMetadataInfo(tenant, seriesUid);
}
// evict study-level cache
Set<String> affectedStudyUids = items.stream()
    .map(item -> item.dicom().studyInstanceUid())
    .collect(Collectors.toSet());
for (String studyUid : affectedStudyUids) {
    wadoRsCacheService.evictSeriesByStudy(tenant, studyUid);
}
```

**`LifecycleService.processMigrationTask()`** (SPEC-16): sau update `instance.volume_id`:
```java
wadoRsCacheService.evictInstanceLocations(cacheManager, tenantCode, seriesUid);
```

**`FileCorrectionJob`** (SPEC-13): sau rebuild metadata:
```java
wadoRsCacheService.evictSeriesMetadataInfo(tenantCode, seriesUid);
wadoRsCacheService.evictSeriesByStudy(tenantCode, studyUid);
```

**`StudyAdminService.deleteStudy()`** (SPEC-14):
```java
// evict all series of this study
for (String seriesUid : seriesUids) {
    wadoRsCacheService.evictInstanceLocations(cacheManager, tenantCode, seriesUid);
    wadoRsCacheService.evictSeriesMetadataInfo(tenantCode, seriesUid);
}
wadoRsCacheService.evictSeriesByStudy(tenantCode, studyUid);
```

**`TenantController`** (SPEC-15): sau create/update/delete tenant:
```java
tenantCacheService.evictActiveTenants();
```

**`LifecycleRuleController`** (SPEC-15/20): sau create/update/delete rule:
```java
lifecycleRuleCacheService.evictAll();
```

## Config mẫu trong `application.yml`

```yaml
spax:
  cache:
    type: ehcache   # ehcache (default, local) | redis (distributed)
```

Khi `type: redis`, Spring Cache dùng cùng Redis connection với IngestQueue nhưng key prefix khác (`spax:cache:` vs `spax:ingest:`). Không conflict.

## Dependencies cần thêm vào `pom.xml`

```xml
<!-- Spring Cache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- EhCache (default backend) -->
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
<dependency>
    <groupId>javax.cache</groupId>
    <artifactId>cache-api</artifactId>
</dependency>
```

Redis dependency (`spring-boot-starter-data-redis`) đã có sẵn trong SPEC-01.

## Memory footprint (EhCache)

| Cache | Max Entries | Avg Entry Size | Max Memory |
|-------|------------|----------------|------------|
| `instance-locations` | 500 series | 300 inst × 400 bytes | ~60 MB |
| `series-metadata-lookup` | 2,000 | ~100 bytes | ~0.2 MB |
| `active-tenants` | 1 | ~1 KB | ~1 KB |
| `series-by-study` | 1,000 studies | ~10 series × 200 bytes | ~2 MB |
| `lifecycle-rules` | 10 | ~500 bytes | ~5 KB |
| **Tổng** | | | **~63 MB** |

Hoàn toàn chấp nhận được trong JVM heap 1-4 GB.

## Lưu ý
- Records implement `Serializable` — bắt buộc cho Redis cache (JSON serialization)
- `instance-locations` dùng programmatic cache (không `@Cacheable`) vì pattern batch-load-return-one không map tự nhiên vào annotation
- **Partition pruning**: `loadSeriesLocations()` query 2-step: (1) `series.id + created_at::date`, (2) `instance WHERE series_fk = ? AND created_date = ?`. Tất cả instances cùng series có cùng `created_date` (= `series.created_at::date`, set khi ingest — xem SPEC-09) → PostgreSQL prune xuống đúng 1 partition thay vì scan tất cả
- Các cache khác dùng `@Cacheable`/`@CacheEvict` annotation — clean, dễ đọc
- EhCache dùng heap-only (không off-heap) cho đơn giản. Off-heap mở rộng sau nếu cần
- `VolumeManager.providerCache` giữ nguyên ConcurrentHashMap — không đưa vào Spring Cache vì lifecycle khác (reload toàn bộ khi admin thay đổi)

## Kiểm tra thành công

### EhCache mode
1. Start app với `spax.cache.type=ehcache` (default)
2. Request WADO-RS frame lần 1 → cache miss → DB query → cache put
3. Request frame lần 2 (cùng series) → cache hit → 0 DB queries
4. Trigger lifecycle migration → cache evict → next request loads fresh
5. `getActiveTenantCodes()` gọi 100 lần trong 60s → chỉ 1 DB query

### Redis mode
1. Start app với `spax.cache.type=redis`
2. Kiểm tra Redis keys: `KEYS spax:cache:*` → thấy cache entries
3. Restart app → cache vẫn còn trong Redis (persistent)
4. Evict → `DEL spax:cache:instance-locations::tenant1:1.2.3.4` → key biến mất
5. Multi-tenant: 2 tenants → keys khác nhau

### Multi-tenant isolation
1. Tenant A request series X → cache key `tenantA:seriesX`
2. Tenant B request series X → cache key `tenantB:seriesX` (khác entry)
3. Evict tenant A → tenant B cache vẫn còn
