# EvidenceVault v1.0 — SQLCipher 数据库 Schema

> 所有结构化数据存储在单一 SQLCipher 加密数据库中。
> 数据库密钥由 Master Key (Keystore/Keychain) 保护。

---

## 1. 数据库基础配置

```sql
-- SQLCipher 配置
PRAGMA cipher_compatibility = 4;      -- SQLCipher 4.x
PRAGMA kdf_iter = 256000;             -- PBKDF2 迭代次数
PRAGMA cipher_page_size = 4096;       -- 页大小
PRAGMA journal_mode = WAL;            -- Write-Ahead Logging（并发读写）
PRAGMA foreign_keys = ON;             -- 强制外键约束
PRAGMA auto_vacuum = INCREMENTAL;     -- 增量自动清理
```

---

## 2. 表关系总览

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│   t_case     │────<│  t_evidence_item │────<│ t_hash_proof │
│  (案件)      │ 1:N │  (证据条目)       │ 1:N │ (存证证明)   │
└─────────────┘     └──────────────────┘     └──────────────┘
                           │ 1:N
                    ┌──────┴───────┐
                    │              │
             ┌──────▼─────┐ ┌─────▼──────────┐
             │ t_audio    │ │ t_evidence_tag │
             │ _segment   │ │ (标签关联)      │
             │ (录音分段)  │ └────────────────┘
             └────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│ t_key_store  │     │ t_audit_log  │     │ t_export_record  │
│ (密钥存储)    │     │ (审计日志)    │     │ (导出记录)        │
└──────────────┘     └──────────────┘     └──────────────────┘

┌──────────────┐
│ t_app_config │
│ (应用配置)    │
└──────────────┘
```

---

## 3. 完整 DDL

### 3.1 t_case — 案件表

```sql
CREATE TABLE t_case (
    case_id              TEXT PRIMARY KEY,           -- UUIDv4
    case_title           TEXT NOT NULL,              -- 用户自定义案件名称
    case_id_short        TEXT NOT NULL UNIQUE,       -- 短 ID: CASE-{date}-{hash4}
    description          TEXT DEFAULT '',            -- 案件描述
    status               TEXT NOT NULL DEFAULT 'active'
                         CHECK(status IN ('active', 'archived', 'exported', 'wiped')),
    evidence_count       INTEGER NOT NULL DEFAULT 0, -- 冗余计数，触发器维护
    next_sequence_no     INTEGER NOT NULL DEFAULT 1, -- 下一个证据序号
    created_at           TEXT NOT NULL,              -- ISO 8601 UTC
    updated_at           TEXT NOT NULL,              -- ISO 8601 UTC
    archived_at          TEXT                        -- 归档时间
);

CREATE INDEX idx_case_status ON t_case(status);
CREATE INDEX idx_case_created ON t_case(created_at);
```

### 3.2 t_evidence_item — 证据条目表

```sql
CREATE TABLE t_evidence_item (
    evidence_id          TEXT PRIMARY KEY,           -- UUIDv4
    case_id              TEXT NOT NULL REFERENCES t_case(case_id) ON DELETE RESTRICT,
    sequence_no          INTEGER NOT NULL,           -- 案件内序号

    -- 分类
    media_type           TEXT NOT NULL
                         CHECK(media_type IN ('photo','audio','video','screenshot','document')),
    evidence_category    TEXT NOT NULL
                         CHECK(evidence_category IN (
                             'overtime','verbal_instruction','written_instruction',
                             'salary_record','attendance','reward_or_praise',
                             'harassment','termination_notice','contract',
                             'communication','other'
                         )),

    -- 文件引用
    original_filename    TEXT NOT NULL,
    mime_type            TEXT NOT NULL,
    file_size_bytes      INTEGER NOT NULL CHECK(file_size_bytes > 0),
    file_hash_hex        TEXT NOT NULL,              -- SHA-256, 64 chars
    encrypted_path       TEXT NOT NULL,              -- 沙箱相对路径

    -- 加密参数
    encrypt_algorithm    TEXT NOT NULL DEFAULT 'AES-256-GCM',
    encrypt_key_ref      TEXT NOT NULL,              -- 引用 t_key_store.key_id
    encrypt_iv_hex       TEXT NOT NULL,              -- 12 bytes = 24 hex
    encrypt_auth_tag_hex TEXT NOT NULL,              -- 16 bytes = 32 hex

    -- 采集上下文
    capture_ntp_time     TEXT NOT NULL,              -- NTP 校准 UTC ISO 8601
    capture_local_time   TEXT NOT NULL,              -- 设备本地时钟 UTC ISO 8601
    capture_clock_drift  INTEGER NOT NULL,           -- 毫秒
    capture_timezone     TEXT NOT NULL,              -- IANA timezone
    capture_device_model TEXT NOT NULL,
    capture_os_version   TEXT NOT NULL,
    capture_app_version  TEXT NOT NULL,
    capture_locale       TEXT NOT NULL DEFAULT 'zh-CN',
    capture_gps_lat      REAL,                       -- 纬度，可 NULL
    capture_gps_lng      REAL,                       -- 经度，可 NULL
    capture_gps_accuracy REAL,                       -- 米，可 NULL
    capture_network      TEXT CHECK(capture_network IN ('wifi','cellular','none')),
    capture_battery      INTEGER CHECK(capture_battery BETWEEN 0 AND 100),

    -- 完整性
    record_hash_hex      TEXT NOT NULL,              -- 全字段哈希
    hash_input_fields    TEXT NOT NULL,              -- JSON 数组: ["field1","field2",...]

    -- 可变域
    user_note            TEXT DEFAULT '',
    is_starred           INTEGER NOT NULL DEFAULT 0, -- 0=false, 1=true

    -- 生命周期
    status               TEXT NOT NULL DEFAULT 'sealing'
                         CHECK(status IN ('sealing','sealed','seal_failed','exported','wiped')),
    created_at           TEXT NOT NULL,
    last_verified_at     TEXT,
    last_exported_at     TEXT,
    verify_fail_count    INTEGER NOT NULL DEFAULT 0,

    -- 录音专有（非录音时为 NULL）
    audio_metadata_json  TEXT,                       -- JSON: codec, duration, segments 等

    UNIQUE(case_id, sequence_no)
);

CREATE INDEX idx_evidence_case ON t_evidence_item(case_id);
CREATE INDEX idx_evidence_status ON t_evidence_item(status);
CREATE INDEX idx_evidence_category ON t_evidence_item(evidence_category);
CREATE INDEX idx_evidence_capture_time ON t_evidence_item(capture_ntp_time);
CREATE INDEX idx_evidence_hash ON t_evidence_item(file_hash_hex);
```

### 3.3 t_hash_proof — 哈希存证证明表

```sql
CREATE TABLE t_hash_proof (
    proof_id             TEXT PRIMARY KEY,           -- UUIDv4
    evidence_id          TEXT NOT NULL REFERENCES t_evidence_item(evidence_id) ON DELETE RESTRICT,
    is_primary           INTEGER NOT NULL DEFAULT 0, -- 1=主证明（参与 record_hash）

    -- 通道
    channel              TEXT NOT NULL
                         CHECK(channel IN ('tsa_rfc3161','judicial_chain','public_chain','local_mockup')),
    status               TEXT NOT NULL DEFAULT 'pending'
                         CHECK(status IN ('pending','requesting','granted','failed','retry_scheduled')),

    -- 输入
    input_hash_algorithm TEXT NOT NULL DEFAULT 'SHA-256',
    input_hash_hex       TEXT NOT NULL,              -- 被存证的哈希
    input_nonce          TEXT NOT NULL,              -- 防重放随机数

    -- TSA 输出（channel = tsa_rfc3161 时有值）
    tsa_url              TEXT,
    tsa_granted_time     TEXT,                       -- ISO 8601 UTC
    tsa_serial_number    TEXT,
    tsa_response_base64  TEXT,                       -- 完整 TSR
    tsa_cert_chain_pem   TEXT,
    tsa_response_status  INTEGER,                    -- 0=granted

    -- 司法链输出（channel = judicial_chain 时有值）
    chain_name           TEXT,                       -- tianping / zhixin / antchain
    chain_tx_hash        TEXT,
    chain_block_height   INTEGER,
    chain_block_time     TEXT,
    chain_certificate_url TEXT,
    chain_raw_response   TEXT,                       -- Base64

    -- 公链输出（channel = public_chain 时有值）
    pub_chain_id         TEXT,                       -- ethereum_mainnet / polygon
    pub_tx_hash          TEXT,
    pub_block_number     INTEGER,
    pub_block_time       TEXT,
    pub_explorer_url     TEXT,

    -- 本地模拟输出（channel = local_mockup 时有值）
    mock_granted_time    TEXT,
    mock_signature_hex   TEXT,

    -- 重试
    retry_count          INTEGER NOT NULL DEFAULT 0,
    last_error           TEXT,
    next_retry_at        TEXT,                       -- ISO 8601 UTC

    -- 时间
    created_at           TEXT NOT NULL,
    updated_at           TEXT NOT NULL
);

CREATE INDEX idx_proof_evidence ON t_hash_proof(evidence_id);
CREATE INDEX idx_proof_status ON t_hash_proof(status);
CREATE INDEX idx_proof_retry ON t_hash_proof(next_retry_at) WHERE status = 'retry_scheduled';
```

### 3.4 t_audio_segment — 录音分段表

```sql
CREATE TABLE t_audio_segment (
    segment_id           TEXT PRIMARY KEY,           -- UUIDv4
    evidence_id          TEXT NOT NULL REFERENCES t_evidence_item(evidence_id) ON DELETE CASCADE,
    segment_index        INTEGER NOT NULL,           -- 段序号 (0-based)

    -- 时间
    offset_sec           REAL NOT NULL,              -- 在完整录音中的偏移秒数
    duration_sec         REAL NOT NULL,              -- 段时长
    ntp_timestamp        TEXT NOT NULL,              -- 段开始的 NTP 时间

    -- 内容
    audio_hash_hex       TEXT NOT NULL,              -- 段内容 SHA-256
    chain_hash_hex       TEXT NOT NULL,              -- 链式哈希
    size_bytes           INTEGER NOT NULL,

    -- 加密
    encrypted_path       TEXT NOT NULL,              -- 段加密文件路径
    encrypt_iv_hex       TEXT NOT NULL,
    encrypt_auth_tag_hex TEXT NOT NULL,

    -- 状态
    status               TEXT NOT NULL DEFAULT 'written'
                         CHECK(status IN ('writing','written','merged','wiped')),

    UNIQUE(evidence_id, segment_index)
);

CREATE INDEX idx_segment_evidence ON t_audio_segment(evidence_id);
```

### 3.5 t_evidence_tag — 证据标签关联表

```sql
CREATE TABLE t_evidence_tag (
    evidence_id          TEXT NOT NULL REFERENCES t_evidence_item(evidence_id) ON DELETE CASCADE,
    tag_name             TEXT NOT NULL,              -- 标签文本，最长 50 字符
    created_at           TEXT NOT NULL,

    PRIMARY KEY (evidence_id, tag_name)
);

CREATE INDEX idx_tag_name ON t_evidence_tag(tag_name);
```

### 3.6 t_key_store — 密钥元数据表

```sql
-- 注意：实际密钥材料存储在 Keystore/Keychain 中
-- 此表只存储密钥元数据和经 MK 加密后的 DEK 密文
CREATE TABLE t_key_store (
    key_id               TEXT PRIMARY KEY,           -- 密钥引用 ID
    key_type             TEXT NOT NULL
                         CHECK(key_type IN ('master_key_ref','data_encryption_key')),
    case_id              TEXT REFERENCES t_case(case_id), -- DEK 关联案件，MK 为 NULL
    algorithm            TEXT NOT NULL DEFAULT 'AES-256',
    encrypted_key_base64 TEXT,                       -- DEK 经 MK 加密后的密文（MK 本身此字段为 NULL）
    keystore_alias       TEXT,                       -- Keystore/Keychain 中的别名（MK 有此字段）
    status               TEXT NOT NULL DEFAULT 'active'
                         CHECK(status IN ('active','rotated','destroyed')),
    created_at           TEXT NOT NULL,
    rotated_at           TEXT
);

CREATE INDEX idx_key_case ON t_key_store(case_id);
```

### 3.7 t_audit_log — 审计日志表

```sql
CREATE TABLE t_audit_log (
    log_id               TEXT PRIMARY KEY,           -- UUIDv4
    sequence_no          INTEGER NOT NULL UNIQUE,    -- 全局单调递增
    timestamp_utc        TEXT NOT NULL,              -- ISO 8601 UTC

    action               TEXT NOT NULL,              -- 操作名称
    evidence_id          TEXT,                       -- 关联证据（可 NULL）
    case_id              TEXT,                       -- 关联案件（可 NULL）

    input_summary        TEXT NOT NULL DEFAULT '',   -- 输入摘要
    output_summary       TEXT NOT NULL DEFAULT '',   -- 输出摘要
    result               TEXT NOT NULL
                         CHECK(result IN ('success','failure')),
    error_detail         TEXT,

    prev_log_hash        TEXT NOT NULL,              -- 上一条日志哈希（首条为 "GENESIS"）
    this_log_hash        TEXT NOT NULL               -- 本条日志哈希
);

-- sequence_no 上已有 UNIQUE，查询直接用它
CREATE INDEX idx_audit_action ON t_audit_log(action);
CREATE INDEX idx_audit_evidence ON t_audit_log(evidence_id);
CREATE INDEX idx_audit_time ON t_audit_log(timestamp_utc);
```

### 3.8 t_export_record — 导出记录表

```sql
CREATE TABLE t_export_record (
    export_id            TEXT PRIMARY KEY,           -- UUIDv4
    case_id              TEXT NOT NULL REFERENCES t_case(case_id),
    export_time          TEXT NOT NULL,              -- ISO 8601 UTC
    evidence_count       INTEGER NOT NULL,
    evidence_ids_json    TEXT NOT NULL,              -- JSON 数组
    package_hash_hex     TEXT NOT NULL,              -- ZIP 包 SHA-256
    export_path          TEXT,                       -- 导出路径（参考）
    status               TEXT NOT NULL DEFAULT 'completed'
                         CHECK(status IN ('generating','completed','failed'))
);

CREATE INDEX idx_export_case ON t_export_record(case_id);
```

### 3.9 t_app_config — 应用配置表

```sql
CREATE TABLE t_app_config (
    config_key           TEXT PRIMARY KEY,
    config_value         TEXT NOT NULL,
    updated_at           TEXT NOT NULL
);

-- 预置配置项
INSERT INTO t_app_config VALUES ('schema_version',       '1.0.0',         datetime('now'));
INSERT INTO t_app_config VALUES ('app_lock_enabled',     'true',          datetime('now'));
INSERT INTO t_app_config VALUES ('biometric_enabled',    'false',         datetime('now'));
INSERT INTO t_app_config VALUES ('tsa_primary_url',      'https://tsa.tsa.cn/', datetime('now'));
INSERT INTO t_app_config VALUES ('tsa_fallback_url',     'https://freetsa.org/tsr', datetime('now'));
INSERT INTO t_app_config VALUES ('segment_duration_sec', '30',            datetime('now'));
INSERT INTO t_app_config VALUES ('hash_proof_mode',      'tsa_rfc3161',   datetime('now'));
INSERT INTO t_app_config VALUES ('ntp_offset_ms',        '0',             datetime('now'));
INSERT INTO t_app_config VALUES ('last_ntp_sync',        '',              datetime('now'));
```

---

## 4. 触发器

### 4.1 证据计数自动维护

```sql
CREATE TRIGGER trg_evidence_count_insert
AFTER INSERT ON t_evidence_item
BEGIN
    UPDATE t_case
    SET evidence_count = evidence_count + 1,
        next_sequence_no = next_sequence_no + 1,
        updated_at = datetime('now')
    WHERE case_id = NEW.case_id;
END;

CREATE TRIGGER trg_evidence_count_delete
AFTER DELETE ON t_evidence_item
BEGIN
    UPDATE t_case
    SET evidence_count = evidence_count - 1,
        updated_at = datetime('now')
    WHERE case_id = OLD.case_id;
END;
```

### 4.2 审计日志序列号自动递增

```sql
-- 使用应用层生成（SQLite 无序列对象）
-- 在插入前通过 SELECT COALESCE(MAX(sequence_no), 0) + 1 FROM t_audit_log 获取
```

---

## 5. 数据迁移策略

```sql
-- 未来 schema 升级时:
-- 1. 读取 t_app_config.schema_version
-- 2. 按版本号依次执行迁移脚本
-- 3. 更新 schema_version

-- 示例: v1.0.0 → v1.1.0（增加司法链字段）
-- ALTER TABLE t_hash_proof ADD COLUMN chain_verify_url TEXT;
```
