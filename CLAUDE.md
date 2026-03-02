# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
SPAX (Simple PACS Archive System) — Long-term DICOM storage for 1000+ on-premise deployments.
Core functions: receive DICOM files → index metadata → serve DICOMWeb queries (QIDO-RS, WADO-RS, STOW-RS) for OHIF Viewer.

**Full architecture details**: See `PLAN.md` at project root.

## Build & Run

### Prerequisites
- Java 21 JDK
- Maven 3.9.x
- PostgreSQL 16+
- Redis 7.x (if using default queue type)

### Commands
```bash
# Build
mvn clean package

# Run (dev)
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"

# Run all tests
mvn test

# Run single test class
mvn test -Dtest=IngestServiceTest

# Run single test method
mvn test -Dtest=IngestServiceTest#shouldParseAndIndexDicom

# Skip tests during build
mvn clean package -DskipTests
```

## Architecture

### Multi-Tenancy (Schema per Tenant)
Tenant isolation uses PostgreSQL schema-per-tenant. `TenantInterceptor` extracts the tenant code from the HTTP request (`X-Tenant-ID` header or path parameter), stores it in `TenantContext` (ThreadLocal), and `TenantSchemaResolver` executes `SET search_path TO tenant_{code}, public` on every connection. All JPA/JDBC queries then operate on the correct schema without any `tenant_id` filter.

### Ingest Pipeline (3 paths → 1 queue → IndexingConsumer)
Three ingest paths converge into `IngestQueue`, then a single consumer handles indexing:
1. **Orthanc integration**: `TransferCommitListener` fires when Transfer Server commits files
2. **REST Batch API**: `POST /api/v1/{tenant}/ingest` (multipart)
3. **STOW-RS**: `POST /dicomweb/{tenant}/studies`

`IndexingConsumer` batches up to 200 messages, parses DICOM metadata (skipping pixel data), moves files to permanent storage, then does a single-transaction batch upsert: Patient → Study → Series → Instance.

### Storage (Multi-Volume SPI)
`StorageProvider` is an SPI with two implementations: `LocalStorageProvider` (Java NIO) and `JCloudStorageProvider` (Apache jclouds — supports S3/GCS/Azure/MinIO).

`VolumeManager` selects which volume to write to based on tier priority (HOT → WARM → COLD) and disk availability. Volume registry is in the `public.storage_volume` table (shared across tenants).

File path resolved at ingest time by `StoragePathResolver`, which wraps dcm4che's `org.dcm4che3.util.AttributesFormat` (from `dcm4che-core`). `AttributesFormat` instances are **cached** (immutable, thread-safe) — compiled template reused across all ingest calls. Template uses DICOM tag numbers with type modifiers: `{tag,hash}` = Java hashCode (8 hex), `{tag,md5}` = MD5 base32 (26 chars), `{now,date,pattern}` = current date, `{tag}` = raw value, `{tag,slice,start,end}` = substring, `{rnd}` = random. Full reference in SPEC-05. SPAX prepends `tenantCode/` to the result. Default template: `{now,date,yyyy/MM/dd}/{0020000D,hash}/{0020000E,hash}/{00080018,hash}`. Config per volume via `storage_volume.path_template`; null falls back to `spax.storage.default-path-template`.

### Database Design
- **Patient, Study, Series**: No partitioning — B-tree indexes sufficient
- **Instance**: `PARTITION BY RANGE (created_date)` — monthly, system-generated. `created_date = series.created_at::date` (NOT `CURRENT_DATE`) → all instances of the same series land in the same partition → enables partition pruning via 2-step query: series table → `created_date` → instance table with `WHERE series_fk = ? AND created_date = ?`. DICOMWeb standard has no `created_date` concept, so it must be fetched from series table first
- **No `dicom_attrs` on any table** — DICOM files are the source of truth. QIDO-RS responses are built from indexed columns. WADO-RS metadata reads the DICOM file directly (skipping pixel data via `IncludeBulkData.NO`)
- **Unique keys**: DICOM UIDs (StudyUID, SeriesUID, SOPUID) are NOT globally unique in practice (buggy/cloned machines). Strategy: patient=`public_id SHA1(pid)`, study=`public_id SHA1(pid|studyUid)` — PID in formula handles UID collision between different patients. Series/Instance rely on "cascade correctness": series=`UNIQUE(study_fk, series_uid)`, instance=no DB-level unique (PostgreSQL partitioned tables require partition key in unique index) → dedup enforced at application level in `BulkInsertRepository`: filter out existing `(series_fk, sop_uid)` before bulk insert. Correction (change PID): only patient (1 row) + async study `public_id` update (N rows); series/instance never touched. `is_provisional=true` when PID is missing or suspected collision
- **Optimistic locking**: Patient/Study/Series have a `version` column

### Queue Abstraction
`IngestQueue` supports two implementations, swappable via config:
```yaml
spax:
  queue:
    type: redis   # or: wal
```
- `RedisStreamIngestQueue`: Default. Stream key `spax:ingest:{tenantCode}`, consumer group `indexer-group`
- `WalIngestQueue`: Future. Append-only file, no Redis dependency

### Caching Layer (Spring Cache — EhCache or Redis)
Configurable cache backend via `spax.cache.type`: `ehcache` (default, local in-memory) or `redis` (distributed, shared across instances). Uses Spring Cache abstraction (`@Cacheable`, `@CacheEvict`).

**Cache names:**
- `instance-locations`: batch-loaded per series (key: `{tenant}:{seriesUid}` → Map<sopUid, (volumeId, storagePath)>). TTL 30 min. Critical for WADO-RS frame retrieval — 1000 frames = 1 DB query instead of 1000.
- `series-metadata-lookup`: seriesUid → (metadataPath, metadataVolumeId). TTL 1 hour. For WADO-RS metadata endpoint.
- `active-tenants`: singleton list of active tenant codes. TTL 60s. Used by IndexingConsumer loop.
- `series-by-study`: studyUid → List<SeriesSummary>. TTL 1 hour.
- `lifecycle-rules`: actionType → List<LifecycleRule>. TTL 6 hours.

**Multi-tenant cache keys**: all tenant-specific caches use `{tenantCode}:{uid}` composite key.

**Invalidation**: `WadoRsCacheService` provides evict methods. Called by IndexingConsumer (after ingest), LifecycleService (after migration), FileCorrectionJob (after correction), StudyAdminService (after deletion).

**`instance-locations` batch load**: 2-step via series_fk (not direct series_instance_uid query — no index on that column in partitioned instance table). Step 1: `series.id` via `idx_series_uid`. Step 2: instances via `idx_instance_dedup(series_fk, sop_instance_uid)`.

### Self-Management Services
- `AutoPartitionService`: Runs daily, ensures Instance partitions exist 12 months ahead
- `DiskSpaceMonitor`: Runs every 5 min. Rejects ingest (HTTP 507) when disk < 10% free
- `AutoRecoveryService`: Restarts `IndexingConsumer` on failure, skips bad DICOM files

## Key Conventions
- `public` schema = shared config (tenant registry, storage volumes, lifecycle rules)
- `tenant_{code}` schema = per-tenant DICOM data (Patient/Study/Series/Instance)
- Bad/unreadable DICOM files → moved to `{volume}/error/` directory, never block the pipeline
- Admin module is API-only (no frontend in this repo)
- DICOM correction updates DB immediately, then queues a background `FileCorrectionJob` for file header patching
- Compression replaces files in-place (same storage path/volume). Updates: `instance.transfer_syntax_uid`, `instance.file_size`, `series.compress_tsuid`, `series.compress_time`. After task completes, `study_size`/`series_size` recalculated via `SUM(instance.file_size)`
- Lifecycle rules have `action_type`: `MIGRATE` (move between tiers) or `COMPRESS` (transcode in place). `compression_type` is set only for `COMPRESS` rules
- **WADO-RS series metadata cache**: `series.metadata_path` + `series.metadata_volume_id` trỏ đến pre-built DICOM JSON array file (tất cả instances của series, không có pixel data). Build bởi `SeriesMetadataBuilder` (implements `DicomMetadataBuilder`) ngay sau ingest batch. Fallback khi null: LOCAL → đọc DICOM files trực tiếp + async rebuild; CLOUD → buộc sync build trước khi trả về (tránh N API calls tốn tiền). Invalidate + rebuild sau `FileCorrectionJob`. Di chuyển cùng DICOM files khi lifecycle migration. Path convention: `{tenantCode}/series-meta/{uid[0:2]}/{uid[2:4]}/{seriesUid}.json` trên cùng volume
- `DicomMetadataBuilder` là interface chung (implement cho Series, Study nếu cần). Khác với `DicomJsonBuilder` (chỉ convert DB records → DICOM JSON cho QIDO, không đọc file)
