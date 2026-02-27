# SPEC-20: Lifecycle Extend — COMPRESS Action Type

## Mục tiêu
Extend LifecycleService để xử lý rules với `action_type='COMPRESS'`. Auto-compression tích hợp vào lifecycle rule engine hiện có (SPEC-16).

## Dependencies
- SPEC-16 (LifecycleService — đã có MIGRATE flow)
- SPEC-19 (CompressionService — đã có createTask())

## Files cần sửa đổi

### 1. `src/main/java/com/spax/lifecycle/LifecycleService.java` — thêm COMPRESS support

Thêm method `evaluateCompressRules()` và gọi nó từ `MigrationJob`:

```java
// Thêm vào LifecycleService:

@Autowired private CompressionService compressionService;

/**
 * Evaluate all enabled COMPRESS rules.
 * Creates compression tasks for qualifying studies.
 */
public void evaluateCompressRules() {
    List<LifecycleRule> rules = loadEnabledCompressRules();

    for (LifecycleRule rule : rules) {
        try {
            evaluateCompressRule(rule);
        } catch (Exception e) {
            log.error("Failed to evaluate compress rule {}: {}", rule.getId(), e.getMessage());
        }
    }
}

private void evaluateCompressRule(LifecycleRule rule) {
    CompressionType compressionType = CompressionType.valueOf(rule.getCompressionType());
    String targetTsUid = compressionType.getTransferSyntaxUid();

    List<String> tenants = getApplicableTenants(rule);

    for (String tenantCode : tenants) {
        jdbcTemplate.execute("SET search_path TO tenant_" + sanitize(tenantCode) + ", public");
        try {
            evaluateCompressRuleForTenant(rule, tenantCode, compressionType, targetTsUid);
        } finally {
            jdbcTemplate.execute("SET search_path TO public");
        }
    }
}

private void evaluateCompressRuleForTenant(LifecycleRule rule, String tenantCode,
                                             CompressionType compressionType, String targetTsUid) {
    // Build condition SQL (same logic as MIGRATE — age based on conditionType)
    String conditionSql = buildConditionSql(rule);

    // Find studies with instances on source tier that are NOT yet compressed with target syntax
    // Group by study to create 1 task per study
    String sql = """
        SELECT DISTINCT i.study_instance_uid
        FROM instance i
        JOIN study s ON i.study_instance_uid = s.study_instance_uid
        JOIN public.storage_volume sv ON i.volume_id = sv.id
        WHERE sv.tier = ?
        """ + conditionSql + """
        AND EXISTS (
            -- Study has at least 1 instance NOT yet in target transfer syntax
            SELECT 1 FROM instance i2
            WHERE i2.study_instance_uid = i.study_instance_uid
            AND (i2.transfer_syntax_uid IS NULL OR i2.transfer_syntax_uid != ?)
        )
        AND NOT EXISTS (
            -- No existing PENDING/IN_PROGRESS compression task for this study+type
            SELECT 1 FROM compression_task ct
            WHERE ct.study_instance_uid = i.study_instance_uid
            AND ct.compression_type = ?
            AND ct.status IN ('PENDING', 'IN_PROGRESS')
        )
        ORDER BY i.study_instance_uid
        LIMIT 100
        """;

    List<String> studyUids = jdbcTemplate.queryForList(
        sql, String.class,
        rule.getSourceTier(), targetTsUid, compressionType.name()
    );

    for (String studyUid : studyUids) {
        try {
            long taskId = compressionService.createTask(
                studyUid,
                compressionType,
                rule.getId(),   // ruleId
                null            // triggeredBy = null (auto)
            );
            log.info("Created auto-compression task {} for study {} (rule {})",
                taskId, studyUid, rule.getId());
        } catch (Exception e) {
            log.warn("Failed to create compression task for study {}: {}", studyUid, e.getMessage());
        }
    }
}

private List<LifecycleRule> loadEnabledCompressRules() {
    return jdbcTemplate.query("""
        SELECT * FROM public.lifecycle_rule
        WHERE enabled = true AND action_type = 'COMPRESS'
        ORDER BY id
        """,
        new BeanPropertyRowMapper<>(LifecycleRule.class)
    );
}

private String sanitize(String code) {
    if (!code.matches("[a-zA-Z0-9_]+")) throw new IllegalArgumentException("Invalid code: " + code);
    return code;
}
```

### 2. `src/main/java/com/spax/lifecycle/MigrationJob.java` — thêm COMPRESS trigger

```java
// Thêm vào MigrationJob.java:

/**
 * Evaluate COMPRESS rules nightly (same time as MIGRATE evaluation).
 */
@Scheduled(cron = "0 0 2 * * *")  // 2:00 AM daily (cùng với evaluateMigrateRules)
public void evaluateAllRules() {
    log.info("Starting lifecycle rule evaluation (MIGRATE + COMPRESS)");
    lifecycleService.evaluateMigrateRules();
    lifecycleService.evaluateCompressRules();
    log.info("Lifecycle rule evaluation complete");
}
```

**Note**: Nếu `evaluateRules()` đã tồn tại từ SPEC-16, thêm `evaluateCompressRules()` vào đó thay vì tạo method mới.

### 3. `src/main/java/com/spax/lifecycle/LifecycleController.java` — `POST /run` đã cover

`POST /api/v1/admin/lifecycle/run` từ SPEC-16 cần gọi thêm `evaluateCompressRules()`:

```java
@PostMapping("/run")
public ResponseEntity<Map<String, Object>> triggerMigration() {
    Thread.startVirtualThread(() -> {
        lifecycleService.evaluateMigrateRules();   // existing
        lifecycleService.evaluateCompressRules();  // new
    });
    return ResponseEntity.accepted().body(Map.of("message", "Lifecycle evaluation started (MIGRATE + COMPRESS)"));
}
```

## Idempotency Strategy

Compression idempotency có 2 lớp:
1. **Rule evaluation**: Không tạo task mới nếu đã có task PENDING/IN_PROGRESS cho study đó với cùng compression_type
2. **Task worker**: Trong `CompressionService.processInstance()`, check `transfer_syntax_uid` trước khi compress — skip nếu đã đúng

Kết quả: Chạy `POST /admin/lifecycle/run` nhiều lần an toàn — không duplicate, không re-nén unnecessarily.

## Ví dụ workflow

```bash
# 1. Tạo lifecycle rule auto-compress sau 90 ngày
POST /api/v1/admin/lifecycle/rules
{
  "name": "Compress HOT after 3 months",
  "actionType": "COMPRESS",
  "sourceTier": "HOT",
  "conditionType": "STUDY_AGE_DAYS",
  "conditionValue": 90,
  "compressionType": "JPEG_LS_LOSSLESS"
}

# 2. Trigger manually để test (không đợi midnight)
POST /api/v1/admin/lifecycle/run

# 3. Check compression tasks được tạo
GET /api/v1/{tenant}/admin/compressions?status=PENDING

# 4. Monitor progress
GET /api/v1/{tenant}/admin/compressions/{taskId}
# → { "status": "IN_PROGRESS", "processedFiles": 45, "totalFiles": 120 }

# 5. After completion
GET /api/v1/{tenant}/admin/compressions/{taskId}
# → { "status": "COMPLETED", "processedFiles": 120, "totalFiles": 120 }
```

## Lưu ý quan trọng
- Rule `action_type='COMPRESS'` không cần `target_tier` (file ở lại cùng volume/tier)
- Rule `action_type='COMPRESS'` không dùng `delete_source` (không có source/target)
- Chạy MIGRATE rule + COMPRESS rule cùng 1 study: OK — MIGRATE chạy trước di chuyển file, COMPRESS chạy sau nén file tại đích
- `LIMIT 100` per evaluation pass — tránh create quá nhiều tasks cùng lúc (CompressionService có virtual thread pool riêng)

## Kiểm tra thành công
- Tạo COMPRESS lifecycle rule với conditionValue=0 (ngay lập tức)
- `POST /admin/lifecycle/run` → compression tasks được tạo cho tất cả studies đủ điều kiện
- Mỗi study chỉ tạo 1 task (không duplicate)
- Re-run `POST /admin/lifecycle/run` → không tạo thêm task (vì PENDING/IN_PROGRESS đã tồn tại)
- Sau COMPLETED → instances có `transfer_syntax_uid` mới
- `GET /dicomweb/{tenant}/studies/{uid}/series/{uid}/instances/{sopUid}` → WADO-RS trả file đã nén
