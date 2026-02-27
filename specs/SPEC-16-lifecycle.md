# SPEC-16: Data Lifecycle Service (Rules + Migration Job)

## Mục tiêu
Implement lifecycle rule engine: tự động migrate files giữa storage tiers (HOT → WARM → COLD) theo điều kiện tuổi. Scheduled background job, chạy hàng đêm.

## Dependencies
- SPEC-02 (schema — lifecycle_rule, migration_task tables)
- SPEC-05 (StorageService, VolumeManager)

## Files cần tạo

### 1. `src/main/java/com/spax/lifecycle/LifecycleRule.java`

JPA entity map với `public.lifecycle_rule`:

```java
@Entity
@Table(name = "lifecycle_rule", schema = "public")
public class LifecycleRule {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    private String name;
    private boolean enabled;
    private String actionType;       // "MIGRATE" | "COMPRESS"
    private String sourceTier;
    private String targetTier;       // nullable (not used for COMPRESS)
    private String conditionType;    // "STUDY_AGE_DAYS" | "LAST_ACCESS_DAYS"
    private int conditionValue;      // e.g. 180 (days)
    private boolean deleteSource;
    private String compressionType;  // nullable (used only for COMPRESS)
    private String tenantCode;       // nullable = apply to all tenants
    // getters/setters or Lombok
}
```

### 2. `src/main/java/com/spax/lifecycle/LifecycleService.java`

Core business logic — evaluate rules và create migration tasks:

```java
@Service
public class LifecycleService {

    @Autowired private JdbcTemplate jdbcTemplate;  // public schema
    @Autowired private VolumeManager volumeManager;
    @Autowired private StorageService storageService;

    /**
     * Evaluate all enabled MIGRATE rules.
     * Creates migration_task records for qualifying instances.
     * Called by MigrationJob scheduler.
     */
    public void evaluateMigrateRules() {
        List<LifecycleRule> rules = loadEnabledMigrateRules();

        for (LifecycleRule rule : rules) {
            try {
                evaluateMigrateRule(rule);
            } catch (Exception e) {
                log.error("Failed to evaluate lifecycle rule {}: {}", rule.getId(), e.getMessage());
                // Continue with next rule — don't stop all processing for 1 rule failure
            }
        }
    }

    private void evaluateMigrateRule(LifecycleRule rule) {
        // Find target volume for the target tier
        StorageVolume targetVolume;
        try {
            targetVolume = volumeManager.getActiveWriteVolume(rule.getTargetTier());
        } catch (StorageException e) {
            log.warn("No active write volume for tier {}, skipping rule {}", rule.getTargetTier(), rule.getId());
            return;
        }

        // Find qualifying instances (in source tier, meeting age condition)
        String conditionSql = buildConditionSql(rule);

        // Get all tenants for this rule
        List<String> tenants = getApplicableTenants(rule);

        for (String tenantCode : tenants) {
            // Set tenant schema context
            jdbcTemplate.execute("SET search_path TO tenant_" + tenantCode + ", public");

            try {
                // Query instances in source tier meeting condition
                // Join instance + study to check study_date or last_accessed_at
                String sql = """
                    SELECT i.id, i.volume_id, i.storage_path, i.study_instance_uid
                    FROM instance i
                    JOIN study s ON i.study_instance_uid = s.study_instance_uid
                    JOIN public.storage_volume sv ON i.volume_id = sv.id
                    WHERE sv.tier = ?
                    """ + conditionSql + """
                    ORDER BY i.created_date
                    LIMIT 10000
                    """;

                Object[] args = buildConditionArgs(rule);
                List<Map<String, Object>> qualifying = jdbcTemplate.queryForList(sql, args);

                // Create migration_task records
                for (Map<String, Object> instance : qualifying) {
                    // Check if migration task already exists (idempotent)
                    int existing = jdbcTemplate.queryForObject(
                        """
                        SELECT COUNT(*) FROM public.migration_task
                        WHERE tenant_code = ? AND instance_id = ? AND status IN ('PENDING','IN_PROGRESS','COMPLETED')
                        """,
                        Integer.class, tenantCode, instance.get("id")
                    );

                    if (existing > 0) continue;

                    jdbcTemplate.update("""
                        INSERT INTO public.migration_task
                          (tenant_code, rule_id, instance_id, study_date, source_volume_id, target_volume_id)
                        VALUES (?, ?, ?, CURRENT_DATE, ?, ?)
                        """,
                        tenantCode, rule.getId(), instance.get("id"),
                        instance.get("volume_id"), targetVolume.getId()
                    );
                }

                log.info("Rule {}: queued {} instances for migration in tenant {}",
                    rule.getId(), qualifying.size(), tenantCode);

            } finally {
                jdbcTemplate.execute("SET search_path TO public");
            }
        }
    }

    /**
     * Process pending migration tasks.
     * Called by MigrationJob scheduler (separate from evaluation).
     */
    public void processMigrationTasks() {
        // Get batch of pending tasks
        List<Map<String, Object>> tasks = jdbcTemplate.queryForList("""
            SELECT * FROM public.migration_task
            WHERE status = 'PENDING'
            ORDER BY created_at
            LIMIT 100
            """);

        for (Map<String, Object> task : tasks) {
            processMigrationTask(task);
        }
    }

    private void processMigrationTask(Map<String, Object> task) {
        long taskId = (long) task.get("id");
        String tenantCode = (String) task.get("tenant_code");
        long instanceId = (long) task.get("instance_id");
        int sourceVolumeId = (int) task.get("source_volume_id");
        int targetVolumeId = (int) task.get("target_volume_id");

        // Mark IN_PROGRESS
        jdbcTemplate.update(
            "UPDATE public.migration_task SET status = 'IN_PROGRESS' WHERE id = ?", taskId
        );

        try {
            // Get instance storage path
            jdbcTemplate.execute("SET search_path TO tenant_" + tenantCode + ", public");

            Map<String, Object> instance = jdbcTemplate.queryForMap(
                "SELECT storage_path FROM instance WHERE id = ? AND created_date >= CURRENT_DATE - INTERVAL '3 years'",
                instanceId
            );
            String storagePath = (String) instance.get("storage_path");

            // 1. Copy file to target volume
            storageService.migrate(sourceVolumeId, storagePath, targetVolumeId, storagePath);

            // 2. Verify target exists
            if (!volumeManager.getProvider(targetVolumeId).exists(storagePath)) {
                throw new IOException("File not found at target after copy: " + storagePath);
            }

            // 3. Update instance.volume_id
            jdbcTemplate.update(
                "UPDATE instance SET volume_id = ? WHERE id = ?",
                targetVolumeId, instanceId
            );

            // 4. Delete source (if deleteSource configured)
            boolean deleteSource = (boolean) jdbcTemplate.queryForObject(
                "SELECT delete_source FROM public.lifecycle_rule WHERE id = ?",
                Boolean.class, task.get("rule_id")
            );
            if (deleteSource) {
                volumeManager.getProvider(sourceVolumeId).delete(storagePath);
            }

            // 5. Move series metadata cache file nếu toàn bộ series đã migrate sang target volume
            // (Chỉ move khi tất cả instances của series đã ở targetVolume)
            migrateSeriesMetadataIfComplete(tenantCode, instanceId, sourceVolumeId, targetVolumeId);

            // 6. Mark COMPLETED
            jdbcTemplate.update(
                "UPDATE public.migration_task SET status = 'COMPLETED', completed_at = now() WHERE id = ?",
                taskId
            );

        } catch (Exception e) {
            log.error("Migration task {} failed: {}", taskId, e.getMessage());
            jdbcTemplate.update(
                "UPDATE public.migration_task SET status = 'FAILED', error_message = ? WHERE id = ?",
                e.getMessage(), taskId
            );
        } finally {
            jdbcTemplate.execute("SET search_path TO public");
        }
    }

    /**
     * Sau khi 1 instance migrate xong, kiểm tra nếu toàn bộ series đã ở targetVolume
     * thì move metadata cache file sang volume mới.
     * Rebuild thay vì copy (đơn giản hơn, metadata file nhỏ).
     */
    private void migrateSeriesMetadataIfComplete(String tenantCode, long instanceId,
                                                   int sourceVolumeId, int targetVolumeId) {
        try {
            // Lấy seriesUid của instance này
            String seriesUid = jdbcTemplate.queryForObject(
                "SELECT series_instance_uid FROM instance WHERE id = ?", String.class, instanceId
            );

            // Kiểm tra có còn instance nào của series ở source volume không
            int remainingOnSource = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM instance WHERE series_instance_uid = ? AND volume_id = ?",
                Integer.class, seriesUid, sourceVolumeId
            );

            if (remainingOnSource == 0) {
                // Toàn bộ series đã migrate → rebuild metadata cache trên target volume
                seriesMetadataBuilder.invalidate(seriesUid, tenantCode);
                seriesMetadataBuilder.buildAndSave(seriesUid, tenantCode);
                log.debug("Rebuilt series metadata on target volume for series {}", seriesUid);
            }
        } catch (Exception e) {
            log.warn("Failed to migrate series metadata cache: {}", e.getMessage());
            // Non-fatal: WADO-RS sẽ fallback rebuild khi cần
        }
    }

    @Autowired
    @Qualifier("seriesMetadataBuilder")
    private DicomMetadataBuilder seriesMetadataBuilder;

    private String buildConditionSql(LifecycleRule rule) {
        return switch (rule.getConditionType()) {
            case "STUDY_AGE_DAYS" -> " AND s.created_at < now() - INTERVAL '" + rule.getConditionValue() + " days'";
            case "LAST_ACCESS_DAYS" -> " AND (s.last_accessed_at IS NULL OR s.last_accessed_at < now() - INTERVAL '" + rule.getConditionValue() + " days')";
            default -> throw new IllegalArgumentException("Unknown condition type: " + rule.getConditionType());
        };
    }

    private Object[] buildConditionArgs(LifecycleRule rule) {
        return new Object[] { rule.getSourceTier() };
    }

    private List<String> getApplicableTenants(LifecycleRule rule) {
        if (rule.getTenantCode() != null) {
            return List.of(rule.getTenantCode());
        }
        return jdbcTemplate.queryForList(
            "SELECT code FROM public.tenant WHERE active = true", String.class
        );
    }

    private List<LifecycleRule> loadEnabledMigrateRules() {
        return jdbcTemplate.query("""
            SELECT * FROM public.lifecycle_rule
            WHERE enabled = true AND action_type = 'MIGRATE'
            ORDER BY id
            """,
            new BeanPropertyRowMapper<>(LifecycleRule.class)
        );
    }
}
```

### 3. `src/main/java/com/spax/lifecycle/MigrationJob.java`

Scheduled job:

```java
@Component
public class MigrationJob {

    @Autowired private LifecycleService lifecycleService;

    /**
     * Evaluate rules nightly at 2:00 AM to create migration tasks.
     */
    @Scheduled(cron = "0 0 2 * * *")  // 2:00 AM daily
    public void evaluateRules() {
        log.info("Starting lifecycle rule evaluation");
        lifecycleService.evaluateMigrateRules();
        log.info("Lifecycle rule evaluation complete");
    }

    /**
     * Process migration tasks every 10 minutes.
     */
    @Scheduled(fixedDelay = 600_000)
    public void processTasks() {
        lifecycleService.processMigrationTasks();
    }
}
```

### 4. `src/main/java/com/spax/lifecycle/LifecycleController.java`

```java
@RestController
@RequestMapping("/api/v1/admin/lifecycle")
public class LifecycleController {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private LifecycleService lifecycleService;

    @GetMapping("/rules")
    public ResponseEntity<List<Map<String, Object>>> listRules() {
        return ResponseEntity.ok(jdbcTemplate.queryForList(
            "SELECT * FROM public.lifecycle_rule ORDER BY id"
        ));
    }

    @PostMapping("/rules")
    public ResponseEntity<Map<String, Object>> createRule(@RequestBody Map<String, Object> request) {
        // Validate action_type
        String actionType = (String) request.getOrDefault("actionType", "MIGRATE");
        if (!Set.of("MIGRATE", "COMPRESS").contains(actionType)) {
            return ResponseEntity.badRequest().body(Map.of("error", "Invalid actionType"));
        }

        // For MIGRATE: require target_tier
        if ("MIGRATE".equals(actionType) && request.get("targetTier") == null) {
            return ResponseEntity.badRequest().body(Map.of("error", "targetTier required for MIGRATE rules"));
        }

        // For COMPRESS: require compression_type
        if ("COMPRESS".equals(actionType) && request.get("compressionType") == null) {
            return ResponseEntity.badRequest().body(Map.of("error", "compressionType required for COMPRESS rules"));
        }

        jdbcTemplate.update("""
            INSERT INTO public.lifecycle_rule
              (name, action_type, source_tier, target_tier, condition_type, condition_value,
               delete_source, compression_type, tenant_code)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """,
            request.get("name"), actionType, request.get("sourceTier"),
            request.get("targetTier"), request.get("conditionType"), request.get("conditionValue"),
            request.getOrDefault("deleteSource", true), request.get("compressionType"),
            request.get("tenantCode")
        );

        return ResponseEntity.status(201).body(Map.of("created", true));
    }

    @PutMapping("/rules/{id}")
    public ResponseEntity<Void> updateRule(@PathVariable int id, @RequestBody Map<String, Object> request) {
        jdbcTemplate.update("""
            UPDATE public.lifecycle_rule SET
                enabled = COALESCE(?::boolean, enabled),
                condition_value = COALESCE(?::int, condition_value)
            WHERE id = ?
            """,
            request.get("enabled"), request.get("conditionValue"), id
        );
        return ResponseEntity.ok().build();
    }

    /**
     * POST /api/v1/admin/lifecycle/run
     * Manually trigger rule evaluation (don't wait for midnight schedule).
     */
    @PostMapping("/run")
    public ResponseEntity<Map<String, Object>> triggerMigration() {
        // Run async to avoid timeout
        Thread.startVirtualThread(() -> lifecycleService.evaluateMigrateRules());

        return ResponseEntity.accepted().body(Map.of("message", "Migration evaluation started"));
    }

    /**
     * GET /api/v1/admin/lifecycle/status
     * View pending/in-progress migration tasks.
     */
    @GetMapping("/status")
    public ResponseEntity<Map<String, Object>> getMigrationStatus() {
        Map<String, Object> status = new LinkedHashMap<>();
        status.put("pending", jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM public.migration_task WHERE status = 'PENDING'", Long.class
        ));
        status.put("inProgress", jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM public.migration_task WHERE status = 'IN_PROGRESS'", Long.class
        ));
        status.put("failed", jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM public.migration_task WHERE status = 'FAILED'", Long.class
        ));
        status.put("completedToday", jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM public.migration_task WHERE status = 'COMPLETED' AND completed_at >= CURRENT_DATE",
            Long.class
        ));
        return ResponseEntity.ok(status);
    }
}
```

## Lưu ý quan trọng
- Lifecycle evaluation tạo `migration_task` records (không migrate ngay) → tách eval vs execution
- Idempotent: kiểm tra task đã tồn tại trước khi INSERT
- Migration flow: copy → verify → update DB → delete source (không reverse được, log cẩn thận)
- `SET search_path` trực tiếp trên JdbcTemplate — không dùng TenantContext (lifecycle chạy background, không có HTTP request)
- COMPRESS action_type sẽ được xử lý ở SPEC-20

## Kiểm tra thành công
- Tạo lifecycle rule HOT→WARM after 180 days
- Có study cũ > 180 days trên HOT tier
- `POST /admin/lifecycle/run` → migration tasks được tạo
- `GET /admin/lifecycle/status` → pending count > 0
- Sau processTasks() → files xuất hiện trên WARM volume, instance.volume_id đã đổi
