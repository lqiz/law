# EvidenceVault v1.0 — Skills 规格清单

> 设计原则：每个 Skill 是一个**纯函数式原子操作**——相同输入必须产生相同输出，
> 任何第三方（法官、鉴定机构、对方律师）可独立复算验证。

---

## S01: HashFile

**用途**：对任意文件计算密码学摘要，生成内容指纹。

**输入**：
```json
{
  "file_path": "string   // 本地沙箱内文件绝对路径",
  "algorithm": "string   // 固定值 'SHA-256'"
}
```

**输出**：
```json
{
  "hash_hex": "string          // 64 字符十六进制哈希值",
  "algorithm": "SHA-256",
  "file_size_bytes": "number   // 原始文件字节数",
  "computed_at": "string       // ISO 8601 本地时间（仅记录，不作为可信时间）"
}
```

**不可做的事**：
- 不可在哈希前对文件做任何转码、压缩、裁剪
- 不可使用 MD5 / SHA-1 等已被证明碰撞不安全的算法
- 不可将文件内容传到任何外部服务

**可复算验证**：任何人拿到原始文件，执行 `sha256sum <file>` 即可得到相同哈希值。

---

## S02: CollectDeviceMeta

**用途**：在证据采集瞬间，抓取设备环境快照，证明证据来源。

**输入**：
```json
{
  "trigger": "string   // 触发事件：'capture_start' | 'capture_end'"
}
```

**输出**：
```json
{
  "device_model": "string       // 如 'iPhone 15 Pro' / 'Pixel 8'",
  "os_version": "string         // 如 'iOS 18.2' / 'Android 15'",
  "app_version": "string        // EvidenceVault 构建版本号",
  "locale": "string             // 如 'zh-CN'",
  "timezone": "string           // 如 'Asia/Shanghai'",
  "gps_lat": "number | null     // 纬度，用户授权时采集",
  "gps_lng": "number | null     // 经度，用户授权时采集",
  "gps_accuracy_m": "number | null",
  "ntp_time_utc": "string       // NTP 校准后的 UTC 时间 ISO 8601",
  "local_clock_utc": "string    // 设备本地时钟 UTC 时间 ISO 8601",
  "clock_drift_ms": "number     // ntp_time - local_clock 的差值毫秒",
  "network_type": "string       // 'wifi' | 'cellular' | 'none'",
  "battery_level": "number      // 0-100"
}
```

**不可做的事**：
- 不可伪造或覆写任何硬件返回值
- 不可在 GPS 未授权时填入默认坐标（必须为 null）
- 不可缓存上一次的元数据复用

**可复算验证**：虽然环境快照本身不可重放，但 `clock_drift_ms` 可与运营商基站记录交叉验证；GPS 可与基站定位交叉比对。

---

## S03: RequestTimestamp

**用途**：将哈希值发送到 RFC 3161 TSA 服务器，获取不可抵赖的时间证明。

**输入**：
```json
{
  "hash_hex": "string        // S01 输出的 SHA-256 哈希",
  "algorithm": "SHA-256",
  "tsa_url": "string         // TSA 服务地址",
  "nonce": "string           // 随机数，防重放攻击"
}
```

**输出**：
```json
{
  "tsr_bytes_base64": "string   // RFC 3161 TimeStampResp 的 Base64 编码",
  "tsa_cert_chain": "string     // TSA 证书链 PEM 格式",
  "granted_time_utc": "string   // TSA 签发的可信时间 ISO 8601",
  "serial_number": "string      // TSA 返回的序列号",
  "status": "string             // 'granted' | 'rejected' | 'network_error'"
}
```

**不可做的事**：
- 不可将原始文件内容发送到 TSA（只发哈希）
- 不可修改 TSA 返回的任何字段
- 不可在本地伪造 TSR
- 不可使用未经认证的 TSA 服务（v1.0 必须为国内有资质机构）

**可复算验证**：任何人使用 `openssl ts -verify -in resp.tsr -data <hash> -CAfile tsa_ca.pem` 可独立验证。

---

## S04: EncryptFile

**用途**：对原始证据文件进行本地加密存储，防止设备丢失后泄露。

**输入**：
```json
{
  "file_path": "string          // 明文文件路径",
  "key_ref": "string            // Keystore/Keychain 中的密钥引用 ID",
  "algorithm": "AES-256-GCM"
}
```

**输出**：
```json
{
  "encrypted_file_path": "string   // 加密后文件路径（.evault 扩展名）",
  "iv_hex": "string                // 初始化向量十六进制",
  "auth_tag_hex": "string          // GCM 认证标签十六进制",
  "original_hash_hex": "string     // 加密前原文件的 SHA-256（来自 S01）"
}
```

**不可做的事**：
- 不可使用 ECB 模式或其他非认证加密模式
- 不可将加密密钥明文写入数据库或日志
- 不可复用 IV（每次加密必须生成新的随机 IV）
- 加密后不可删除 `original_hash_hex` 的记录（解密后需比对完整性）

**可复算验证**：解密后对文件执行 S01，哈希必须与 `original_hash_hex` 完全一致。

---

## S05: DecryptFile

**用途**：解密证据文件用于查看或导出，并自动验证完整性。

**输入**：
```json
{
  "encrypted_file_path": "string   // .evault 文件路径",
  "key_ref": "string               // Keystore/Keychain 密钥引用",
  "iv_hex": "string",
  "auth_tag_hex": "string",
  "expected_hash_hex": "string     // 原始文件的期望 SHA-256"
}
```

**输出**：
```json
{
  "decrypted_file_path": "string   // 解密后的临时文件路径",
  "integrity_check": "string       // 'pass' | 'fail'",
  "actual_hash_hex": "string       // 解密后实际计算的 SHA-256"
}
```

**不可做的事**：
- 如果 GCM auth_tag 验证失败，不可输出解密内容（必须中止并报告篡改）
- 如果 `integrity_check` 为 `fail`，不可允许该文件进入导出流程
- 解密后的临时文件在查看结束后必须安全擦除

**可复算验证**：`actual_hash_hex === expected_hash_hex` 且与 S01 原始记录一致。

---

## S06: BuildEvidenceRecord

**用途**：将一次采集的所有产物（文件、哈希、元数据、时间戳）组装为一条完整的证据记录。

**输入**：
```json
{
  "evidence_id": "string          // UUIDv4",
  "case_id": "string              // 所属案件 ID",
  "file_hash": "S01.output        // HashFile 的完整输出",
  "device_meta": "S02.output      // CollectDeviceMeta 的完整输出",
  "timestamp_resp": "S03.output   // RequestTimestamp 的完整输出",
  "encrypt_info": "S04.output     // EncryptFile 的完整输出",
  "media_type": "string           // 'photo' | 'audio' | 'video' | 'screenshot' | 'document'",
  "user_note": "string            // 用户备注，可为空"
}
```

**输出**：
```json
{
  "evidence_record": {
    "evidence_id": "string",
    "case_id": "string",
    "created_at": "string",
    "media_type": "string",
    "user_note": "string",
    "file_hash": { "..." },
    "device_meta": { "..." },
    "timestamp_resp": { "..." },
    "encrypt_info": { "..." },
    "record_hash": "string   // 对以上所有字段序列化后的 SHA-256"
  }
}
```

**不可做的事**：
- 不可遗漏任何子模块输出（所有字段必须存在）
- 不可在组装后修改任何子字段
- `record_hash` 必须是对确定性 JSON 序列化（键排序 + 无空格）的哈希
- 不可将 `user_note` 纳入 `record_hash` 计算（备注允许后续修改）

**可复算验证**：取出 record 中除 `record_hash` 和 `user_note` 外的所有字段，按键排序 JSON 序列化后 SHA-256，结果必须等于 `record_hash`。

---

## S07: VerifyEvidenceRecord

**用途**：对已存储的证据记录执行全链路完整性校验。这是**司法验证的核心入口**。

**输入**：
```json
{
  "evidence_record": "S06.output   // 完整的 EvidenceRecord",
  "encrypted_file_path": "string   // 加密文件路径（可选，用于文件级验证）"
}
```

**输出**：
```json
{
  "checks": [
    {"name": "record_hash_integrity",   "result": "pass|fail", "detail": "string"},
    {"name": "file_hash_consistency",    "result": "pass|fail", "detail": "string"},
    {"name": "tsa_signature_valid",      "result": "pass|fail", "detail": "string"},
    {"name": "tsa_hash_matches_file",    "result": "pass|fail", "detail": "string"},
    {"name": "tsa_cert_chain_trusted",   "result": "pass|fail", "detail": "string"},
    {"name": "encryption_integrity",     "result": "pass|fail|skipped", "detail": "string"}
  ],
  "overall": "string   // 'all_passed' | 'has_failures'",
  "verified_at": "string"
}
```

**不可做的事**：
- 不可跳过任何一项检查
- 不可在某项失败时自动修复（只报告，不修改）
- 不可缓存校验结果（每次必须重新计算）

**可复算验证**：这个 Skill 本身就是验证器。第三方使用相同输入执行相同检查，必须得到相同结果。

---

## S08: ExportEvidencePackage

**用途**：将一个案件的全部证据打包为司法提交格式的证据包。

**输入**：
```json
{
  "case_id": "string",
  "evidence_ids": ["string"],
  "export_path": "string            // 导出目标目录",
  "include_verification_guide": true
}
```

**输出**：
```json
{
  "package_path": "string              // 生成的 ZIP 文件路径",
  "package_hash": "string              // ZIP 文件整体 SHA-256",
  "evidence_count": "number",
  "manifest": {
    "items": [
      {
        "evidence_id": "string",
        "original_filename": "string",
        "hash_hex": "string",
        "tsa_time": "string"
      }
    ]
  },
  "structure": "见下方目录树"
}
```

导出包内部结构：
```
EvidencePackage_{case_id}_{date}/
├── evidence/                          # 解密后的原始文件
│   ├── 001_{media_type}_{date}.{ext}
│   └── ...
├── certificates/                      # TSA 时间戳响应文件
│   ├── 001_{media_type}_{date}.tsr
│   └── ...
├── metadata/                          # 每条证据的完整记录 JSON
│   ├── 001_{media_type}_{date}.json
│   └── ...
├── hash_manifest.json                 # 全部文件哈希清单
├── chain_of_custody.json              # 证据链操作日志
├── verification_guide.html            # 验证指南（含命令行验证步骤）
└── README.txt                         # 证据包说明
```

**不可做的事**：
- 不可导出 `VerifyEvidenceRecord` 未全部通过的证据（必须先校验）
- 不可对原始文件做任何格式转换（保留原始格式）
- 不可在导出包中包含加密密钥
- 导出完成后必须安全擦除解密临时文件

**可复算验证**：对导出包中每个文件执行 `sha256sum`，与 `hash_manifest.json` 逐一比对。

---

## S09: SecureWipe

**用途**：安全删除指定文件或全部数据，不可恢复。

**输入**：
```json
{
  "target": "string           // 'single_file' | 'single_case' | 'all_data'",
  "target_id": "string|null   // 文件路径或 case_id，all_data 时为 null",
  "confirmation": "string     // 用户输入的确认码（如 'DELETE-{case_id}'）"
}
```

**输出**：
```json
{
  "wiped_files_count": "number",
  "wiped_db_records_count": "number",
  "wipe_method": "string         // 'overwrite_3pass_then_delete'",
  "completed_at": "string"
}
```

**不可做的事**：
- 不可在没有用户二次确认的情况下执行
- 不可只删除文件引用而不删除实际数据
- 不可删除正在导出中的证据
- 对于 `all_data`，必须同时清除数据库、加密文件、临时文件、日志

**可复算验证**：删除后尝试读取目标路径和数据库记录，均应返回不存在。

---

## S10: AuditLog

**用途**：记录每一次 Skill 调用，形成不可篡改的操作审计链。

**输入**：
```json
{
  "action": "string            // Skill 名称，如 'HashFile'",
  "evidence_id": "string|null  // 关联的证据 ID",
  "case_id": "string|null",
  "input_summary": "string     // 输入摘要（不含敏感数据）",
  "output_summary": "string    // 输出摘要",
  "result": "string            // 'success' | 'failure'",
  "error_detail": "string|null"
}
```

**输出**：
```json
{
  "log_entry": {
    "log_id": "string           // UUIDv4",
    "sequence_no": "number      // 单调递增序列号",
    "timestamp_utc": "string",
    "action": "string",
    "evidence_id": "string|null",
    "case_id": "string|null",
    "input_summary": "string",
    "output_summary": "string",
    "result": "string",
    "error_detail": "string|null",
    "prev_log_hash": "string    // 上一条日志的哈希（链式结构）",
    "this_log_hash": "string    // 本条日志全字段哈希"
  }
}
```

**不可做的事**：
- 不可删除或修改已写入的日志条目
- 不可跳过序列号（必须严格连续）
- `prev_log_hash` 不可断链（第一条记录为 `"GENESIS"`）
- 不可在日志中记录原始文件内容或加密密钥

**可复算验证**：从第一条日志开始，逐条计算哈希并验证 `prev_log_hash` 链是否连续未断裂。

---

## Skills 依赖关系（流水线视图）

```
采集阶段:
  [用户操作] → S02:CollectDeviceMeta → S01:HashFile → S03:RequestTimestamp
                                                    → S04:EncryptFile
                                                            ↓
组装阶段:                                     S06:BuildEvidenceRecord
                                                            ↓
全程伴随:                                     S10:AuditLog（每步自动触发）

查看阶段:
  S05:DecryptFile → [用户查看] → SecureWipe(临时文件)

验证阶段:
  S07:VerifyEvidenceRecord（可在任意时刻独立执行）

导出阶段:
  S07:Verify(前置) → S05:Decrypt → S08:ExportEvidencePackage

销毁阶段:
  S09:SecureWipe
```

---

## 司法验证路径（给鉴定机构 / 对方律师 / 法官）

```
第三方拿到证据包后的验证步骤：

1. 解压证据包
2. 对 evidence/ 中每个文件执行 sha256sum        → 比对 hash_manifest.json ✓
3. 对每个 .tsr 文件执行 openssl ts -verify      → 验证 TSA 签名 ✓
4. 确认 TSR 中的哈希值 = 文件哈希值              → 证明"该文件在 TSA 时间前已存在" ✓
5. 检查 chain_of_custody.json 的哈希链连续性     → 证明操作记录未被篡改 ✓
6. 全部通过 → 证据具备"内容完整 + 时间可信 + 操作可追溯"三要素
```
