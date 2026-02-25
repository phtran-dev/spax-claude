# SPAX - Simple PACS Archive System

## Context

Cần xây dựng hệ thống PACS lưu trữ dài hạn, tập trung vào **lưu file, index metadata, và query nhanh** qua DICOMWeb cho OHIF viewer. Khác với Orthanc/dcm4chee (full-featured nhưng chậm ở quy mô lớn), SPAX chỉ làm 3 việc: **nhận file, lưu/index, phục vụ query** - nhưng làm cực nhanh ở quy mô 100M-1B bản ghi.

**Kiến trúc tổng quan:**
```
Orthanc/Gateway → [Batch REST API] → SPAX → [Redis Streams] → [Indexing Workers]
                                        ↓                            ↓
                                   File Storage              PostgreSQL
                                  (Local + S3)           (Schema per tenant)
                                        ↓
                              OHIF Viewer ← [DICOMWeb API (QIDO-RS, WADO-RS)]
```

**Quyết định kiến trúc chính:**
- **Multi-tenant**: Schema per tenant (mỗi tenant 1 schema riêng, isolation tốt, query không cần filter tenant_id)
- **Storage**: Local filesystem + S3/MinIO (abstraction layer)
- **Queue**: Redis Streams (bền hơn in-memory, hỗ trợ consumer groups)
- **Protocol**: REST API batch ingest + DICOMWeb (QIDO-RS, WADO-RS) cho OHIF
- **Không DIMSE**: Orthanc/gateway nhận ảnh từ máy chụp, đẩy batch lên SPAX

---

## Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Language | Java | 21 (virtual threads) |
| Framework | Spring Boot | 3.4.x |
| Database | PostgreSQL | 16+ |
| DICOM Library | dcm4che | 5.31.x (core, json modules) |
| Queue | Redis Streams | 7.x |
| Cloud Storage | Apache jclouds | 2.6.x (S3, GCS, Azure, etc.) |
| Local Storage | Java NIO | (JDK built-in) |
| Build | Maven | 3.9.x |
| Connection Pool | HikariCP | (Spring Boot default) |

---

## Project Structure

```
spax-claude/
├── pom.xml
├── src/main/java/com/spax/
│   ├── SpaxApplication.java
│   ├── config/
│   │   ├── DataSourceConfig.java          # Multi-tenant datasource routing
│   │   ├── RedisConfig.java               # Redis Streams config
│   │   ├── StorageConfig.java             # Storage abstraction config
│   │   └── WebConfig.java                 # CORS, multipart config
│   ├── tenant/
│   │   ├── TenantContext.java             # ThreadLocal tenant holder
│   │   ├── TenantInterceptor.java         # Extract tenant from request header/path
│   │   ├── TenantSchemaResolver.java      # Map tenant → schema
│   │   └── TenantManagementService.java   # Create/migrate tenant schemas
│   ├── storage/
│   │   ├── StorageService.java            # Interface chung cho read/write
│   │   ├── LocalStorageProvider.java      # Local filesystem impl (Java NIO)
│   │   ├── JCloudStorageProvider.java     # jclouds impl (S3, GCS, Azure, ...)
│   │   ├── VolumeManager.java             # Volume registry + active volume selection
│   │   ├── VolumeController.java          # Admin API quản lý volumes
│   │   ├── StoreResult.java              # Value object (volumeId + path)
│   │   └── StorageProvider.java           # SPI interface cho từng loại storage
│   ├── lifecycle/
│   │   ├── LifecycleRule.java             # Entity: rule chuyển file giữa volumes
│   │   ├── LifecycleService.java          # Evaluate rules + migrate files
│   │   ├── MigrationJob.java             # Scheduled job chạy migration
│   │   └── LifecycleController.java      # Admin API quản lý lifecycle rules
│   ├── queue/
│   │   ├── IngestQueue.java               # Interface: publish/consume abstraction
│   │   ├── IngestMessage.java             # Value object: file paths + tenant + timestamp
│   │   ├── RedisStreamIngestQueue.java    # Impl: Redis Streams (default)
│   │   └── WalIngestQueue.java            # Impl: file-based WAL (future)
│   ├── transfer/
│   │   └── TransferCommitListener.java    # Nhận event từ Transfer Server → push to queue
│   ├── ingest/
│   │   ├── IngestController.java          # POST /api/v1/{tenant}/ingest (batch upload, REST client)
│   │   ├── IngestService.java             # Core: move file + queue.publish()
│   │   ├── IndexingConsumer.java          # queue.consume() → parse → store → batch insert DB
│   │   └── DicomParser.java              # Extract metadata from DICOM file (skip pixel data)
│   ├── selfmanage/
│   │   ├── AutoPartitionService.java      # Tạo partitions tự động 12 tháng trước
│   │   ├── DiskSpaceMonitor.java          # Monitor disk, tự reject ingest khi gần đầy
│   │   └── AutoRecoveryService.java       # Restart indexer khi fail, retry DB connection
│   ├── model/
│   │   ├── Patient.java                   # JPA entity
│   │   ├── Study.java                     # JPA entity
│   │   ├── Series.java                    # JPA entity
│   │   └── Instance.java                 # JPA entity (partitioned by created_date)
│   ├── repository/
│   │   ├── PatientRepository.java
│   │   ├── StudyRepository.java
│   │   ├── SeriesRepository.java
│   │   ├── InstanceRepository.java
│   │   └── BulkInsertRepository.java      # Raw JDBC batch upserts
│   ├── dicomweb/
│   │   ├── QidoController.java            # QIDO-RS: GET /dicomweb/studies, series, instances
│   │   ├── WadoController.java            # WADO-RS: GET /dicomweb/studies/{uid}/...
│   │   ├── StowController.java            # STOW-RS: POST /dicomweb/studies (standard compliant)
│   │   ├── DicomJsonBuilder.java          # Build DICOM JSON response (PS3.18)
│   │   └── QueryBuilder.java             # Dynamic SQL from QIDO query params
│   ├── admin/
│   │   ├── CorrectionController.java      # PUT patient/study metadata + xem audit log
│   │   ├── CorrectionService.java         # Update DB + queue file correction
│   │   ├── FileCorrectionJob.java         # Background: sửa DICOM file headers
│   │   ├── AuditService.java             # Ghi + query audit log
│   │   ├── TenantController.java          # CRUD tenants
│   │   ├── VolumeController.java          # CRUD storage volumes
│   │   ├── LifecycleController.java       # CRUD lifecycle rules
│   │   ├── SystemInfoController.java      # Disk usage, DB stats, queue status
│   │   └── StudyAdminController.java      # Xóa study, merge patient, ...
│   └── schema/
│       └── migration/
│           └── V1__init.sql               # Flyway migration for tenant schema
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   └── db/migration/
│       └── V1__shared_schema.sql          # Shared/public schema (tenant registry)
└── src/test/java/com/spax/
    └── ...
```

---

## Phase 1: Project Setup & Database Schema

### 1.1 Maven Project (pom.xml)

Dependencies:
- `spring-boot-starter-web` (REST API)
- `spring-boot-starter-data-jpa` (JPA cho basic queries)
- `spring-boot-starter-data-redis` (Redis Streams)
- `dcm4che-core` (DICOM parsing)
- `dcm4che-json` (DICOM JSON cho DICOMWeb)
- `org.flywaydb:flyway-core` (schema migration)
- `org.apache.jclouds:jclouds-blobstore` (storage abstraction)
- `org.apache.jclouds.provider:aws-s3` (Amazon S3)
- `org.apache.jclouds.provider:google-cloud-storage` (GCS)
- `org.apache.jclouds.provider:azureblob` (Azure Blob)
- Thêm provider khác khi cần (MinIO dùng `s3` provider + custom endpoint)
- `org.postgresql:postgresql` (JDBC driver)
- `com.zaxxer:HikariCP` (connection pool)

### 1.2 Database Schema

**Shared schema (public)** - tenant registry:

```sql
-- public schema
CREATE TABLE tenant (
    id          SERIAL PRIMARY KEY,
    code        VARCHAR(50) UNIQUE NOT NULL,  -- schema name = 'tenant_' + code
    name        VARCHAR(200) NOT NULL,
    storage_type VARCHAR(20) DEFAULT 'LOCAL', -- LOCAL, S3, BOTH
    storage_config JSONB,                     -- S3 bucket, prefix, etc.
    created_at  TIMESTAMPTZ DEFAULT now(),
    active      BOOLEAN DEFAULT true
);
```

**Tenant schema** (created per tenant, e.g. `tenant_hospital_a`):

**Chiến lược partition:**
- Patient, Study, Series: **KHÔNG partition** (< 500M rows, B-tree index đủ nhanh)
- Instance: **Partition by `created_date` (range, monthly)** — system-generated, luôn đúng
- Không dùng DICOM StudyDate làm partition key (unreliable, máy chụp có thể gửi sai/thiếu)

**Thiết kế lấy ý tưởng từ dcm4chee arc-light:**
- `dicom_attrs BYTEA` trên Study/Series — lưu full DICOM dataset binary, WADO-RS metadata đọc từ đây
- `version` — optimistic locking cho Correction module (tránh race condition concurrent update)
- `study_size`, `series_size` — tổng file size cho admin dashboard
- `num_studies` trên Patient — denormalized count
- `institution`, `station_name`, `sending_aet` trên Series — tracking máy chụp nào gửi
- Instance giữ **lean** (~300 bytes/row) — không có dicom_attrs vì 29 tỷ rows, đọc file header khi cần

```sql
-- Patient table (không partition, < 10M rows)
CREATE TABLE patient (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    patient_id      VARCHAR(64) NOT NULL,
    patient_name    VARCHAR(324),
    patient_name_search TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('simple', coalesce(patient_name, ''))
    ) STORED,
    birth_date      DATE,
    sex             CHAR(1),
    num_studies     INT DEFAULT 0,                    -- dcm4chee: denormalized count
    version         INT DEFAULT 0,                    -- dcm4chee: optimistic locking
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE UNIQUE INDEX idx_patient_pid ON patient (patient_id);
CREATE INDEX idx_patient_name_fts ON patient USING GIN (patient_name_search);

-- Study table (không partition, < 500M rows)
CREATE TABLE study (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    study_instance_uid  VARCHAR(64) NOT NULL,
    study_date          VARCHAR(16),              -- raw DICOM tag, nullable (unreliable)
    study_time          VARCHAR(14),
    study_description   VARCHAR(1024),
    accession_number    VARCHAR(64),
    referring_physician VARCHAR(324),
    modalities_in_study VARCHAR(100),
    num_series          INT DEFAULT 0,
    num_instances       INT DEFAULT 0,
    study_size          BIGINT DEFAULT 0,             -- dcm4chee: tổng file size (bytes)
    dicom_attrs         BYTEA,                        -- dcm4chee: full DICOM dataset binary
    version             INT DEFAULT 0,                -- dcm4chee: optimistic locking
    patient_fk          BIGINT NOT NULL,
    patient_id          VARCHAR(64) NOT NULL,          -- denormalized for common queries
    created_at          TIMESTAMPTZ DEFAULT now(),
    updated_at          TIMESTAMPTZ DEFAULT now()
);

CREATE UNIQUE INDEX idx_study_uid ON study (study_instance_uid);
CREATE INDEX idx_study_patient ON study (patient_id);
CREATE INDEX idx_study_accession ON study (accession_number) WHERE accession_number IS NOT NULL;
CREATE INDEX idx_study_created ON study (created_at DESC);

-- Series table (không partition, < 600M rows)
CREATE TABLE series (
    id                   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    series_instance_uid  VARCHAR(64) NOT NULL,
    series_number        INT,
    modality             VARCHAR(16) NOT NULL,
    series_description   VARCHAR(1024),
    body_part            VARCHAR(64),
    institution          VARCHAR(200),                 -- dcm4chee: institution name
    station_name         VARCHAR(200),                 -- dcm4chee: tên máy chụp
    sending_aet          VARCHAR(64),                  -- dcm4chee: AE title máy gửi
    num_instances        INT DEFAULT 0,
    series_size          BIGINT DEFAULT 0,             -- dcm4chee: tổng file size (bytes)
    dicom_attrs          BYTEA,                        -- dcm4chee: full DICOM dataset binary
    version              INT DEFAULT 0,                -- dcm4chee: optimistic locking
    study_fk             BIGINT NOT NULL,
    study_instance_uid   VARCHAR(64) NOT NULL,          -- denormalized
    created_at           TIMESTAMPTZ DEFAULT now()
);

CREATE UNIQUE INDEX idx_series_uid ON series (series_instance_uid);
CREATE INDEX idx_series_study ON series (study_instance_uid);
CREATE INDEX idx_series_modality ON series (modality);
CREATE INDEX idx_series_station ON series (station_name) WHERE station_name IS NOT NULL;

-- Instance table: PARTITION BY RANGE (created_date) — monthly
-- LEAN schema: ~300 bytes/row — KHÔNG có dicom_attrs (29 tỷ rows, đọc file header khi cần)
-- 29B rows (10 năm) / 120 partitions = 240M rows/partition
CREATE TABLE instance (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY,
    sop_instance_uid    VARCHAR(64) NOT NULL,
    sop_class_uid       VARCHAR(64) NOT NULL,
    instance_number     INT,
    transfer_syntax_uid VARCHAR(64),
    num_frames          INT DEFAULT 1,
    file_size           BIGINT NOT NULL,
    volume_id           INT NOT NULL,
    storage_path        VARCHAR(512) NOT NULL,
    series_fk           BIGINT NOT NULL,
    series_instance_uid VARCHAR(64) NOT NULL,        -- denormalized
    study_instance_uid  VARCHAR(64) NOT NULL,         -- denormalized
    created_date        DATE NOT NULL DEFAULT CURRENT_DATE,  -- partition key (system-generated)
    CONSTRAINT pk_instance PRIMARY KEY (id, created_date)
) PARTITION BY RANGE (created_date);
-- Monthly partitions auto-created by AutoPartitionService (12 tháng trước)

CREATE INDEX idx_instance_sop ON instance (sop_instance_uid);
CREATE INDEX idx_instance_series ON instance (series_instance_uid);
CREATE INDEX idx_instance_study ON instance (study_instance_uid);
```

---

## Phase 2: Core Infrastructure

### 2.1 Multi-Tenant (Schema per tenant)

**TenantContext.java**: ThreadLocal holder cho current tenant code.

**TenantInterceptor.java**: Spring HandlerInterceptor, đọc tenant từ header `X-Tenant-ID` hoặc path `/api/v1/{tenant}/...`. Set vào TenantContext.

**TenantSchemaResolver.java**: Set `search_path` trên JDBC connection dựa vào TenantContext. Implement `ConnectionCustomizer` hoặc dùng Spring's `AbstractRoutingDataSource` để set schema.

**Cách hoạt động:**
```
Request → TenantInterceptor (extract tenant) → TenantContext.set("hospital_a")
    → DataSource connection → SET search_path TO tenant_hospital_a, public
    → All queries run against tenant's schema
```

**TenantManagementService.java**:
- Tạo schema mới cho tenant
- Chạy Flyway migration trên schema tenant
- Tạo monthly partitions tự động (study, instance tables)

### 2.2 Storage Abstraction - Multi-Volume + Tiered Storage + jclouds

**3 vấn đề cần giải quyết:**
1. Đĩa đầy → chuyển sang đĩa khác không downtime
2. Multi-cloud → 1 interface cho S3, GCS, Azure, MinIO, ...
3. Data lifecycle → tự động chuyển file từ hot → cold storage theo thời gian

**Giải pháp tổng quan:**
```
                        ┌─────────────────────────────────┐
                        │         StorageService           │
                        │  store() / retrieve() / delete() │
                        └──────────┬──────────────────────┘
                                   │
                        ┌──────────▼──────────────┐
                        │      VolumeManager       │
                        │  chọn volume theo tier   │
                        │  + status + priority     │
                        └──────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                     ▼
   ┌──────────────────┐  ┌─────────────────┐  ┌─────────────────┐
   │ LocalStorageProvider│  │ JCloudStorageProvider │  │ JCloudStorageProvider │
   │ (Java NIO)        │  │ (aws-s3)        │  │ (google-cloud)  │
   │ /mnt/storage/fs1  │  │ S3 bucket       │  │ GCS bucket      │
   │ tier=HOT          │  │ tier=WARM       │  │ tier=COLD       │
   └──────────────────┘  └─────────────────┘  └─────────────────┘

   ┌──────────────────────────────────────────────────┐
   │              LifecycleService                     │
   │  Rule: study_date > 6 months → move HOT → WARM   │
   │  Rule: study_date > 2 years  → move WARM → COLD  │
   │  Background job, batch migration                  │
   └──────────────────────────────────────────────────┘
```

**Storage Provider SPI:**
```java
// Mỗi loại storage implement interface này
public interface StorageProvider {
    void write(String relativePath, InputStream data, long size);
    InputStream read(String relativePath);
    void delete(String relativePath);
    boolean exists(String relativePath);
    // copy giữa 2 provider (dùng cho migration)
    default void copyFrom(StorageProvider source, String srcPath, String destPath) {
        try (InputStream in = source.read(srcPath)) { write(destPath, in, -1); }
    }
}
```

**LocalStorageProvider.java**: Java NIO, ghi file lên local filesystem.

**JCloudStorageProvider.java**: Dùng Apache jclouds BlobStore API.
- 1 class duy nhất, config provider name khi tạo volume:
  - `aws-s3` → Amazon S3
  - `google-cloud-storage` → Google Cloud Storage
  - `azureblob` → Azure Blob Storage
  - `s3` → S3-compatible (MinIO, Ceph, etc.)
  - `b2` → Backblaze B2
- jclouds abstract hết, chỉ cần đổi provider + credentials

**Database (shared schema - public):**
```sql
CREATE TABLE storage_volume (
    id              SERIAL PRIMARY KEY,
    code            VARCHAR(50) UNIQUE NOT NULL,   -- 'local-nvme', 'aws-s3-hot', 'gcs-archive'
    provider_type   VARCHAR(30) NOT NULL,          -- 'LOCAL', 'aws-s3', 'google-cloud-storage', 'azureblob', 's3' (minio)
    base_path       VARCHAR(500) NOT NULL,         -- local: '/mnt/storage/fs1', cloud: prefix path
    tier            VARCHAR(10) NOT NULL DEFAULT 'HOT',  -- HOT, WARM, COLD
    status          VARCHAR(20) DEFAULT 'ACTIVE',  -- ACTIVE, READ_ONLY, OFFLINE
    priority        INT DEFAULT 0,                 -- trong cùng tier, ưu tiên ghi vào priority cao hơn
    total_bytes     BIGINT,
    used_bytes      BIGINT DEFAULT 0,
    -- Cloud credentials (nullable, chỉ dùng cho cloud volumes)
    cloud_endpoint  VARCHAR(500),                  -- custom endpoint (MinIO, etc.)
    cloud_region    VARCHAR(50),
    cloud_bucket    VARCHAR(200),
    cloud_identity  VARCHAR(500),                  -- access key / service account
    cloud_credential TEXT,                          -- secret key / JSON key (encrypted)
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Lifecycle rules
CREATE TABLE lifecycle_rule (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    enabled         BOOLEAN DEFAULT true,
    source_tier     VARCHAR(10) NOT NULL,          -- 'HOT'
    target_tier     VARCHAR(10) NOT NULL,          -- 'WARM'
    condition_type  VARCHAR(30) NOT NULL,           -- 'STUDY_AGE_DAYS', 'LAST_ACCESS_DAYS'
    condition_value INT NOT NULL,                   -- 180 (days)
    delete_source   BOOLEAN DEFAULT true,           -- xóa file ở source sau khi copy xong
    tenant_code     VARCHAR(50),                    -- NULL = áp dụng tất cả tenant
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Migration tracking
CREATE TABLE migration_task (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_code     VARCHAR(50) NOT NULL,
    rule_id         INT REFERENCES lifecycle_rule(id),
    instance_id     BIGINT NOT NULL,
    study_date      DATE NOT NULL,
    source_volume_id INT NOT NULL,
    target_volume_id INT NOT NULL,
    status          VARCHAR(20) DEFAULT 'PENDING',  -- PENDING, IN_PROGRESS, COMPLETED, FAILED
    error_message   TEXT,
    created_at      TIMESTAMPTZ DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_migration_status ON migration_task (status) WHERE status != 'COMPLETED';
```

**Instance table:**
```sql
    volume_id       INT NOT NULL,                  -- FK → storage_volume.id (biết file ở volume nào)
    storage_path    VARCHAR(512) NOT NULL,          -- relative path trong volume
```

**VolumeManager.java** (core):
- Cache danh sách volumes in-memory, reload khi gọi admin API
- `getActiveWriteVolume(String tier)` → trả volume ACTIVE có priority cao nhất trong tier
- Mặc định ghi vào tier HOT, lifecycle service sẽ chuyển sang WARM/COLD sau
- Auto-detect đĩa đầy cho LOCAL volumes (check free space trước khi ghi)
- `getProvider(int volumeId)` → trả StorageProvider tương ứng (Local hoặc JCloud)

**StorageService.java** (facade):
```java
public interface StorageService {
    StoreResult store(String tenantCode, byte[] data, String studyUid, String seriesUid, String sopUid);
    InputStream retrieve(int volumeId, String storagePath);
    void delete(int volumeId, String storagePath);
    void migrate(int sourceVolumeId, String sourcePath, int targetVolumeId, String targetPath);
}
```

**Relative path format** (giống nhau cho mọi volume/provider):
```
{tenantCode}/{hash[0:2]}/{hash[2:4]}/{studyUid}/{seriesUid}/{sopUid}.dcm
```

**Admin API cho volume management:**
| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/admin/volumes` | Liệt kê tất cả volumes + tier + status + usage |
| `POST /api/v1/admin/volumes` | Thêm volume mới |
| `PUT /api/v1/admin/volumes/{id}` | Cập nhật volume (status, priority, ...) |
| `POST /api/v1/admin/volumes/reload` | Reload volume config từ DB |
| `GET /api/v1/admin/lifecycle/rules` | Liệt kê lifecycle rules |
| `POST /api/v1/admin/lifecycle/rules` | Tạo rule mới |
| `PUT /api/v1/admin/lifecycle/rules/{id}` | Cập nhật rule |
| `POST /api/v1/admin/lifecycle/run` | Trigger migration thủ công |
| `GET /api/v1/admin/lifecycle/status` | Xem tiến độ migration |

**Ví dụ sử dụng:**
```bash
# 1. Thêm local NVMe làm HOT storage
POST /api/v1/admin/volumes
{"code": "local-nvme", "providerType": "LOCAL", "basePath": "/mnt/nvme/dicom",
 "tier": "HOT", "status": "ACTIVE"}

# 2. local-nvme đầy → chuyển READ_ONLY, thêm ổ mới
PUT /api/v1/admin/volumes/1  {"status": "READ_ONLY"}
POST /api/v1/admin/volumes
{"code": "local-nvme2", "providerType": "LOCAL", "basePath": "/mnt/nvme2/dicom",
 "tier": "HOT", "status": "ACTIVE"}

# 3. Thêm AWS S3 làm WARM storage
POST /api/v1/admin/volumes
{"code": "aws-s3-warm", "providerType": "aws-s3", "basePath": "dicom/",
 "tier": "WARM", "status": "ACTIVE",
 "cloudBucket": "hospital-pacs-warm", "cloudRegion": "ap-southeast-1",
 "cloudIdentity": "AKIA...", "cloudCredential": "secret..."}

# 4. Thêm Google Cloud Storage làm COLD archive
POST /api/v1/admin/volumes
{"code": "gcs-cold", "providerType": "google-cloud-storage", "basePath": "archive/",
 "tier": "COLD", "status": "ACTIVE",
 "cloudBucket": "hospital-pacs-archive",
 "cloudIdentity": "service-account@project.iam.gserviceaccount.com",
 "cloudCredential": "{...service account JSON...}"}

# 5. Tạo lifecycle rules
POST /api/v1/admin/lifecycle/rules
{"name": "Hot to Warm after 6 months", "sourceTier": "HOT", "targetTier": "WARM",
 "conditionType": "STUDY_AGE_DAYS", "conditionValue": 180, "deleteSource": true}

POST /api/v1/admin/lifecycle/rules
{"name": "Warm to Cold after 2 years", "sourceTier": "WARM", "targetTier": "COLD",
 "conditionType": "STUDY_AGE_DAYS", "conditionValue": 730, "deleteSource": true}
```

### 2.3 Data Lifecycle Service

**LifecycleService.java**:
- Scheduled job (chạy hàng đêm hoặc theo cấu hình)
- Evaluate từng rule: query instances ở source_tier thỏa điều kiện (tuổi study > N ngày)
- Tạo migration_task records
- Worker threads thực hiện migration: copy file → update instance.volume_id → xóa file cũ (nếu deleteSource=true)
- Batch processing: migrate theo study (tất cả instances của 1 study cùng lúc)
- Idempotent: nếu fail giữa chừng, retry được

**Migration flow cho 1 instance:**
```
1. Copy file: sourceProvider.read(path) → targetProvider.write(path)
2. Verify: targetProvider.exists(path) + check file size
3. Update DB: UPDATE instance SET volume_id = targetVolumeId WHERE id = ?
4. Delete source: sourceProvider.delete(path)  (nếu deleteSource=true)
5. Mark migration_task = COMPLETED
```

### 2.4 Ingest Queue Abstraction

**Mục tiêu**: Tách interface để swap Redis ↔ WAL mà không sửa business logic.

```java
public interface IngestQueue {
    /** Publish file path vào queue */
    void publish(IngestMessage message);

    /** Consume batch messages, gọi handler, tự ACK khi handler thành công */
    void consume(int batchSize, Consumer<List<IngestMessage>> handler);
}

public record IngestMessage(
    String filePath,
    String tenantCode,
    Instant receivedAt
) {}
```

**IngestService** và **IndexingConsumer** chỉ depend vào `IngestQueue` interface:

```
IngestService → ingestQueue.publish(msg)        // không biết Redis hay WAL
IndexingConsumer → ingestQueue.consume(200, handler)  // không biết Redis hay WAL
```

**Impl 1: RedisStreamIngestQueue (default)**
- Stream name: `spax:ingest:{tenantCode}`
- Consumer group: `indexer-group`
- Multiple consumers (1 consumer per worker thread)
- Dùng Spring Data Redis StreamOperations

**Impl 2: WalIngestQueue (future - khi muốn bỏ Redis)**
- Append-only WAL file trên disk
- Background thread đọc + xử lý
- Crash recovery: đọc lại entries PENDING khi restart
- Không cần infrastructure bên ngoài

**Chuyển đổi**: Chỉ đổi 1 dòng config:
```yaml
spax:
  queue:
    type: redis    # hoặc: wal
```

Spring `@ConditionalOnProperty` tự chọn đúng implementation.

### 2.5 Self-Management Services (Zero-Touch Operation)

**Yêu cầu**: 1000 điểm triển khai, không có người monitor. Hệ thống phải tự vận hành.

**AutoPartitionService.java** (chạy mỗi ngày 2:00 AM):
```
- Kiểm tra partition tồn tại cho 12 tháng tới
- Thiếu → CREATE TABLE instance_YYYY_MM PARTITION OF instance ...
- Thất bại → log error + gửi alert về central
- Không cần con người can thiệp
```

**DiskSpaceMonitor.java** (chạy mỗi 5 phút):
```
- Check free space trên mỗi mount point
- > 20% free  → OK
- 10-20% free → WARNING → gửi alert
- < 10% free  → CRITICAL → tạm reject ingest (trả HTTP 507)
- < 5% free   → EMERGENCY → chỉ cho phép read
- Tự bảo vệ: không để PostgreSQL crash vì disk đầy
```

**AutoRecoveryService.java**:
```
- Indexer crash → catch → log → wait 5s → restart
- 3 lần fail liên tiếp → retry mỗi 60s
- DB connection lost → retry backoff (5s, 10s, 30s, 60s)
- DICOM parse fail → move file sang /error/ → skip → tiếp tục
- KHÔNG dừng cả pipeline vì 1 file lỗi
```

> **Mở rộng sau (không implement giai đoạn đầu):**
> - `HealthReporter` — gửi heartbeat/metrics về Central Server
> - `RemoteConfigService` — pull config từ central, không cần SSH
> - Chỉ cần khi có Central Server và site expose được network ra ngoài

---

## Phase 3: Ingest Pipeline

### 3.0 Tổng quan các đường ingest

```
Đường 1 (chính):  Orthanc Gateway → [Transfers Plugin] → Transfer Server (đã implement)
                                                            → files on temp dir
                                                            → commit xong
                                                            → TransferCommitListener
                                                            → ingestQueue.publish()

Đường 2 (REST):   Any client → POST /api/v1/{tenant}/ingest (multipart)
                                → IngestController
                                → lưu file vào temp
                                → ingestQueue.publish()

Đường 3 (STOW):   DICOMWeb client → POST /dicomweb/{tenant}/studies (STOW-RS)
                                → StowController
                                → lưu file vào temp
                                → ingestQueue.publish()

Tất cả 3 đường hội tụ vào cùng pipeline:
    ingestQueue → IndexingConsumer → parse → move file → index DB
```

### 3.1 Transfer Server Integration

**TransferCommitListener.java**: Được gọi khi Transfer Server hoàn tất commit.

**Input**: Danh sách file paths trên temp dir + tenant code.

**Flow:**
1. Transfer Server nhận đủ files từ Orthanc → commit thành công
2. TransferCommitListener nhận event (callback hoặc scan temp dir)
3. Với mỗi file: `ingestQueue.publish(IngestMessage{filePath, tenantCode, receivedAt})`
4. Transfer Server trả OK cho Orthanc

### 3.2 Batch Ingest API (REST)

**IngestController.java**: `POST /api/v1/{tenant}/ingest`
- Accept: `multipart/form-data` với nhiều DICOM files
- Flow: nhận file → lưu vào temp dir → `ingestQueue.publish()` → return 200 OK

### 3.3 Async Indexing Worker

**IndexingConsumer.java**: Consume từ queue, xử lý batch.

**Flow:**
```
1. Consume batch messages từ queue (max 200 messages/lần)
2. Với mỗi file trên temp dir:
   a. Parse DICOM header (skip pixel data) → extract metadata
   b. Move file từ temp → permanent storage (volume)
      Path: {volume}/{tenant}/{hash}/{studyUid}/{seriesUid}/{sopUid}.dcm
3. Group metadata: patient → study → series → instances
4. Batch upsert vào PostgreSQL (1 transaction):
   - UPSERT patients (ON CONFLICT patient_id DO UPDATE)
   - UPSERT studies (ON CONFLICT study_instance_uid DO UPDATE)
   - UPSERT series (ON CONFLICT series_instance_uid DO UPDATE)
   - Bulk INSERT instances
5. ACK messages trong queue
6. Nếu fail 1 file → move sang /error/ → log → tiếp tục
```

**DicomParser.java**:
- Dùng `DicomInputStream` với `IncludeBulkData.NO` → skip pixel data
- Extract tags cần thiết cho 4 levels (Patient/Study/Series/Instance)

**BulkInsertRepository.java**: Raw JDBC, không dùng JPA cho insert path.
- Prepared statement batching
- `INSERT ... ON CONFLICT ... DO UPDATE` cho patient, study, series
- Batch size 200 instances per flush

---

## Phase 4: DICOMWeb API (cho OHIF Viewer)

### 4.1 QIDO-RS (Query)

**QidoController.java**:

| Endpoint | Purpose |
|----------|---------|
| `GET /dicomweb/{tenant}/studies` | Search studies |
| `GET /dicomweb/{tenant}/studies/{studyUid}/series` | Get series of a study |
| `GET /dicomweb/{tenant}/studies/{studyUid}/series/{seriesUid}/instances` | Get instances |

**Query params**: PatientName, PatientID, StudyDate, Modality, AccessionNumber, StudyDescription, limit, offset, includefield

**Response**: `application/dicom+json`

**QueryBuilder.java**: Build dynamic SQL từ QIDO query params.
- Wildcard support: `PatientName=DOE*` → `LIKE 'DOE%'`
- Date range: `StudyDate=20240101-20240131` → `BETWEEN`
- Pagination: `limit` + `offset`
- Luôn kèm partition key (`study_date`) trong WHERE clause khi có thể → partition pruning

### 4.2 WADO-RS (Retrieve)

**WadoController.java**:

| Endpoint | Accept | Purpose |
|----------|--------|---------|
| `GET /dicomweb/{tenant}/studies/{studyUid}` | `multipart/related` | All instances of study |
| `GET /dicomweb/{tenant}/studies/{studyUid}/series/{seriesUid}` | `multipart/related` | All instances of series |
| `GET /dicomweb/{tenant}/studies/{studyUid}/series/{seriesUid}/instances/{sopUid}` | `application/dicom` | Single instance |
| `GET /dicomweb/{tenant}/.../instances/{sopUid}/metadata` | `application/dicom+json` | Metadata only |
| `GET /dicomweb/{tenant}/.../instances/{sopUid}/frames/{frames}` | `multipart/related` | Specific frames |

**Flow**: Query DB for storage_path → Read file from storage → Stream to client

### 4.3 STOW-RS (Store - DICOMWeb standard)

**StowController.java**: `POST /dicomweb/{tenant}/studies`
- Accept: `multipart/related; type="application/dicom"`
- Reuse IngestService logic
- Return DICOM XML/JSON response per standard

### 4.4 DicomJsonBuilder.java
- Convert DB records → DICOM JSON format (PS3.18)
- OHIF viewer expects specific format: `{ "00100010": { "vr": "PN", "Value": [{"Alphabetic": "DOE^JOHN"}] } }`

---

## Phase 5: Administration Module

### 5.1 DICOM Correction (sửa thông tin bệnh nhân / study)

**Use case**: Chụp cấp cứu → KTV nhập sai/thiếu thông tin → cần sửa lại.

**Flow:**
```
Admin sửa PatientName
    │
    ├─① Update DB ngay (instant) → QIDO trả về data mới
    │
    ├─② Queue background job → sửa DICOM file headers (4000 files async)
    │
    └─③ Ghi audit log → ai sửa, sửa gì, lúc nào, giá trị cũ/mới
```

**Database (tenant schema):**
```sql
CREATE TABLE audit_log (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    entity_type     VARCHAR(20) NOT NULL,    -- 'PATIENT', 'STUDY', 'SERIES'
    entity_id       BIGINT NOT NULL,
    field_name      VARCHAR(100) NOT NULL,
    old_value       TEXT,
    new_value       TEXT,
    changed_by      VARCHAR(100) NOT NULL,
    changed_at      TIMESTAMPTZ DEFAULT now(),
    reason          VARCHAR(500)
);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);

CREATE TABLE file_correction_task (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    study_instance_uid VARCHAR(64) NOT NULL,
    correction_type VARCHAR(50) NOT NULL,
    changes         JSONB NOT NULL,
    total_files     INT NOT NULL,
    processed_files INT DEFAULT 0,
    status          VARCHAR(20) DEFAULT 'PENDING',
    created_at      TIMESTAMPTZ DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

**CorrectionController.java:**

| Endpoint | Purpose |
|----------|---------|
| `PUT /api/v1/{tenant}/admin/patients/{patientId}` | Sửa patient (name, birthdate, sex) |
| `PUT /api/v1/{tenant}/admin/studies/{studyUid}` | Sửa study (description, accession, referring) |
| `GET /api/v1/{tenant}/admin/audit` | Xem lịch sử thay đổi |
| `GET /api/v1/{tenant}/admin/corrections` | Xem tiến độ file correction jobs |

**FileCorrectionJob.java** (background):
```
Cho mỗi DICOM file trong study:
  1. Đọc file từ storage (via StorageService)
  2. Parse DICOM header (skip pixel data)
  3. Sửa tag values
  4. Ghi file lại (overwrite in-place)
  5. Update processed_files count
  6. Nếu fail 1 file → log error → tiếp tục file khác
```

### 5.2 Study Management

**StudyAdminController.java:**

| Endpoint | Purpose |
|----------|---------|
| `DELETE /api/v1/{tenant}/admin/studies/{studyUid}` | Xóa study + series + instances + files |
| `POST /api/v1/{tenant}/admin/patients/merge` | Merge 2 patient records (duplicate) |
| `GET /api/v1/{tenant}/admin/stats` | Thống kê: số study, instance, disk usage |

### 5.3 System Administration

**TenantController.java:**

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/admin/tenants` | Liệt kê tenants |
| `POST /api/v1/admin/tenants` | Tạo tenant mới (auto create schema + partitions) |
| `PUT /api/v1/admin/tenants/{code}` | Cập nhật tenant |

**VolumeController.java:**

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/admin/volumes` | Liệt kê volumes + tier + status + usage |
| `POST /api/v1/admin/volumes` | Thêm volume mới |
| `PUT /api/v1/admin/volumes/{id}` | Cập nhật volume (status, priority) |

**SystemInfoController.java:**

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/health` | Health check (DB, Redis, storage) |
| `GET /api/v1/admin/system/info` | Disk usage, DB size, queue status, partition info |

---

## Phase 6: Self-Management & Operations

### 6.1 AutoPartitionService
- Chạy hàng ngày, tạo trước partitions cho 12 tháng tới
- Chỉ bảng `instance` (các bảng khác không partition)
- Pattern: `instance_2026_01`, `instance_2026_02`, ...

### 6.2 DiskSpaceMonitor, AutoRecoveryService
- (Như mô tả ở Phase 2.5)

---

## Implementation Order

| Order | Component | Dependencies | Est. Files |
|-------|-----------|-------------|------------|
| 1 | Project setup (pom.xml, application.yml) | None | 3 |
| 2 | Database schema (Flyway migrations) | #1 | 2 |
| 3 | Multi-tenant infrastructure | #1 | 5 |
| 4 | Storage: Provider SPI + Local + jclouds | #1 | 5 |
| 5 | Storage: VolumeManager + Admin API | #2,4 | 3 |
| 6 | Queue abstraction + Redis Streams impl | #1 | 4 |
| 7 | DICOM parser | #1 | 1 |
| 8 | Ingest pipeline (controller + service + consumer) | #2,3,5,6,7 | 4 |
| 9 | Bulk insert repository | #2,3 | 1 |
| 10 | DICOMWeb QIDO-RS | #2,3,9 | 3 |
| 11 | DICOMWeb WADO-RS | #2,3,5 | 1 |
| 12 | DICOMWeb STOW-RS | #8 | 1 |
| 13 | Admin: Correction + Audit | #2,3,5,7 | 4 |
| 14 | Admin: Study management + System info | #2,3 | 3 |
| 15 | Admin: Tenant + Volume management | #2,3,5 | 2 |
| 16 | Data Lifecycle (rules + migration job) | #2,5 | 4 |
| 17 | Self-management (partition, disk, recovery) | #2,3 | 3 |
| 18 | Health endpoint | All | 1 |

---

## Key Configuration (application.yml)

```yaml
server:
  port: 8080
  servlet:
    multipart:
      max-file-size: 2GB
      max-request-size: 10GB

spring:
  threads:
    virtual:
      enabled: true  # Java 21 virtual threads
  datasource:
    url: jdbc:postgresql://localhost:5432/spax
    hikari:
      maximum-pool-size: 30
      minimum-idle: 10
      connection-timeout: 5000
  data:
    redis:
      host: localhost
      port: 6379

spax:
  storage:
    # Volumes được quản lý trong DB (bảng storage_volume)
    # Dùng Admin API để thêm/sửa volumes
    disk-space-threshold-mb: 5120   # cảnh báo khi ổ đĩa còn < 5GB free
  ingest:
    batch-size: 200              # instances per DB flush
    flush-interval-ms: 2000      # max time between flushes
    consumer-threads: 4          # Redis Stream consumers
  partition:
    months-ahead: 6              # pre-create partitions
```

---

## Verification Plan

1. **Start dependencies**: PostgreSQL + Redis (docker-compose)
2. **Boot application**: `mvn spring-boot:run`
3. **Create tenant**: `POST /api/v1/admin/tenants` với `{"code": "test", "name": "Test Hospital"}`
4. **Ingest test DICOM files**: `curl -X POST -F "file=@test.dcm" http://localhost:8080/api/v1/test/ingest`
5. **Verify indexing**: Check Redis Stream lag → 0, query DB for records
6. **QIDO-RS**: `curl http://localhost:8080/dicomweb/test/studies` → should return DICOM JSON
7. **WADO-RS**: `curl http://localhost:8080/dicomweb/test/studies/{uid}/series/{uid}/instances/{uid}` → should return DICOM file
8. **OHIF integration**: Configure OHIF viewer to point to SPAX DICOMWeb endpoint
9. **Load test**: Batch ingest 10,000 DICOM files, verify all indexed correctly
10. **Partition check**: Verify monthly partitions created in tenant schema
