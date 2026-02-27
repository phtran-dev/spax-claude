# SPEC-18: Health Endpoint

## Mục tiêu
Implement `/api/v1/health` endpoint cho monitoring và load balancer health check. Kiểm tra tất cả critical dependencies.

## Dependencies
- Tất cả các SPEC trước (thực hiện sau cùng)
- SPEC-05 (VolumeManager)
- SPEC-06 (IngestQueue)

## Files cần tạo

### 1. `src/main/java/com/spax/admin/HealthController.java`

```java
@RestController
@RequestMapping("/api/v1")
public class HealthController {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private StringRedisTemplate redisTemplate;
    @Autowired private VolumeManager volumeManager;
    @Autowired private DiskSpaceMonitor diskSpaceMonitor;
    @Autowired private IndexingConsumer indexingConsumer;

    /**
     * GET /api/v1/health
     *
     * Returns:
     * 200 OK:         All systems healthy
     * 200 OK (degraded): Some non-critical issues, but serving requests
     * 503 Service Unavailable: Critical failure (DB down, no storage)
     *
     * Response:
     * {
     *   "status": "UP" | "DEGRADED" | "DOWN",
     *   "checks": {
     *     "database": { "status": "UP", "latencyMs": 3 },
     *     "redis":    { "status": "UP", "latencyMs": 1 },
     *     "storage":  { "status": "UP", "volumes": [...] },
     *     "indexer":  { "status": "UP", "pendingMessages": 0 }
     *   }
     * }
     */
    @GetMapping("/health")
    public ResponseEntity<Map<String, Object>> health() {
        Map<String, Object> checks = new LinkedHashMap<>();
        List<String> criticalFailures = new ArrayList<>();
        List<String> warnings = new ArrayList<>();

        // ── Database check ──────────────────────────────────────────────
        checks.put("database", checkDatabase(criticalFailures));

        // ── Redis check ─────────────────────────────────────────────────
        checks.put("redis", checkRedis(warnings));

        // ── Storage check ───────────────────────────────────────────────
        checks.put("storage", checkStorage(warnings));

        // ── Indexer check ───────────────────────────────────────────────
        checks.put("indexer", checkIndexer(warnings));

        // ── Overall status ──────────────────────────────────────────────
        String status;
        int httpStatus;
        if (!criticalFailures.isEmpty()) {
            status = "DOWN";
            httpStatus = 503;
        } else if (!warnings.isEmpty()) {
            status = "DEGRADED";
            httpStatus = 200;
        } else {
            status = "UP";
            httpStatus = 200;
        }

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("status", status);
        response.put("checks", checks);
        response.put("timestamp", Instant.now().toString());

        if (!criticalFailures.isEmpty()) response.put("criticalFailures", criticalFailures);
        if (!warnings.isEmpty()) response.put("warnings", warnings);

        return ResponseEntity.status(httpStatus).body(response);
    }

    private Map<String, Object> checkDatabase(List<String> criticalFailures) {
        long start = System.currentTimeMillis();
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            long latency = System.currentTimeMillis() - start;

            // Also check tenant count
            int tenantCount = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM public.tenant WHERE active = true", Integer.class
            );

            return Map.of("status", "UP", "latencyMs", latency, "activeTenants", tenantCount);
        } catch (Exception e) {
            criticalFailures.add("Database: " + e.getMessage());
            return Map.of("status", "DOWN", "error", e.getMessage());
        }
    }

    private Map<String, Object> checkRedis(List<String> warnings) {
        long start = System.currentTimeMillis();
        try {
            redisTemplate.getConnectionFactory().getConnection().ping();
            long latency = System.currentTimeMillis() - start;
            return Map.of("status", "UP", "latencyMs", latency);
        } catch (Exception e) {
            warnings.add("Redis unavailable (ingest queue degraded): " + e.getMessage());
            return Map.of("status", "DOWN", "error", e.getMessage());
        }
    }

    private Map<String, Object> checkStorage(List<String> warnings) {
        Map<String, Object> storageCheck = new LinkedHashMap<>();
        List<Map<String, Object>> volumeStatus = new ArrayList<>();
        boolean anyActiveVolume = false;

        List<Map<String, Object>> volumes = jdbcTemplate.queryForList(
            "SELECT id, code, provider_type, tier, status FROM public.storage_volume WHERE status != 'OFFLINE'"
        );

        for (Map<String, Object> vol : volumes) {
            Map<String, Object> volInfo = new LinkedHashMap<>();
            volInfo.put("code", vol.get("code"));
            volInfo.put("tier", vol.get("tier"));
            volInfo.put("status", vol.get("status"));

            try {
                int volId = (int) vol.get("id");
                if ("LOCAL".equals(vol.get("provider_type"))) {
                    LocalStorageProvider local = (LocalStorageProvider) volumeManager.getProvider(volId);
                    double percentFree = (local.getAvailableBytes() * 100.0) / local.getTotalBytes();
                    volInfo.put("diskPercentFree", String.format("%.1f%%", percentFree));

                    if (percentFree < 10) {
                        warnings.add("Volume " + vol.get("code") + " has only " + String.format("%.1f%%", percentFree) + " free");
                    }
                }

                if ("ACTIVE".equals(vol.get("status"))) anyActiveVolume = true;
                volInfo.put("accessible", true);

            } catch (Exception e) {
                volInfo.put("accessible", false);
                volInfo.put("error", e.getMessage());
                warnings.add("Volume " + vol.get("code") + " unreachable: " + e.getMessage());
            }

            volumeStatus.add(volInfo);
        }

        storageCheck.put("status", anyActiveVolume ? "UP" : "DEGRADED");
        storageCheck.put("volumes", volumeStatus);
        storageCheck.put("ingestBlocked", diskSpaceMonitor.isIngestBlocked());

        return storageCheck;
    }

    private Map<String, Object> checkIndexer(List<String> warnings) {
        boolean running = indexingConsumer.isRunning();
        if (!running) {
            warnings.add("IndexingConsumer is not running");
        }

        return Map.of("status", running ? "UP" : "DOWN");
    }
}
```

**Lưu ý**: Cần thêm `isIngestBlocked()` public method vào `DiskSpaceMonitor`:
```java
// DiskSpaceMonitor.java
public boolean isIngestBlocked() { return ingestBlocked; }
```

## Endpoint Summary

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/health` | Full health check |

## Lưu ý quan trọng
- Endpoint không yêu cầu authentication (dùng được bởi load balancer)
- DB failure = critical (503), Redis failure = warning (200 DEGRADED)
- Latency check đơn giản: chỉ `SELECT 1`, không query large tables
- Ingest blocked = warning (hệ thống vẫn serve QIDO/WADO)

## Kiểm tra thành công
- Normal: `GET /api/v1/health` → `{"status": "UP"}`
- Stop Redis: → `{"status": "DEGRADED", "warnings": ["Redis unavailable..."]}`
- Stop PostgreSQL: → `{"status": "DOWN", "criticalFailures": ["Database: ..."], HTTP 503}`
- Low disk: → `{"status": "DEGRADED", "warnings": ["Volume local-nvme has only 8.5% free"]}`
