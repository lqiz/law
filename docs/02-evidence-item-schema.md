# EvidenceVault v1.0 — EvidenceItem 数据结构规格

---

## 1. 设计原则

| 原则 | 规则 |
|------|------|
| 完整性边界 | 字段分为 **integrity 域**（参与哈希、不可改）和 **mutable 域**（允许修改、不参与哈希） |
| 访问分级 | 每个字段标注 `display`（UI 可展示）或 `verify_only`（仅司法校验使用，UI 不暴露） |
| 零冗余 | 原始文件不存在此结构中，只存引用路径和哈希 |

**字段标记说明**：
- 🔒 `[H]` = 参与 integrity hash 计算
- 👁 `[D]` = UI 可展示给用户
- 🔬 `[V]` = 仅用于司法校验，UI 不暴露原始值

---

## 2. 顶层结构总览

```
EvidenceItem
├── identity        # 身份标识
├── classification  # 证据分类
├── file_ref        # 文件引用与完整性
├── capture_context # 采集上下文（设备 + 环境）
├── encryption      # 加密存储信息
├── timestamp_proof # 时间戳证明
├── integrity       # 整体完整性校验
├── mutable         # 可修改域（不参与哈希）
└── lifecycle       # 生命周期状态
```

---

## 3. 完整 JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://evidencevault.local/schemas/evidence-item/v1.0.0",
  "title": "EvidenceItem",
  "description": "EvidenceVault 证据对象 v1.0 — 职场证据的最小完整单元",
  "type": "object",
  "required": [
    "schema_version",
    "identity",
    "classification",
    "file_ref",
    "capture_context",
    "encryption",
    "timestamp_proof",
    "integrity",
    "mutable",
    "lifecycle"
  ],
  "additionalProperties": false,

  "properties": {

    "schema_version": {
      "const": "1.0.0",
      "description": "[H][D] 数据结构版本，用于未来迁移"
    },

    "identity": {
      "type": "object",
      "description": "证据身份标识",
      "required": ["evidence_id", "case_id"],
      "additionalProperties": false,
      "properties": {
        "evidence_id": {
          "type": "string",
          "format": "uuid",
          "description": "[H][D] 证据全局唯一 ID，UUIDv4，创建时生成"
        },
        "case_id": {
          "type": "string",
          "format": "uuid",
          "description": "[H][D] 所属案件 ID"
        },
        "sequence_no": {
          "type": "integer",
          "minimum": 1,
          "description": "[H][D] 案件内证据序号，单调递增"
        }
      }
    },

    "classification": {
      "type": "object",
      "description": "证据分类",
      "required": ["media_type", "evidence_category"],
      "additionalProperties": false,
      "properties": {
        "media_type": {
          "type": "string",
          "enum": ["photo", "audio", "video", "screenshot", "document"],
          "description": "[H][D] 文件媒体类型"
        },
        "evidence_category": {
          "type": "string",
          "enum": [
            "overtime",
            "verbal_instruction",
            "written_instruction",
            "salary_record",
            "attendance",
            "reward_or_praise",
            "harassment",
            "termination_notice",
            "contract",
            "communication",
            "other"
          ],
          "description": "[H][D] 证据事项分类"
        },
        "evidence_category_label": {
          "type": "string",
          "description": "分类中文标签，仅用于展示，由 enum 映射生成",
          "enum": [
            "加班记录",
            "口头指令",
            "书面指令",
            "工资/报酬记录",
            "考勤记录",
            "嘉奖/表扬",
            "骚扰/歧视",
            "解雇/终止通知",
            "劳动合同",
            "沟通记录",
            "其他"
          ]
        }
      }
    },

    "file_ref": {
      "type": "object",
      "description": "原始文件引用与内容指纹",
      "required": [
        "original_filename",
        "mime_type",
        "file_size_bytes",
        "hash_algorithm",
        "file_hash_hex",
        "encrypted_storage_path"
      ],
      "additionalProperties": false,
      "properties": {
        "original_filename": {
          "type": "string",
          "description": "[H][D] 原始文件名"
        },
        "mime_type": {
          "type": "string",
          "description": "[H][D] MIME 类型，如 image/jpeg, audio/aac"
        },
        "file_size_bytes": {
          "type": "integer",
          "minimum": 1,
          "description": "[H][D] 原始文件大小（字节）"
        },
        "hash_algorithm": {
          "const": "SHA-256",
          "description": "[H][V] 哈希算法标识"
        },
        "file_hash_hex": {
          "type": "string",
          "pattern": "^[a-f0-9]{64}$",
          "description": "[H][V] 原始文件 SHA-256 哈希值（64 字符十六进制）"
        },
        "encrypted_storage_path": {
          "type": "string",
          "description": "[V] 加密后文件在沙箱内的相对路径"
        }
      }
    },

    "capture_context": {
      "type": "object",
      "description": "采集瞬间的设备与环境快照",
      "required": [
        "device_model",
        "os_version",
        "app_version",
        "ntp_time_utc",
        "local_clock_utc",
        "clock_drift_ms",
        "timezone"
      ],
      "additionalProperties": false,
      "properties": {
        "device_model": {
          "type": "string",
          "description": "[H][V] 设备型号"
        },
        "os_version": {
          "type": "string",
          "description": "[H][V] 操作系统版本"
        },
        "app_version": {
          "type": "string",
          "description": "[H][V] App 构建版本"
        },
        "ntp_time_utc": {
          "type": "string",
          "format": "date-time",
          "description": "[H][D] NTP 校准后的采集时间（UTC ISO 8601）— 展示用主时间"
        },
        "local_clock_utc": {
          "type": "string",
          "format": "date-time",
          "description": "[H][V] 设备本地时钟 UTC 时间"
        },
        "clock_drift_ms": {
          "type": "integer",
          "description": "[H][V] NTP 与本地时钟偏差（毫秒），用于鉴定时钟是否被篡改"
        },
        "timezone": {
          "type": "string",
          "description": "[H][D] IANA 时区，如 Asia/Shanghai"
        },
        "locale": {
          "type": "string",
          "description": "[H][V] 系统语言区域"
        },
        "gps": {
          "oneOf": [
            {
              "type": "object",
              "required": ["lat", "lng", "accuracy_m"],
              "additionalProperties": false,
              "properties": {
                "lat":        { "type": "number", "description": "[H][D] 纬度" },
                "lng":        { "type": "number", "description": "[H][D] 经度" },
                "accuracy_m": { "type": "number", "description": "[H][V] 精度（米）" }
              }
            },
            { "type": "null" }
          ],
          "description": "GPS 坐标，未授权时为 null"
        },
        "network_type": {
          "type": "string",
          "enum": ["wifi", "cellular", "none"],
          "description": "[H][V] 网络类型"
        },
        "battery_level": {
          "type": "integer",
          "minimum": 0,
          "maximum": 100,
          "description": "[H][V] 电池电量百分比"
        }
      }
    },

    "encryption": {
      "type": "object",
      "description": "本地加密存储参数",
      "required": ["algorithm", "key_ref", "iv_hex", "auth_tag_hex"],
      "additionalProperties": false,
      "properties": {
        "algorithm": {
          "const": "AES-256-GCM",
          "description": "[H][V] 加密算法"
        },
        "key_ref": {
          "type": "string",
          "description": "[V] Keystore/Keychain 密钥引用 ID（非密钥本身）"
        },
        "iv_hex": {
          "type": "string",
          "pattern": "^[a-f0-9]{24}$",
          "description": "[V] 初始化向量（12 字节 = 24 hex chars）"
        },
        "auth_tag_hex": {
          "type": "string",
          "pattern": "^[a-f0-9]{32}$",
          "description": "[V] GCM 认证标签（16 字节 = 32 hex chars）"
        }
      }
    },

    "timestamp_proof": {
      "type": "object",
      "description": "RFC 3161 可信时间戳证明",
      "required": ["status"],
      "additionalProperties": false,
      "properties": {
        "status": {
          "type": "string",
          "enum": ["granted", "pending", "failed"],
          "description": "[H][D] 时间戳状态"
        },
        "tsa_url": {
          "type": "string",
          "format": "uri",
          "description": "[H][V] TSA 服务地址"
        },
        "granted_time_utc": {
          "type": "string",
          "format": "date-time",
          "description": "[H][D] TSA 签发的可信时间（UTC ISO 8601）"
        },
        "tsr_base64": {
          "type": "string",
          "description": "[V] RFC 3161 TimeStampResp 完整 Base64 编码"
        },
        "tsa_cert_chain_pem": {
          "type": "string",
          "description": "[V] TSA 证书链 PEM"
        },
        "tsa_serial_number": {
          "type": "string",
          "description": "[V] TSA 返回的序列号"
        },
        "nonce": {
          "type": "string",
          "description": "[H][V] 请求随机数（防重放）"
        },
        "request_hash_hex": {
          "type": "string",
          "pattern": "^[a-f0-9]{64}$",
          "description": "[H][V] 发送给 TSA 的哈希值（必须等于 file_ref.file_hash_hex）"
        }
      }
    },

    "integrity": {
      "type": "object",
      "description": "整体完整性校验",
      "required": ["record_hash_hex", "hash_input_fields"],
      "additionalProperties": false,
      "properties": {
        "record_hash_hex": {
          "type": "string",
          "pattern": "^[a-f0-9]{64}$",
          "description": "[V] 对所有 [H] 字段确定性序列化后的 SHA-256"
        },
        "hash_input_fields": {
          "type": "array",
          "items": { "type": "string" },
          "description": "[V] 参与哈希计算的字段路径列表（自描述，便于第三方复算）"
        }
      }
    },

    "mutable": {
      "type": "object",
      "description": "可修改域 — 不参与完整性哈希",
      "additionalProperties": false,
      "properties": {
        "user_note": {
          "type": "string",
          "maxLength": 2000,
          "description": "[D] 用户备注说明"
        },
        "tags": {
          "type": "array",
          "items": { "type": "string", "maxLength": 50 },
          "maxItems": 20,
          "description": "[D] 用户自定义标签"
        },
        "is_starred": {
          "type": "boolean",
          "default": false,
          "description": "[D] 用户标记为重要"
        }
      }
    },

    "lifecycle": {
      "type": "object",
      "description": "生命周期状态跟踪",
      "required": ["status", "created_at"],
      "additionalProperties": false,
      "properties": {
        "status": {
          "type": "string",
          "enum": ["sealing", "sealed", "seal_failed", "exported", "wiped"],
          "description": "[D] 证据当前状态"
        },
        "created_at": {
          "type": "string",
          "format": "date-time",
          "description": "[H][D] 记录创建时间"
        },
        "last_verified_at": {
          "type": ["string", "null"],
          "format": "date-time",
          "description": "[D] 最近一次完整性校验时间"
        },
        "last_exported_at": {
          "type": ["string", "null"],
          "format": "date-time",
          "description": "[D] 最近一次导出时间"
        },
        "verify_failure_count": {
          "type": "integer",
          "default": 0,
          "description": "[D] 完整性校验失败次数（非零即告警）"
        }
      }
    }
  }
}
```

---

## 4. 参与哈希计算的字段清单

以下字段参与 `integrity.record_hash_hex` 计算，一经写入不可修改：

```
schema_version

identity.evidence_id
identity.case_id
identity.sequence_no

classification.media_type
classification.evidence_category

file_ref.original_filename
file_ref.mime_type
file_ref.file_size_bytes
file_ref.hash_algorithm
file_ref.file_hash_hex

capture_context.device_model
capture_context.os_version
capture_context.app_version
capture_context.ntp_time_utc
capture_context.local_clock_utc
capture_context.clock_drift_ms
capture_context.timezone
capture_context.locale
capture_context.gps.lat            (if not null)
capture_context.gps.lng            (if not null)
capture_context.gps.accuracy_m     (if not null)
capture_context.network_type
capture_context.battery_level

encryption.algorithm

timestamp_proof.status
timestamp_proof.tsa_url            (if granted)
timestamp_proof.granted_time_utc   (if granted)
timestamp_proof.nonce              (if granted)
timestamp_proof.request_hash_hex   (if granted)

lifecycle.created_at
```

**哈希计算规则**：
```
1. 提取上述字段，按路径字母序排列
2. 构建有序 JSON 对象（键排序，无空格，无换行）
3. UTF-8 编码 → SHA-256 → 十六进制小写
```

**不参与哈希的字段**（允许后续修改）：
```
classification.evidence_category_label   (派生展示字段)
file_ref.encrypted_storage_path          (本地路径可变)
encryption.key_ref                       (密钥轮换时可变)
encryption.iv_hex                        (重加密时可变)
encryption.auth_tag_hex                  (重加密时可变)
timestamp_proof.tsr_base64               (二进制体，由 TSA 签名保护)
timestamp_proof.tsa_cert_chain_pem       (证书可更新)
timestamp_proof.tsa_serial_number        (由 TSR 内部包含)
mutable.*                                (设计即为可变)
lifecycle.status                         (状态流转)
lifecycle.last_verified_at               (动态更新)
lifecycle.last_exported_at               (动态更新)
lifecycle.verify_failure_count           (动态更新)
integrity.*                              (自身不参与自身哈希)
```

---

## 5. 字段访问分级矩阵

| 字段路径 | UI 展示 [D] | 司法校验 [V] | 参与哈希 [H] |
|---------|:-----------:|:-----------:|:-----------:|
| `identity.evidence_id` | ✓ | ✓ | ✓ |
| `identity.case_id` | ✓ | ✓ | ✓ |
| `identity.sequence_no` | ✓ | ✓ | ✓ |
| `classification.media_type` | ✓ | ✓ | ✓ |
| `classification.evidence_category` | ✓ | ✓ | ✓ |
| `classification.evidence_category_label` | ✓ | — | — |
| `file_ref.original_filename` | ✓ | ✓ | ✓ |
| `file_ref.mime_type` | ✓ | ✓ | ✓ |
| `file_ref.file_size_bytes` | ✓ | ✓ | ✓ |
| `file_ref.hash_algorithm` | — | ✓ | ✓ |
| `file_ref.file_hash_hex` | — | ✓ | ✓ |
| `file_ref.encrypted_storage_path` | — | ✓ | — |
| `capture_context.ntp_time_utc` | ✓ | ✓ | ✓ |
| `capture_context.local_clock_utc` | — | ✓ | ✓ |
| `capture_context.clock_drift_ms` | — | ✓ | ✓ |
| `capture_context.timezone` | ✓ | ✓ | ✓ |
| `capture_context.device_model` | — | ✓ | ✓ |
| `capture_context.os_version` | — | ✓ | ✓ |
| `capture_context.app_version` | — | ✓ | ✓ |
| `capture_context.locale` | — | ✓ | ✓ |
| `capture_context.gps.lat` | ✓ | ✓ | ✓ |
| `capture_context.gps.lng` | ✓ | ✓ | ✓ |
| `capture_context.gps.accuracy_m` | — | ✓ | ✓ |
| `capture_context.network_type` | — | ✓ | ✓ |
| `capture_context.battery_level` | — | ✓ | ✓ |
| `encryption.algorithm` | — | ✓ | ✓ |
| `encryption.key_ref` | — | ✓ | — |
| `encryption.iv_hex` | — | ✓ | — |
| `encryption.auth_tag_hex` | — | ✓ | — |
| `timestamp_proof.status` | ✓ | ✓ | ✓ |
| `timestamp_proof.granted_time_utc` | ✓ | ✓ | ✓ |
| `timestamp_proof.tsa_url` | — | ✓ | ✓ |
| `timestamp_proof.tsr_base64` | — | ✓ | — |
| `timestamp_proof.tsa_cert_chain_pem` | — | ✓ | — |
| `timestamp_proof.tsa_serial_number` | — | ✓ | — |
| `timestamp_proof.nonce` | — | ✓ | ✓ |
| `timestamp_proof.request_hash_hex` | — | ✓ | ✓ |
| `integrity.record_hash_hex` | — | ✓ | — |
| `integrity.hash_input_fields` | — | ✓ | — |
| `mutable.user_note` | ✓ | — | — |
| `mutable.tags` | ✓ | — | — |
| `mutable.is_starred` | ✓ | — | — |
| `lifecycle.status` | ✓ | — | — |
| `lifecycle.created_at` | ✓ | ✓ | ✓ |
| `lifecycle.last_verified_at` | ✓ | — | — |
| `lifecycle.last_exported_at` | ✓ | — | — |
| `lifecycle.verify_failure_count` | ✓ | — | — |

---

## 6. evidence_category 枚举与司法场景映射

| enum 值 | 中文标签 | 典型劳动争议场景 | 建议搭配媒体类型 |
|---------|---------|-----------------|----------------|
| `overtime` | 加班记录 | 加班费争议 | screenshot, photo |
| `verbal_instruction` | 口头指令 | 领导违法指令、口头承诺 | audio, video |
| `written_instruction` | 书面指令 | 邮件/消息中的工作安排 | screenshot, document |
| `salary_record` | 工资/报酬记录 | 欠薪、克扣工资 | screenshot, document |
| `attendance` | 考勤记录 | 旷工争议、出勤证明 | screenshot, photo |
| `reward_or_praise` | 嘉奖/表扬 | 证明工作表现（反驳不胜任） | screenshot, photo, document |
| `harassment` | 骚扰/歧视 | 职场骚扰、歧视 | audio, video, screenshot |
| `termination_notice` | 解雇/终止通知 | 违法解雇 | photo, document, audio |
| `contract` | 劳动合同 | 合同条款争议 | photo, document |
| `communication` | 沟通记录 | 综合沟通证据 | screenshot, audio |
| `other` | 其他 | 未归类证据 | 任意 |

---

## 7. 完整性校验伪代码（第三方可独立执行）

```python
import json, hashlib

def verify_evidence_item(item: dict) -> bool:
    """任何人拿到 EvidenceItem JSON 即可执行此验证"""

    # 1. 从 integrity.hash_input_fields 获取字段路径列表
    field_paths = item["integrity"]["hash_input_fields"]

    # 2. 按路径提取值，构建有序字典
    hash_input = {}
    for path in sorted(field_paths):
        value = resolve_path(item, path)  # 按 . 分隔逐层取值
        if value is not None:
            hash_input[path] = value

    # 3. 确定性 JSON 序列化
    canonical = json.dumps(hash_input, sort_keys=True,
                           ensure_ascii=False, separators=(',', ':'))

    # 4. SHA-256
    computed = hashlib.sha256(canonical.encode('utf-8')).hexdigest()

    # 5. 比对
    return computed == item["integrity"]["record_hash_hex"]
```

---

## 8. 设计决策记录

| # | 决策 | 理由 |
|---|------|------|
| D1 | `encryption.key_ref/iv_hex/auth_tag_hex` 不参与哈希 | 密钥轮换或重加密时这些值会变，但原始文件内容不变 |
| D2 | `tsr_base64` 不参与哈希 | TSR 自身由 TSA 私钥签名保护，其完整性由 `openssl ts -verify` 独立验证 |
| D3 | `evidence_category_label` 不参与哈希 | 纯展示用派生字段，国际化时可能变化 |
| D4 | `mutable.*` 全部不参与哈希 | 用户需要能修改备注和标签，不应破坏证据完整性 |
| D5 | `hash_input_fields` 写入记录本身 | 自描述设计——第三方无需阅读源码即可知道哪些字段参与了哈希 |
| D6 | GPS 为 null 时不参与哈希输入 | 避免 null 值的序列化歧义 |
| D7 | `timestamp_proof` 条件性参与哈希 | pending/failed 状态下 TSA 字段不存在，只有 granted 时才纳入 |
