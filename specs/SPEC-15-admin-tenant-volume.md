# SPEC-15: Admin — Tenant Management + Volume Management

## Mục tiêu
REST API để quản lý tenants và storage volumes. Thao tác global (không phụ thuộc tenant context).

## Dependencies
- SPEC-02 (schema — public.tenant, public.storage_volume)
- SPEC-03 (TenantManagementService)
- SPEC-05 (VolumeManager, StoragePathResolver)

## Files cần tạo

### 1. `src/main/java/com/spax/admin/TenantController.java`

```java
@RestController
@RequestMapping("/api/v1/admin/tenants")
public class TenantController {

    @Autowired private TenantManagementService tenantManagementService;
    @Autowired private JdbcTemplate jdbcTemplate;  // uses public schema (no tenant context)

    /**
     * GET /api/v1/admin/tenants
     * List all tenants.
     */
    @GetMapping
    public ResponseEntity<List<Map<String, Object>>> listTenants() {
        return ResponseEntity.ok(jdbcTemplate.queryForList(
            "SELECT id, code, name, storage_type, active, created_at FROM public.tenant ORDER BY code"
        ));
    }

    /**
     * POST /api/v1/admin/tenants
     * Create a new tenant: creates schema, runs V1__init.sql, creates initial partitions.
     *
     * Body: { "code": "hospital_a", "name": "Hospital A" }
     *
     * Validation:
     * - code must match [a-z0-9_]+ (lowercase alphanumeric + underscore)
     * - code must be unique
     * - name is required
     */
    @PostMapping
    public ResponseEntity<Map<String, Object>> createTenant(@RequestBody CreateTenantRequest request) {
        // Validate code format
        if (!request.code().matches("[a-z0-9_]+")) {
            return ResponseEntity.badRequest().body(Map.of(
                "error", "Tenant code must be lowercase alphanumeric and underscore only"
            ));
        }

        // Check uniqueness
        int existing = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM public.tenant WHERE code = ?", Integer.class, request.code()
        );
        if (existing > 0) {
            return ResponseEntity.status(409).body(Map.of("error", "Tenant code already exists"));
        }

        tenantManagementService.createTenant(request.code(), request.name());

        return ResponseEntity.status(201).body(Map.of(
            "code", request.code(),
            "name", request.name(),
            "schema", "tenant_" + request.code()
        ));
    }

    /**
     * PUT /api/v1/admin/tenants/{code}
     * Update tenant name or active status.
     */
    @PutMapping("/{code}")
    public ResponseEntity<Void> updateTenant(
            @PathVariable String code,
            @RequestBody UpdateTenantRequest request) {

        int updated = jdbcTemplate.update(
            "UPDATE public.tenant SET name = COALESCE(?, name), active = COALESCE(?, active) WHERE code = ?",
            request.name(), request.active(), code
        );

        if (updated == 0) return ResponseEntity.notFound().build();
        return ResponseEntity.ok().build();
    }

    public record CreateTenantRequest(String code, String name) {}
    public record UpdateTenantRequest(String name, Boolean active) {}
}
```

### 2. `src/main/java/com/spax/admin/VolumeController.java`

```java
@RestController
@RequestMapping("/api/v1/admin/volumes")
public class VolumeController {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private VolumeManager volumeManager;
    @Autowired private StoragePathResolver pathResolver;

    /**
     * GET /api/v1/admin/volumes
     * List all storage volumes with disk usage.
     */
    @GetMapping
    public ResponseEntity<List<Map<String, Object>>> listVolumes() {
        List<Map<String, Object>> volumes = jdbcTemplate.queryForList(
            "SELECT * FROM public.storage_volume ORDER BY tier, priority DESC"
        );

        // Enrich local volumes with real-time disk info
        for (Map<String, Object> vol : volumes) {
            if ("LOCAL".equals(vol.get("provider_type"))) {
                try {
                    int volId = (int) vol.get("id");
                    if (volumeManager.getProvider(volId) instanceof LocalStorageProvider local) {
                        vol.put("availableBytes", local.getAvailableBytes());
                        vol.put("totalDiskBytes", local.getTotalBytes());
                        vol.put("usedPercent",
                            100 - (local.getAvailableBytes() * 100.0 / local.getTotalBytes()));
                    }
                } catch (Exception e) {
                    vol.put("status", "ERROR");
                }
            }
            // Hide credentials in response
            vol.remove("cloud_credential");
        }

        return ResponseEntity.ok(volumes);
    }

    /**
     * POST /api/v1/admin/volumes
     * Add a new storage volume.
     *
     * Validation:
     * - code must be unique
     * - provider_type must be valid: LOCAL, aws-s3, google-cloud-storage, azureblob, s3
     * - For LOCAL: base_path must be accessible
     * - path_template (if provided) must contain {00080018} (SOP UID)
     */
    @PostMapping
    public ResponseEntity<Map<String, Object>> addVolume(@RequestBody AddVolumeRequest request) {
        // Validate path template
        if (request.pathTemplate() != null) {
            try {
                pathResolver.validateTemplate(request.pathTemplate());
            } catch (IllegalArgumentException e) {
                return ResponseEntity.badRequest().body(Map.of("error", e.getMessage()));
            }
        }

        // Validate LOCAL path accessibility
        if ("LOCAL".equals(request.providerType())) {
            Path path = Path.of(request.basePath());
            if (!Files.exists(path)) {
                try { Files.createDirectories(path); }
                catch (IOException e) {
                    return ResponseEntity.badRequest().body(Map.of(
                        "error", "Cannot access or create base path: " + request.basePath()
                    ));
                }
            }
        }

        jdbcTemplate.update("""
            INSERT INTO public.storage_volume
              (code, provider_type, base_path, tier, status, priority, path_template,
               cloud_endpoint, cloud_region, cloud_bucket, cloud_identity, cloud_credential)
            VALUES (?, ?, ?, ?, 'ACTIVE', ?, ?, ?, ?, ?, ?, ?)
            """,
            request.code(), request.providerType(), request.basePath(),
            request.tier(), request.priority() != null ? request.priority() : 0,
            request.pathTemplate(), request.cloudEndpoint(), request.cloudRegion(),
            request.cloudBucket(), request.cloudIdentity(), request.cloudCredential()
        );

        // Reload VolumeManager to recognize new volume
        volumeManager.reload();

        return ResponseEntity.status(201).body(Map.of(
            "code", request.code(),
            "tier", request.tier(),
            "status", "ACTIVE"
        ));
    }

    /**
     * PUT /api/v1/admin/volumes/{id}
     * Update volume: status (ACTIVE/READ_ONLY/OFFLINE), priority, path_template.
     */
    @PutMapping("/{id}")
    public ResponseEntity<Void> updateVolume(
            @PathVariable int id,
            @RequestBody UpdateVolumeRequest request) {

        if (request.pathTemplate() != null) {
            try {
                pathResolver.validateTemplate(request.pathTemplate());
            } catch (IllegalArgumentException e) {
                return ResponseEntity.badRequest().build();
            }
        }

        int updated = jdbcTemplate.update("""
            UPDATE public.storage_volume SET
                status       = COALESCE(?, status),
                priority     = COALESCE(?, priority),
                path_template = COALESCE(?, path_template),
                updated_at   = now()
            WHERE id = ?
            """,
            request.status(), request.priority(), request.pathTemplate(), id
        );

        if (updated == 0) return ResponseEntity.notFound().build();

        volumeManager.reload();
        return ResponseEntity.ok().build();
    }

    /**
     * POST /api/v1/admin/volumes/reload
     * Force reload volume config from DB (call after manual DB changes).
     */
    @PostMapping("/reload")
    public ResponseEntity<Void> reloadVolumes() {
        volumeManager.reload();
        return ResponseEntity.ok().build();
    }

    public record AddVolumeRequest(
        String code, String providerType, String basePath, String tier, Integer priority,
        String pathTemplate, String cloudEndpoint, String cloudRegion,
        String cloudBucket, String cloudIdentity, String cloudCredential
    ) {}

    public record UpdateVolumeRequest(String status, Integer priority, String pathTemplate) {}
}
```

## Endpoints Summary

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/admin/tenants` | List all tenants |
| POST | `/api/v1/admin/tenants` | Create tenant + schema |
| PUT | `/api/v1/admin/tenants/{code}` | Update tenant |
| GET | `/api/v1/admin/volumes` | List volumes + disk usage |
| POST | `/api/v1/admin/volumes` | Add volume |
| PUT | `/api/v1/admin/volumes/{id}` | Update volume status/priority |
| POST | `/api/v1/admin/volumes/reload` | Reload VolumeManager cache |

## Lưu ý quan trọng
- Không expose `cloud_credential` trong list/get response (sensitive data)
- `volumeManager.reload()` phải gọi sau mỗi add/update volume để in-memory cache được sync
- Validate path template khi add/update volume (không validate khi app boot)
- Tenant creation là destructive — không có undo. Cần validation tốt.

## Kiểm tra thành công
- `POST /admin/tenants {"code": "test", "name": "Test"}` → schema `tenant_test` tạo thành công
- `POST /admin/volumes` với LOCAL path → volume ACTIVE, VolumeManager recognize
- `PUT /admin/volumes/1 {"status": "READ_ONLY"}` → ingest không còn write vào volume này
- `POST /admin/volumes/reload` → không error
