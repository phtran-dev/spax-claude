# SPAX Implementation Specs

Các spec này dùng để giao cho agent/developer implement từng module. Mỗi spec độc lập với dependency rõ ràng.

## Thứ tự thực hiện (theo dependency)

| Spec | Tên | Files chính | Phụ thuộc |
|------|-----|-------------|-----------|
| [SPEC-01](SPEC-01-project-setup.md) | Project Setup | `pom.xml`, `application.yml`, `SpaxApplication.java` | None |
| [SPEC-02](SPEC-02-database-schema.md) | Database Schema | `V1__shared_schema.sql`, `V1__init.sql` (tenant template) | #01 |
| [SPEC-03](SPEC-03-multitenant.md) | Multi-Tenant Infrastructure | `TenantContext`, `TenantInterceptor`, `TenantSchemaResolver`, `TenantManagementService`, `WebConfig` | #01, #02 |
| [SPEC-04](SPEC-04-storage-provider.md) | Storage Provider SPI | `StorageProvider` (interface), `LocalStorageProvider`, `JCloudStorageProvider`, `StoreResult`, `StorageService` (interface) | #01 |
| [SPEC-05](SPEC-05-volume-manager.md) | Volume Manager | `StorageVolume` (entity), `VolumeManager`, `StoragePathResolver`, `StorageServiceImpl`, `StorageVolumeRepository` | #02, #04 |
| [SPEC-06](SPEC-06-queue.md) | Queue Abstraction + Redis | `IngestMessage`, `IngestQueue` (interface), `RedisStreamIngestQueue`, `WalIngestQueue` (stub), `RedisConfig` | #01 |
| [SPEC-07](SPEC-07-dicom-parser.md) | DICOM Parser | `DicomParser`, `DicomMetadata` | #01 |
| [SPEC-08](SPEC-08-ingest-pipeline.md) | Ingest Pipeline | `IngestService`, `IngestController`, `TransferCommitListener`, `IndexingConsumer` | #02,#03,#05,#06,#07,#09 |
| [SPEC-09](SPEC-09-bulk-insert.md) | Bulk Insert Repository | `BulkInsertRepository`, `PatientRepository`, `StudyRepository`, `SeriesRepository`, `InstanceRepository` | #02, #03 |
| [SPEC-10](SPEC-10-qido-rs.md) | QIDO-RS | `QueryBuilder`, `DicomJsonBuilder`, `QidoController` | #02, #03, #09 |
| [SPEC-11](SPEC-11-wado-rs.md) | WADO-RS | `WadoController` | #02, #03, #05 |
| [SPEC-12](SPEC-12-stow-rs.md) | STOW-RS | `StowController` | #08 |
| [SPEC-13](SPEC-13-admin-correction.md) | Admin: Correction + Audit | `AuditService`, `CorrectionService`, `FileCorrectionJob`, `CorrectionController` | #02, #03, #05, #07 |
| [SPEC-14](SPEC-14-admin-study-system.md) | Admin: Study Mgmt + Stats | `StudyAdminController`, `SystemInfoController` | #02, #03, #05 |
| [SPEC-15](SPEC-15-admin-tenant-volume.md) | Admin: Tenant + Volume | `TenantController`, `VolumeController` | #02, #03, #05 |
| [SPEC-16](SPEC-16-lifecycle.md) | Data Lifecycle | `LifecycleRule`, `LifecycleService`, `MigrationJob`, `LifecycleController` | #02, #05 |
| [SPEC-17](SPEC-17-selfmanage.md) | Self-Management | `AutoPartitionService`, `DiskSpaceMonitor`, `AutoRecoveryService` | #02, #03, #05 |
| [SPEC-18](SPEC-18-health.md) | Health Endpoint | `HealthController` | All |
| [SPEC-19](SPEC-19-compression.md) | Compression Module | `CompressionType`, `DicomCompressor`, `CompressionTask`, `CompressionService`, `CompressionController` | #02, #05, #13 |
| [SPEC-20](SPEC-20-lifecycle-compress.md) | Lifecycle COMPRESS action | Extended `LifecycleService` + `MigrationJob` | #16, #19 |

## Parallel Execution Groups

Tasks trong cùng nhóm có thể làm song song:

**Group A** (no deps):
- SPEC-01 (do first — all depend on it)

**Group B** (depends only on #01):
- SPEC-02, SPEC-04, SPEC-06, SPEC-07

**Group C** (depends on #01, #02):
- SPEC-03, SPEC-09

**Group D** (depends on #02, #04):
- SPEC-05

**Group E** (depends on #02, #03, #05, #06, #07, #09):
- SPEC-08 (ingest — needs all of the above)

**Group F** (depends on #02, #03, #09):
- SPEC-10 (QIDO-RS)

**Group G** (depends on #02, #03, #05):
- SPEC-11 (WADO-RS), SPEC-14, SPEC-15, SPEC-16

**Group H** (depends on #08):
- SPEC-12 (STOW-RS)

**Group I** (depends on #02, #03, #05, #07):
- SPEC-13 (Correction + Audit)

**Group J** (depends on #02, #03):
- SPEC-17 (Self-management)

**Group K** (needs everything):
- SPEC-18 (Health)
- SPEC-19 (Compression — depends on #05, #13)
- SPEC-20 (Lifecycle COMPRESS — depends on #16, #19)

## Key Architecture Notes

1. **Schema convention**: `public` = shared (tenant registry, volumes, lifecycle rules). `tenant_{code}` = per-tenant DICOM data.

2. **Unique key strategy**:
   - `patient.public_id = SHA1(patient_id)` — Orthanc-style
   - `study.public_id = SHA1(patient_id + "|" + study_instance_uid)` — includes PID to handle UID collisions
   - `series`: `UNIQUE(study_fk, series_instance_uid)` — CASCADE via BIGINT FK
   - `instance`: no DB-level UNIQUE (PostgreSQL partitioned table limitation) — application-level dedup in BulkInsertRepository

3. **Storage path**: resolved by `StoragePathResolver` using dcm4che `AttributesFormat` templates. Format: `{tenantCode}/{yyyy/MM/dd}/{studyUid,hash}/{seriesUid,hash}/{sopUid,hash}`

4. **No `dicom_attrs` on any table** — DICOM files are source of truth. QIDO uses indexed columns. WADO-RS reads file directly.

5. **Virtual threads everywhere** — Spring Boot 3.4.x + Java 21. Just set `spring.threads.virtual.enabled=true`.

## Conventions

- All admin endpoints: `/api/v1/{tenant}/admin/...` (tenant-scoped) or `/api/v1/admin/...` (global)
- All DICOMWeb endpoints: `/dicomweb/{tenant}/...`
- Transfer Server integration: `POST /api/v1/transfer/commit`
- Health check: `GET /api/v1/health`
