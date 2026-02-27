# SPEC-02: Database Schema (Flyway Migrations)

## Mục tiêu
Tạo 2 Flyway migration files:
1. **Shared schema** (`public`): tenant registry, storage volumes, lifecycle rules
2. **Tenant schema** template: patient, study, series, instance, audit, tasks

## Dependencies
- SPEC-01 (project setup, package structure)

## Files cần tạo

### 1. `src/main/resources/db/migration/V1__shared_schema.sql`

Chứa tất cả bảng trong `public` schema (shared across all tenants):

```sql
-- Tenant registry
CREATE TABLE IF NOT EXISTS tenant (
    id          SERIAL PRIMARY KEY,
    code        VARCHAR(50) UNIQUE NOT NULL,
    name        VARCHAR(200) NOT NULL,
    storage_type VARCHAR(20) DEFAULT 'LOCAL',
    storage_config JSONB,
    created_at  TIMESTAMPTZ DEFAULT now(),
    active      BOOLEAN DEFAULT true
);

-- Storage volumes (multi-volume pool)
CREATE TABLE IF NOT EXISTS storage_volume (
    id              SERIAL PRIMARY KEY,
    code            VARCHAR(50) UNIQUE NOT NULL,
    provider_type   VARCHAR(30) NOT NULL,      -- 'LOCAL', 'aws-s3', 'google-cloud-storage', 'azureblob', 's3'
    base_path       VARCHAR(500) NOT NULL,
    tier            VARCHAR(10) NOT NULL DEFAULT 'HOT',   -- HOT, WARM, COLD
    status          VARCHAR(20) DEFAULT 'ACTIVE',          -- ACTIVE, READ_ONLY, OFFLINE
    priority        INT DEFAULT 0,
    total_bytes     BIGINT,
    used_bytes      BIGINT DEFAULT 0,
    path_template   VARCHAR(500),              -- NULL = use default from config
    cloud_endpoint  VARCHAR(500),
    cloud_region    VARCHAR(50),
    cloud_bucket    VARCHAR(200),
    cloud_identity  VARCHAR(500),
    cloud_credential TEXT,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Lifecycle rules (MIGRATE or COMPRESS)
CREATE TABLE IF NOT EXISTS lifecycle_rule (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    enabled         BOOLEAN DEFAULT true,
    action_type     VARCHAR(20) DEFAULT 'MIGRATE',   -- 'MIGRATE' | 'COMPRESS'
    source_tier     VARCHAR(10) NOT NULL,
    target_tier     VARCHAR(10),               -- used only for MIGRATE
    condition_type  VARCHAR(30) NOT NULL,       -- 'STUDY_AGE_DAYS', 'LAST_ACCESS_DAYS'
    condition_value INT NOT NULL,
    delete_source   BOOLEAN DEFAULT true,       -- used only for MIGRATE
    compression_type VARCHAR(50),              -- used only for COMPRESS, e.g. 'JPEG_LS_LOSSLESS'
    tenant_code     VARCHAR(50),               -- NULL = apply to all tenants
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Migration tasks tracking (public schema, references instances by ID across tenants)
CREATE TABLE IF NOT EXISTS migration_task (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_code     VARCHAR(50) NOT NULL,
    rule_id         INT REFERENCES lifecycle_rule(id),
    instance_id     BIGINT NOT NULL,
    study_date      DATE NOT NULL,
    source_volume_id INT NOT NULL,
    target_volume_id INT NOT NULL,
    status          VARCHAR(20) DEFAULT 'PENDING',   -- PENDING, IN_PROGRESS, COMPLETED, FAILED
    error_message   TEXT,
    created_at      TIMESTAMPTZ DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX IF NOT EXISTS idx_migration_status ON migration_task (status) WHERE status != 'COMPLETED';
CREATE INDEX IF NOT EXISTS idx_migration_tenant ON migration_task (tenant_code, status);
```

### 2. `src/main/java/com/spax/schema/migration/V1__init.sql`

Đây là template SQL cho tenant schema. **Không phải Flyway migration trực tiếp** — được đọc bởi `TenantManagementService` để execute khi tạo tenant mới. Flyway không chạy file này.

```sql
-- Patient table
CREATE TABLE patient (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id       CHAR(40) NOT NULL,
    patient_id      VARCHAR(64) NOT NULL,
    patient_name    VARCHAR(324),
    patient_name_search TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('simple', coalesce(patient_name, ''))
    ) STORED,
    birth_date      DATE,
    sex             CHAR(1),
    is_provisional  BOOLEAN DEFAULT false,
    num_studies     INT DEFAULT 0,
    version         INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);
CREATE UNIQUE INDEX idx_patient_public_id ON patient (public_id);
CREATE INDEX idx_patient_pid ON patient (patient_id);
CREATE INDEX idx_patient_name_fts ON patient USING GIN (patient_name_search);
CREATE INDEX idx_patient_provisional ON patient (is_provisional) WHERE is_provisional = true;

-- Study table
CREATE TABLE study (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id           CHAR(40) NOT NULL,
    study_instance_uid  VARCHAR(64) NOT NULL,
    study_date          VARCHAR(16),
    study_time          VARCHAR(14),
    study_description   VARCHAR(1024),
    accession_number    VARCHAR(64),
    referring_physician VARCHAR(324),
    modalities_in_study VARCHAR(100),
    num_series          INT DEFAULT 0,
    num_instances       INT DEFAULT 0,
    study_size          BIGINT DEFAULT 0,
    version             INT DEFAULT 0,
    patient_fk          BIGINT NOT NULL REFERENCES patient(id),
    patient_id          VARCHAR(64) NOT NULL,
    created_at          TIMESTAMPTZ DEFAULT now(),
    updated_at          TIMESTAMPTZ DEFAULT now(),
    last_accessed_at    TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_study_public_id ON study (public_id);
CREATE INDEX idx_study_uid ON study (study_instance_uid);
CREATE INDEX idx_study_patient ON study (patient_fk);
CREATE INDEX idx_study_accession ON study (accession_number) WHERE accession_number IS NOT NULL;
CREATE INDEX idx_study_created ON study (created_at DESC);
CREATE INDEX idx_study_last_accessed ON study (last_accessed_at) WHERE last_accessed_at IS NOT NULL;

-- Series table
CREATE TABLE series (
    id                   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    series_instance_uid  VARCHAR(64) NOT NULL,
    series_number        INT,
    modality             VARCHAR(16) NOT NULL,
    series_description   VARCHAR(1024),
    body_part            VARCHAR(64),
    institution          VARCHAR(200),
    station_name         VARCHAR(200),
    sending_aet          VARCHAR(64),
    num_instances        INT DEFAULT 0,
    series_size          BIGINT DEFAULT 0,
    version              INT DEFAULT 0,
    compress_tsuid       VARCHAR(64),
    compress_time        TIMESTAMPTZ,
    metadata_volume_id   INT,           -- volume chứa pre-built metadata cache file
    metadata_path        VARCHAR(512),  -- path đến DICOM JSON array (PS3.18), NULL = chưa build
    study_fk             BIGINT NOT NULL REFERENCES study(id),
    study_instance_uid   VARCHAR(64) NOT NULL,
    created_at           TIMESTAMPTZ DEFAULT now()
);
CREATE UNIQUE INDEX idx_series_unique ON series (study_fk, series_instance_uid);
CREATE INDEX idx_series_uid ON series (series_instance_uid);
CREATE INDEX idx_series_modality ON series (modality);
CREATE INDEX idx_series_station ON series (station_name) WHERE station_name IS NOT NULL;

-- Instance table (partitioned by created_date)
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
    series_instance_uid VARCHAR(64) NOT NULL,
    study_instance_uid  VARCHAR(64) NOT NULL,
    created_date        DATE NOT NULL DEFAULT CURRENT_DATE,
    CONSTRAINT pk_instance PRIMARY KEY (id, created_date)
) PARTITION BY RANGE (created_date);

CREATE INDEX idx_instance_dedup ON instance (series_fk, sop_instance_uid);
CREATE INDEX idx_instance_sop ON instance (sop_instance_uid, created_date);
CREATE INDEX idx_instance_study ON instance (study_instance_uid, created_date);
CREATE INDEX idx_instance_series ON instance (series_fk, created_date);
CREATE INDEX idx_instance_volume ON instance (volume_id);

-- Audit log
CREATE TABLE audit_log (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    entity_type     VARCHAR(20) NOT NULL,
    entity_id       BIGINT NOT NULL,
    field_name      VARCHAR(100) NOT NULL,
    old_value       TEXT,
    new_value       TEXT,
    changed_by      VARCHAR(100) NOT NULL,
    changed_at      TIMESTAMPTZ DEFAULT now(),
    reason          VARCHAR(500)
);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_changed ON audit_log (changed_at DESC);

-- File correction tasks
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
CREATE INDEX idx_file_correction_status ON file_correction_task (status) WHERE status != 'COMPLETED';

-- Compression tasks
CREATE TABLE compression_task (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    study_instance_uid  VARCHAR(64) NOT NULL,
    compression_type    VARCHAR(50) NOT NULL,
    total_files         INT NOT NULL,
    processed_files     INT DEFAULT 0,
    failed_files        INT DEFAULT 0,
    status              VARCHAR(20) DEFAULT 'PENDING',
    triggered_by        VARCHAR(100),
    rule_id             INT,
    error_summary       TEXT,
    created_at          TIMESTAMPTZ DEFAULT now(),
    completed_at        TIMESTAMPTZ
);
CREATE INDEX idx_compression_task_study ON compression_task (study_instance_uid);
CREATE INDEX idx_compression_task_status ON compression_task (status) WHERE status != 'COMPLETED';
```

## Lưu ý quan trọng
- File `V1__init.sql` trong `src/main/java/com/spax/schema/migration/` là **resource file** được load bởi `TenantManagementService` (sẽ làm ở SPEC-03), không phải Flyway migration
- Flyway chỉ chạy files trong `src/main/resources/db/migration/`
- Không tạo default partition cho `instance` table ở đây — `AutoPartitionService` (SPEC-17) sẽ tạo theo schedule
- Không có FK từ tenant schema sang public schema (cross-schema FK không reliable trong multi-tenant setup)

## Kiểm tra thành công
- Application boot thành công (Flyway chạy `V1__shared_schema.sql`)
- `public.tenant`, `public.storage_volume`, `public.lifecycle_rule`, `public.migration_task` tồn tại trong DB
- Không có lỗi SQL syntax
