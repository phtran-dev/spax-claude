# SPEC-17: Self-Management Services (AutoPartition + DiskMonitor + AutoRecovery)

## Mục tiêu
Implement 3 background services cho zero-touch operation tại 1000 on-premise sites.

## Dependencies
- SPEC-02 (instance table partition structure)
- SPEC-03 (TenantManagementService — biết tenant list)
- SPEC-05 (VolumeManager, LocalStorageProvider)

## Files cần tạo

### 1. `src/main/java/com/spax/selfmanage/AutoPartitionService.java`

Tự động tạo monthly partitions cho `instance` table, 12 tháng trước:

```java
@Service
public class AutoPartitionService {

    @Autowired private JdbcTemplate jdbcTemplate;

    @Value("${spax.partition.months-ahead:12}")
    private int monthsAhead;

    /**
     * Run daily at 2:30 AM.
     * Creates instance partitions for all tenants for the next 12 months.
     */
    @Scheduled(cron = "0 30 2 * * *")
    public void ensurePartitions() {
        log.info("AutoPartitionService: checking partitions...");

        List<String> tenants = jdbcTemplate.queryForList(
            "SELECT code FROM public.tenant WHERE active = true", String.class
        );

        for (String tenantCode : tenants) {
            try {
                ensurePartitionsForTenant(tenantCode);
            } catch (Exception e) {
                log.error("Failed to create partitions for tenant {}: {}", tenantCode, e.getMessage());
                // Continue with other tenants
            }
        }

        log.info("AutoPartitionService: done");
    }

    /**
     * Public method so TenantManagementService can call it for new tenants.
     */
    public void ensurePartitionsForTenant(String tenantCode) {
        LocalDate now = LocalDate.now();

        for (int i = 0; i <= monthsAhead; i++) {
            LocalDate month = now.plusMonths(i);
            String partitionName = "instance_" + month.format(DateTimeFormatter.ofPattern("yyyy_MM"));
            String firstDay = month.withDayOfMonth(1).toString();
            String firstDayNextMonth = month.plusMonths(1).withDayOfMonth(1).toString();

            try {
                // SET search_path for this tenant
                jdbcTemplate.execute("SET search_path TO tenant_" + sanitize(tenantCode) + ", public");

                // CREATE TABLE ... IF NOT EXISTS (idempotent)
                jdbcTemplate.execute(String.format("""
                    CREATE TABLE IF NOT EXISTS %s
                    PARTITION OF instance
                    FOR VALUES FROM ('%s') TO ('%s')
                    """, partitionName, firstDay, firstDayNextMonth));

                // Create indexes on partition (PostgreSQL propagates parent indexes automatically for new partitions)
                // But explicit creation is safer for older PostgreSQL versions
                // Actually PostgreSQL 11+ automatically creates indexes for new partitions — skip explicit

            } finally {
                jdbcTemplate.execute("SET search_path TO public");
            }
        }
    }

    private String sanitize(String code) {
        if (!code.matches("[a-zA-Z0-9_]+")) throw new IllegalArgumentException("Invalid code: " + code);
        return code;
    }
}
```

### 2. `src/main/java/com/spax/selfmanage/DiskSpaceMonitor.java`

Monitor disk space, reject ingest khi disk gần đầy:

```java
@Component
public class DiskSpaceMonitor {

    @Autowired private VolumeManager volumeManager;

    @Value("${spax.storage.disk-space-threshold-mb:5120}")
    private long thresholdMb;

    // Thread-safe flag read by IngestService
    private volatile boolean ingestBlocked = false;

    /**
     * Check disk space every 5 minutes.
     */
    @Scheduled(fixedDelay = 300_000)
    public void checkDiskSpace() {
        boolean anyBlocked = false;

        // Check all LOCAL volumes
        for (StorageVolume vol : getLocalVolumes()) {
            try {
                StorageProvider provider = volumeManager.getProvider(vol.getId());
                if (!(provider instanceof LocalStorageProvider local)) continue;

                long availableBytes = local.getAvailableBytes();
                long availableMb = availableBytes / (1024 * 1024);
                long totalMb = local.getTotalBytes() / (1024 * 1024);
                double percentFree = (availableBytes * 100.0) / local.getTotalBytes();

                if (percentFree < 5.0) {
                    log.error("CRITICAL: Volume {} has only {}MB free ({:.1f}%). BLOCKING ALL INGEST.",
                        vol.getCode(), availableMb, percentFree);
                    anyBlocked = true;
                } else if (percentFree < 10.0) {
                    log.warn("WARNING: Volume {} has {}MB free ({:.1f}%). Ingest blocked.",
                        vol.getCode(), availableMb, percentFree);
                    anyBlocked = true;
                } else if (percentFree < 20.0) {
                    log.warn("NOTICE: Volume {} has {}MB free ({:.1f}%). Consider adding storage.",
                        vol.getCode(), availableMb, percentFree);
                }

                // Update used_bytes in DB for reporting
                jdbcTemplate.update(
                    "UPDATE public.storage_volume SET used_bytes = ? WHERE id = ?",
                    local.getTotalBytes() - availableBytes, vol.getId()
                );

            } catch (Exception e) {
                log.error("Failed to check disk space for volume {}: {}", vol.getId(), e.getMessage());
            }
        }

        // Block or unblock ingest based on disk status
        boolean wasBlocked = this.ingestBlocked;
        this.ingestBlocked = anyBlocked;

        if (!wasBlocked && anyBlocked) {
            log.error("Ingest BLOCKED due to low disk space");
        } else if (wasBlocked && !anyBlocked) {
            log.info("Ingest UNBLOCKED — disk space recovered");
        }
    }

    /**
     * Called by IngestService before accepting new files.
     * @throws StorageException with HTTP 507 if disk full
     */
    public void checkIngestAllowed() {
        if (ingestBlocked) {
            throw new StorageException("Insufficient storage space. Ingest temporarily disabled. (HTTP 507)");
        }
    }

    private List<StorageVolume> getLocalVolumes() {
        // Get from VolumeManager or directly from DB
        // ...
        return List.of(); // placeholder
    }

    @Autowired private JdbcTemplate jdbcTemplate;
}
```

**IngestService phải gọi `diskSpaceMonitor.checkIngestAllowed()` trước khi accept file.** HTTP 507 = Insufficient Storage.

### 3. `src/main/java/com/spax/selfmanage/AutoRecoveryService.java`

Watchdog cho `IndexingConsumer` và DB connections:

```java
@Component
public class AutoRecoveryService {

    @Autowired private IndexingConsumer indexingConsumer;
    @Autowired private JdbcTemplate jdbcTemplate;

    private int consecutiveFailures = 0;
    private static final int MAX_IMMEDIATE_RETRIES = 3;

    /**
     * Health check every 30 seconds.
     * Restarts indexer if it's dead.
     */
    @Scheduled(fixedDelay = 30_000)
    public void checkHealth() {
        // 1. Check DB connectivity
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            consecutiveFailures = 0;  // reset on success
        } catch (Exception e) {
            consecutiveFailures++;
            log.error("DB connection failed ({}/{}): {}", consecutiveFailures, MAX_IMMEDIATE_RETRIES, e.getMessage());

            if (consecutiveFailures > MAX_IMMEDIATE_RETRIES) {
                log.error("DB unreachable, waiting for recovery...");
                // Spring will retry datasource.getConnection() via HikariCP's own retry
            }
            return;
        }

        // 2. Check IndexingConsumer is running
        if (!indexingConsumer.isRunning()) {
            log.warn("IndexingConsumer is not running, attempting restart...");
            try {
                indexingConsumer.restart();
                log.info("IndexingConsumer restarted successfully");
            } catch (Exception e) {
                log.error("Failed to restart IndexingConsumer: {}", e.getMessage());
            }
        }
    }
}
```

**IndexingConsumer** cần expose `isRunning()` và `restart()` methods:

```java
// Add to IndexingConsumer.java:
private volatile boolean running = true;
private volatile boolean crashed = false;

public boolean isRunning() {
    return running && !crashed;
}

public void restart() {
    crashed = false;
    running = true;
    // Submit new consumer loop tasks
    for (int i = 0; i < consumerThreads; i++) {
        executor.submit(this::consumeLoop);
    }
}

// Modify consumeLoop to catch and mark crashed:
private void consumeLoop() {
    try {
        while (running) {
            // ... existing logic ...
        }
    } catch (Exception e) {
        log.error("Consumer loop fatal error", e);
        crashed = true;
    }
}
```

## Lưu ý quan trọng
- `AutoPartitionService` phải run khi tenant mới được tạo (gọi từ `TenantManagementService.createTenant()`)
- `DiskSpaceMonitor.ingestBlocked` là volatile flag — multi-thread safe read
- Partition creation là idempotent (`IF NOT EXISTS`) — safe to run multiple times
- AutoRecovery không retry vô hạn — sau 3 consecutive failures → log error, đợi HikariCP tự recover
- Disk threshold: 10% → block ingest, 5% → CRITICAL (log error level), 20% → WARNING

## Kiểm tra thành công
- Stop PostgreSQL → HikariCP logs retry, AutoRecovery logs DB failure
- Restart PostgreSQL → kết nối được recover tự động, indexer tiếp tục
- Fill disk đến > 90% → ingest bị block (507 response)
- Tạo tenant mới → partitions tự động tạo cho 12 tháng
- Kill consumer thread → AutoRecovery restart trong 30s
