# EvidenceVault v1.0 — 证据导出包规格

> 证据导出包是 EvidenceVault 产出的"最终交付物"——
> 它直接交给劳动仲裁员、法官、对方律师。
> 包内的一切必须自包含、自描述、自验证，不依赖 App 即可完成全部校验。

---

## 1. 设计约束

| 约束 | 要求 |
|------|------|
| 自包含 | 脱离 App、脱离网络（除 TSA 验证外）也能完成核心校验 |
| 受众分层 | 法官/仲裁员读 PDF；技术鉴定人员读 JSON + 命令行验证 |
| 格式通用 | 标准 ZIP，任何操作系统可解压 |
| 文件原始 | 证据文件必须是解密后的原始格式，不可做任何转码 |
| 完整性闭环 | 包内自带所有验证所需材料（证书、脚本、指南） |

---

## 2. 目录结构

```
EvidencePackage_{case_id_short}_{export_date}/
│
├── README.txt                              # 人类可读的包说明（纯文本，兼容所有系统）
│
├── 证据报告.pdf                             # 给法官/仲裁员的完整报告（中文）
│
├── evidence/                               # 原始证据文件（解密后原件）
│   ├── E001_photo_20260101_143205.jpg
│   ├── E002_audio_20260115_091030.m4a
│   ├── E003_screenshot_20260120_183000.png
│   └── E004_document_20260201_100000.pdf
│
├── metadata/                               # 每条证据的完整机器可读元数据
│   ├── E001_photo_20260101_143205.json
│   ├── E002_audio_20260115_091030.json
│   ├── E003_screenshot_20260120_183000.json
│   └── E004_document_20260201_100000.json
│
├── proofs/                                 # 哈希存证证明材料
│   ├── tsa_responses/                      # TSA 时间戳响应原始文件
│   │   ├── E001.tsr
│   │   ├── E002.tsr
│   │   ├── E003.tsr
│   │   └── E004.tsr
│   ├── tsa_certs/                          # TSA 证书链
│   │   └── tsa_ca_chain.pem
│   └── chain_certificates/                 # 司法链证书（v1.1，v1.0 为空目录）
│
├── audit/                                  # 操作审计链
│   └── chain_of_custody.json               # 全部操作日志（哈希链）
│
├── manifest.json                           # 包清单（总索引）
│
├── verify/                                 # 验证工具包
│   ├── verification_guide.html             # 可视化验证指南（浏览器打开）
│   ├── verify.sh                           # Linux/Mac 一键验证脚本
│   ├── verify.bat                          # Windows 一键验证脚本
│   └── checksums.sha256                    # 标准 sha256sum 格式校验文件
│
└── package_signature.json                  # 包级完整性签名
```

**命名规则**：
```
证据文件: E{序号3位}_{media_type}_{YYYYMMDD}_{HHmmss}.{ext}
元数据:   与证据文件同名，扩展名改为 .json
TSA:      E{序号3位}.tsr
序号:     按采集时间升序排列
```

---

## 3. 各文件职责与格式

### 3.1 职责划分矩阵

```
           ┌──────────────────────────────────────────────────┐
           │                    受众                           │
           ├────────────────────┬─────────────────────────────┤
           │  法官 / 仲裁员      │  技术鉴定人员 / 对方律师      │
           │  (非技术)          │  (技术)                      │
     ┌─────┼────────────────────┼─────────────────────────────┤
     │ 概览 │ README.txt         │ manifest.json               │
  文 │     │ 证据报告.pdf        │                             │
  件 ├─────┼────────────────────┼─────────────────────────────┤
  类 │ 证据 │ 证据报告.pdf 中     │ evidence/ 原始文件          │
  型 │ 内容 │ 的截图 + 描述       │ metadata/*.json             │
     ├─────┼────────────────────┼─────────────────────────────┤
     │ 验证 │ 证据报告.pdf 中     │ proofs/tsa_responses/*.tsr  │
     │ 证明 │ 的验证结论表格      │ proofs/tsa_certs/*.pem      │
     │     │                    │ verify/verify.sh             │
     ├─────┼────────────────────┼─────────────────────────────┤
     │ 操作 │ 证据报告.pdf 中     │ audit/chain_of_custody.json │
     │ 记录 │ 的操作时间线        │                             │
     └─────┴────────────────────┴─────────────────────────────┘
```

**核心原则**：
- **PDF 是 JSON 的人类可读投影**——PDF 中的每一项数据都有 JSON 作为机器可验证源
- **PDF 不是权威**——如果 PDF 与 JSON 不一致，以 JSON + 原始文件为准
- **PDF 存在的目的**——让不懂技术的仲裁员能在 5 分钟内理解证据链完整性

---

### 3.2 README.txt

```
================================================================
       证据包说明 — EvidenceVault 职场证据固化工具
================================================================

案件编号:   CASE-20260208-a3f7
导出时间:   2026-02-08 16:00:00 CST (UTC+8)
证据数量:   4 条
导出设备:   iPhone 15 Pro / iOS 18.2
App 版本:   EvidenceVault v1.0.3 (build 42)

本证据包由 EvidenceVault App 自动生成。
包内所有证据文件均为采集时的原始格式，未经任何编辑或转码。

── 包内文件说明 ──

  证据报告.pdf         → 阅读此文件了解全部证据概要和验证结论
  evidence/            → 原始证据文件（照片、录音等）
  metadata/            → 每条证据的采集参数（JSON 格式）
  proofs/              → 第三方时间戳证明（可独立验证）
  audit/               → 操作审计日志
  manifest.json        → 机器可读的包总索引
  verify/              → 验证工具和指南

── 如何验证 ──

  非技术人员: 请打开「证据报告.pdf」查看验证结论
  技术人员:   请打开 verify/verification_guide.html
             或执行 verify/verify.sh（Linux/Mac）
             或执行 verify/verify.bat（Windows）

── 法律声明 ──

  本证据包中的时间戳由联合信任时间戳服务签发，
  符合 RFC 3161 国际标准和《中华人民共和国电子签名法》相关规定。
  时间戳证明可通过 OpenSSL 等开源工具独立验证。

================================================================
```

### 3.3 manifest.json（包级总索引）

```json
{
  "manifest_version": "1.0.0",
  "app_version": "1.0.3",
  "export_id": "uuid",
  "export_time_utc": "2026-02-08T08:00:00.000Z",
  "export_time_local": "2026-02-08T16:00:00.000+08:00",

  "case": {
    "case_id": "uuid",
    "case_id_short": "CASE-20260208-a3f7",
    "case_title": "用户自定义案件名称",
    "created_at": "2026-01-01T00:00:00.000Z"
  },

  "evidence_summary": {
    "total_count": 4,
    "by_media_type": { "photo": 1, "audio": 1, "screenshot": 1, "document": 1 },
    "by_category": { "salary_record": 2, "overtime": 1, "communication": 1 },
    "earliest_capture": "2026-01-01T06:32:05.000Z",
    "latest_capture": "2026-02-01T02:00:00.000Z",
    "total_file_size_bytes": 15728640
  },

  "evidence_items": [
    {
      "sequence": "E001",
      "evidence_id": "uuid",
      "media_type": "photo",
      "evidence_category": "salary_record",
      "evidence_category_label": "工资/报酬记录",
      "capture_time_utc": "2026-01-01T06:32:05.000Z",
      "capture_time_local": "2026-01-01T14:32:05.000+08:00",

      "files": {
        "evidence": "evidence/E001_photo_20260101_143205.jpg",
        "metadata": "metadata/E001_photo_20260101_143205.json",
        "tsa_response": "proofs/tsa_responses/E001.tsr"
      },

      "integrity": {
        "file_hash_sha256": "a3f7b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f021",
        "file_size_bytes": 3145728,
        "record_hash_sha256": "1234abcd...",
        "tsa_granted_time_utc": "2026-01-01T06:32:08.000Z",
        "tsa_status": "granted"
      }
    },
    {
      "sequence": "E002",
      "...": "..."
    }
  ],

  "audit_trail": {
    "file": "audit/chain_of_custody.json",
    "total_entries": 24,
    "chain_integrity": "verified",
    "first_entry_hash": "genesis...",
    "last_entry_hash": "final..."
  },

  "verification": {
    "guide_html": "verify/verification_guide.html",
    "script_unix": "verify/verify.sh",
    "script_windows": "verify/verify.bat",
    "checksums_file": "verify/checksums.sha256"
  },

  "package_hash_sha256": "整个包（除本字段和 package_signature.json 外）的哈希"
}
```

### 3.4 metadata/E001_xxx.json（单条证据元数据）

```json
{
  "schema_version": "1.0.0",
  "evidence_id": "uuid",
  "sequence": "E001",

  "classification": {
    "media_type": "photo",
    "evidence_category": "salary_record",
    "evidence_category_label": "工资/报酬记录"
  },

  "file_ref": {
    "original_filename": "E001_photo_20260101_143205.jpg",
    "mime_type": "image/jpeg",
    "file_size_bytes": 3145728,
    "hash_algorithm": "SHA-256",
    "file_hash_hex": "a3f7b2c1..."
  },

  "capture_context": {
    "ntp_time_utc": "2026-01-01T06:32:05.000Z",
    "local_clock_utc": "2026-01-01T06:32:05.120Z",
    "clock_drift_ms": 120,
    "timezone": "Asia/Shanghai",
    "device_model": "iPhone 15 Pro",
    "os_version": "iOS 18.2",
    "app_version": "1.0.3",
    "locale": "zh-CN",
    "gps": {
      "lat": 31.2304,
      "lng": 121.4737,
      "accuracy_m": 8.0
    },
    "network_type": "wifi",
    "battery_level": 72
  },

  "hash_proof": {
    "proof_id": "uuid",
    "channel": "tsa_rfc3161",
    "status": "granted",
    "input": {
      "hash_algorithm": "SHA-256",
      "hash_hex": "a3f7b2c1...",
      "nonce": "random-nonce-string"
    },
    "tsa_url": "https://tsa.tsa.cn/",
    "granted_time_utc": "2026-01-01T06:32:08.000Z",
    "tsa_serial_number": "1234567890",
    "tsa_response_file": "../proofs/tsa_responses/E001.tsr",
    "tsa_cert_file": "../proofs/tsa_certs/tsa_ca_chain.pem"
  },

  "integrity": {
    "record_hash_hex": "1234abcd...",
    "hash_input_fields": [
      "schema_version",
      "evidence_id",
      "classification.media_type",
      "classification.evidence_category",
      "file_ref.original_filename",
      "file_ref.mime_type",
      "file_ref.file_size_bytes",
      "file_ref.hash_algorithm",
      "file_ref.file_hash_hex",
      "capture_context.ntp_time_utc",
      "..."
    ]
  },

  "user_note": "2025年12月工资条截图，显示加班费为0"
}
```

### 3.5 audit/chain_of_custody.json（操作审计链）

```json
{
  "chain_version": "1.0.0",
  "case_id": "uuid",
  "total_entries": 24,

  "entries": [
    {
      "log_id": "uuid",
      "sequence_no": 1,
      "timestamp_utc": "2026-01-01T06:32:05.000Z",
      "action": "CAPTURE_START",
      "evidence_id": "uuid (E001)",
      "detail": "Photo capture initiated",
      "prev_log_hash": "GENESIS",
      "this_log_hash": "hash_of_entry_1"
    },
    {
      "log_id": "uuid",
      "sequence_no": 2,
      "timestamp_utc": "2026-01-01T06:32:06.000Z",
      "action": "HASH_COMPUTED",
      "evidence_id": "uuid (E001)",
      "detail": "SHA-256: a3f7b2c1...(前16位)",
      "prev_log_hash": "hash_of_entry_1",
      "this_log_hash": "hash_of_entry_2"
    },
    {
      "log_id": "uuid",
      "sequence_no": 3,
      "timestamp_utc": "2026-01-01T06:32:06.100Z",
      "action": "FILE_ENCRYPTED",
      "evidence_id": "uuid (E001)",
      "detail": "AES-256-GCM, encrypted and stored",
      "prev_log_hash": "hash_of_entry_2",
      "this_log_hash": "hash_of_entry_3"
    },
    {
      "log_id": "uuid",
      "sequence_no": 4,
      "timestamp_utc": "2026-01-01T06:32:08.000Z",
      "action": "TSA_GRANTED",
      "evidence_id": "uuid (E001)",
      "detail": "TSA granted at 2026-01-01T06:32:08Z, serial: 1234567890",
      "prev_log_hash": "hash_of_entry_3",
      "this_log_hash": "hash_of_entry_4"
    },
    {
      "sequence_no": "...",
      "action": "EVIDENCE_SEAL_COMPLETE | EXPORT_START | EXPORT_COMPLETE | ...",
      "...": "..."
    }
  ],

  "chain_verification": {
    "method": "对每条 entry（除 this_log_hash 外）JSON 序列化后 SHA-256",
    "genesis_hash": "GENESIS",
    "final_hash": "hash_of_last_entry",
    "all_consecutive": true
  }
}
```

### 3.6 verify/checksums.sha256

```
# EvidenceVault Export Package — SHA-256 Checksums
# Generated: 2026-02-08T16:00:00+08:00
# Verify: sha256sum -c checksums.sha256

a3f7b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f021  evidence/E001_photo_20260101_143205.jpg
b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2  evidence/E002_audio_20260115_091030.m4a
c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3  evidence/E003_screenshot_20260120_183000.png
d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4  evidence/E004_document_20260201_100000.pdf
e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5  metadata/E001_photo_20260101_143205.json
f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6  metadata/E002_audio_20260115_091030.json
...
```

### 3.7 verify/verify.sh（一键验证脚本）

```bash
#!/bin/bash
# EvidenceVault 证据包验证脚本
# 运行环境: Linux / macOS (需安装 openssl)
# 用法: chmod +x verify.sh && ./verify.sh

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PACKAGE_DIR="$(dirname "$SCRIPT_DIR")"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m'

PASS=0
FAIL=0
WARN=0

check() {
    local name="$1" result="$2" detail="$3"
    if [ "$result" = "PASS" ]; then
        echo -e "  ${GREEN}✓ PASS${NC}  $name"
        [ -n "$detail" ] && echo "         $detail"
        PASS=$((PASS + 1))
    elif [ "$result" = "WARN" ]; then
        echo -e "  ${YELLOW}⚠ WARN${NC}  $name"
        [ -n "$detail" ] && echo "         $detail"
        WARN=$((WARN + 1))
    else
        echo -e "  ${RED}✗ FAIL${NC}  $name"
        [ -n "$detail" ] && echo "         $detail"
        FAIL=$((FAIL + 1))
    fi
}

echo ""
echo "════════════════════════════════════════════════════════"
echo "  EvidenceVault 证据包完整性验证"
echo "  包路径: $PACKAGE_DIR"
echo "  验证时间: $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
echo "════════════════════════════════════════════════════════"
echo ""

# ── 阶段 1: 文件完整性 ──
echo "【阶段 1/4】文件哈希校验"
echo "─────────────────────────"

if command -v sha256sum &> /dev/null; then
    SHA_CMD="sha256sum"
elif command -v shasum &> /dev/null; then
    SHA_CMD="shasum -a 256"
else
    check "SHA-256 工具" "FAIL" "未找到 sha256sum 或 shasum"
    exit 1
fi

cd "$PACKAGE_DIR"
while IFS= read -r line; do
    [[ "$line" =~ ^#.*$ ]] && continue
    [[ -z "$line" ]] && continue

    expected_hash=$(echo "$line" | awk '{print $1}')
    file_path=$(echo "$line" | awk '{print $2}')

    if [ ! -f "$file_path" ]; then
        check "文件存在: $file_path" "FAIL" "文件不存在"
        continue
    fi

    actual_hash=$($SHA_CMD "$file_path" | awk '{print $1}')

    if [ "$expected_hash" = "$actual_hash" ]; then
        check "哈希校验: $file_path" "PASS" ""
    else
        check "哈希校验: $file_path" "FAIL" "期望: ${expected_hash:0:16}... 实际: ${actual_hash:0:16}..."
    fi
done < verify/checksums.sha256

echo ""

# ── 阶段 2: TSA 时间戳验证 ──
echo "【阶段 2/4】TSA 时间戳验证"
echo "──────────────────────────"

TSA_CA="proofs/tsa_certs/tsa_ca_chain.pem"

if [ ! -f "$TSA_CA" ]; then
    check "TSA CA 证书" "FAIL" "证书文件不存在: $TSA_CA"
else
    for tsr_file in proofs/tsa_responses/*.tsr; do
        [ -f "$tsr_file" ] || continue
        base=$(basename "$tsr_file" .tsr)
        evidence_file=$(ls evidence/${base}_* 2>/dev/null | head -1)

        if [ -z "$evidence_file" ]; then
            check "TSA 验证: $base" "WARN" "未找到对应证据文件"
            continue
        fi

        result=$(openssl ts -verify -in "$tsr_file" -data "$evidence_file" -CAfile "$TSA_CA" 2>&1)

        if echo "$result" | grep -q "Verification: OK"; then
            ts_time=$(openssl ts -reply -in "$tsr_file" -text 2>/dev/null | grep "Time stamp:" | head -1)
            check "TSA 验证: $base" "PASS" "$ts_time"
        else
            check "TSA 验证: $base" "FAIL" "$result"
        fi
    done
fi

echo ""

# ── 阶段 3: 元数据一致性 ──
echo "【阶段 3/4】元数据一致性校验"
echo "────────────────────────────"

for meta_file in metadata/*.json; do
    [ -f "$meta_file" ] || continue
    base=$(basename "$meta_file" .json)
    evidence_file=$(ls evidence/${base}.* 2>/dev/null | head -1)

    if [ -z "$evidence_file" ]; then
        check "元数据对应: $base" "WARN" "未找到对应证据文件"
        continue
    fi

    recorded_hash=$(python3 -c "
import json, sys
with open('$meta_file') as f:
    d = json.load(f)
print(d.get('file_ref',{}).get('file_hash_hex',''))
" 2>/dev/null || echo "")

    if [ -z "$recorded_hash" ]; then
        check "元数据解析: $base" "WARN" "无法解析元数据（需要 python3）"
        continue
    fi

    actual_hash=$($SHA_CMD "$evidence_file" | awk '{print $1}')

    if [ "$recorded_hash" = "$actual_hash" ]; then
        check "元数据哈希一致: $base" "PASS" ""
    else
        check "元数据哈希一致: $base" "FAIL" "元数据记录与实际文件哈希不匹配"
    fi
done

echo ""

# ── 阶段 4: 审计链完整性 ──
echo "【阶段 4/4】审计链完整性校验"
echo "────────────────────────────"

AUDIT_FILE="audit/chain_of_custody.json"

if [ ! -f "$AUDIT_FILE" ]; then
    check "审计日志存在" "FAIL" "文件不存在"
else
    chain_ok=$(python3 -c "
import json, hashlib

with open('$AUDIT_FILE') as f:
    data = json.load(f)

entries = data['entries']
prev = 'GENESIS'
broken = False

for e in entries:
    if e['prev_log_hash'] != prev:
        print(f'BROKEN at seq {e[\"sequence_no\"]}')
        broken = True
        break
    verify_obj = {k: v for k, v in e.items() if k != 'this_log_hash'}
    canonical = json.dumps(verify_obj, sort_keys=True, ensure_ascii=False, separators=(',',':'))
    computed = hashlib.sha256(canonical.encode('utf-8')).hexdigest()
    if computed != e['this_log_hash']:
        print(f'HASH_MISMATCH at seq {e[\"sequence_no\"]}')
        broken = True
        break
    prev = e['this_log_hash']

if not broken:
    print(f'OK:{len(entries)}')
" 2>/dev/null || echo "ERROR")

    if [[ "$chain_ok" == OK:* ]]; then
        count="${chain_ok#OK:}"
        check "审计链连续性" "PASS" "$count 条记录，链式哈希完整"
    else
        check "审计链连续性" "FAIL" "$chain_ok"
    fi
fi

# ── 总结 ──
echo ""
echo "════════════════════════════════════════════════════════"
echo "  验证完成"
echo "  ✓ 通过: $PASS"
echo "  ✗ 失败: $FAIL"
echo "  ⚠ 警告: $WARN"
echo ""
if [ "$FAIL" -eq 0 ]; then
    echo -e "  ${GREEN}结论: 全部校验通过，证据包完整性验证成功${NC}"
else
    echo -e "  ${RED}结论: 存在 $FAIL 项校验失败，证据完整性存疑${NC}"
fi
echo "════════════════════════════════════════════════════════"
```

---

## 4. PDF 报告结构设计

### 4.1 页面编排

```
┌──────────────────────────────────────────────────────────────┐
│ 第 1 页: 封面                                                │
│                                                              │
│              ┌────────────────────────┐                      │
│              │    ⬡ EvidenceVault     │                      │
│              │    电子证据固化报告      │                      │
│              │                        │                      │
│              │ 案件: CASE-20260208... │                      │
│              │ 导出: 2026年2月8日      │                      │
│              │ 证据: 4 份              │                      │
│              │                        │                      │
│              │ ⚠ 本报告由软件自动生成  │                      │
│              │   如与原始数据不一致     │                      │
│              │   以 JSON 文件为准      │                      │
│              └────────────────────────┘                      │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ 第 2 页: 证据总览                                            │
│                                                              │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ 案件信息                                                 │ │
│ │  案件名称:  2025年12月-2026年1月欠薪争议                  │ │
│ │  创建时间:  2026-01-01                                   │ │
│ │  证据数量:  4 份（照片 1, 录音 1, 截图 1, 文档 1）        │ │
│ │  时间跨度:  2026-01-01 至 2026-02-01                     │ │
│ │  存证方式:  RFC 3161 可信时间戳（联合信任 TSA）            │ │
│ ├──────────────────────────────────────────────────────────┤ │
│ │ 证据清单                                                 │ │
│ │  序号  类型    分类        采集时间              状态     │ │
│ │  E001  照片    工资记录    2026-01-01 14:32:05  已认证 ✓ │ │
│ │  E002  录音    加班记录    2026-01-15 09:10:30  已认证 ✓ │ │
│ │  E003  截图    沟通记录    2026-01-20 18:30:00  已认证 ✓ │ │
│ │  E004  文档    工资记录    2026-02-01 10:00:00  已认证 ✓ │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ 第 3-6 页: 逐条证据详情（每条 1 页）                          │
│                                                              │
│ ┌─────────────────────────────────────────────────────────┐  │
│ │ E001 — 照片 — 工资/报酬记录                              │  │
│ │                                                         │  │
│ │ ┌───────────────────┐  基本信息                         │  │
│ │ │                   │   文件: E001_photo_20260101...jpg │  │
│ │ │   [照片缩略图]     │   大小: 3.0 MB                   │  │
│ │ │   (宽度不超过50%)  │   格式: JPEG                     │  │
│ │ │                   │                                   │  │
│ │ └───────────────────┘  采集环境                         │  │
│ │                         时间: 2026-01-01 14:32:05 CST  │  │
│ │                         地点: 31.23°N, 121.47°E ±8m   │  │
│ │                         设备: iPhone 15 Pro / iOS 18.2 │  │
│ │                                                         │  │
│ │ 完整性验证                                               │  │
│ │ ┌─────────────────────────────────────────────────────┐ │  │
│ │ │ 校验项             │ 结果  │ 详情                    │ │  │
│ │ ├────────────────────┼───────┼─────────────────────────┤ │  │
│ │ │ 文件哈希 (SHA-256) │  ✓   │ a3f7b2c1...c921        │ │  │
│ │ │ TSA 时间戳签名     │  ✓   │ 签发: 2026-01-01       │ │  │
│ │ │                    │      │ 14:32:08 UTC           │ │  │
│ │ │ TSA 证书链有效性   │  ✓   │ 联合信任 TSA CA        │ │  │
│ │ │ 哈希值一致性       │  ✓   │ TSA哈希=文件哈希       │ │  │
│ │ └─────────────────────────────────────────────────────┘ │  │
│ │                                                         │  │
│ │ 用户备注                                                 │  │
│ │ "2025年12月工资条截图，显示加班费为0"                       │  │
│ └─────────────────────────────────────────────────────────┘  │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ 第 7 页: 操作时间线                                          │
│                                                              │
│  2026-01-01 14:32:05  ● E001 照片采集                       │
│  2026-01-01 14:32:06  │ E001 SHA-256 哈希计算                │
│  2026-01-01 14:32:06  │ E001 AES-256-GCM 加密存储           │
│  2026-01-01 14:32:08  │ E001 TSA 时间戳签发                  │
│  2026-01-01 14:32:08  ● E001 固化完成                       │
│           ...          ...                                   │
│  2026-02-08 16:00:00  ● 证据包导出                           │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ 第 8 页: 验证方法说明                                        │
│                                                              │
│  本节面向法官/仲裁员，解释证据完整性证明的技术原理:            │
│                                                              │
│  1. 什么是 SHA-256 哈希                                      │
│     "数字指纹"——文件内容的任何修改（哪怕一个像素/一个字节）    │
│     都会导致哈希值完全改变。就像指纹可以唯一标识一个人，        │
│     哈希值可以唯一标识一个文件。                               │
│                                                              │
│  2. 什么是可信时间戳                                          │
│     由国家认可的第三方时间戳机构签发的时间证明。                │
│     证明"某个文件的指纹在某个时间点之前已经存在"。              │
│     类似于"在公证处给信封盖上日期戳"。                         │
│                                                              │
│  3. 如何理解本报告的验证结论                                  │
│     如果某条证据的全部 4 项校验均为 ✓:                        │
│     → 该证据文件自采集以来未被修改过                           │
│     → 该证据在时间戳显示的时间之前已经存在                      │
│     → 采集设备和环境信息真实记录于采集瞬间                      │
│                                                              │
│  4. 如果您希望亲自验证                                        │
│     请参阅 verify/ 目录下的验证指南和脚本                      │
│     或委托技术鉴定机构对本证据包进行独立验证                    │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ 第 9 页: 法律声明与技术声明                                   │
│                                                              │
│  - 时间戳服务: 联合信任时间戳服务                              │
│    资质: 国家密码管理局审批 / CMA 认证 / CNAS 认证             │
│  - 哈希算法: SHA-256（NIST FIPS 180-4 标准）                  │
│  - 加密算法: AES-256-GCM（NIST SP 800-38D 标准）             │
│  - 本报告由 EvidenceVault v1.0.3 自动生成                     │
│  - 如本报告内容与 JSON 原始数据不一致，以 JSON 数据为准         │
│  - 本工具不构成法律建议，建议配合专业律师使用                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 PDF 生成技术方案

```
Flutter 中生成 PDF:
  库: pdf (dart package) — 纯 Dart PDF 生成
  中文字体: 内置 NotoSansSC-Regular.ttf 子集（仅包含常用字符）
  图片缩略图: 证据图片缩放到 800px 宽度嵌入 PDF
  录音/视频: 显示波形图缩略图或首帧截图

PDF 不内嵌完整原始文件（太大）——原始文件在 evidence/ 目录中
```

---

## 5. 完整验真逻辑（10 步）

```
验真流程适用人员: 司法鉴定人员、技术型律师、法官助理
前提: 拿到 ZIP 包并解压

Step 1: 包完整性检查
━━━━━━━━━━━━━━━━━━
  读取 package_signature.json
  计算包内所有文件（除 package_signature.json 自身）的联合哈希
  比对是否与 package_signature.package_hash_sha256 一致
  → 通过 = 包未被篡改

Step 2: 文件级哈希校验
━━━━━━━━━━━━━━━━━━━━
  执行 sha256sum -c verify/checksums.sha256
  → 每个文件的实际哈希 = checksums 中记录的哈希
  → 通过 = 包内所有文件均未被单独篡改

Step 3: 逐条证据 — 文件哈希一致性
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  对每个 evidence/E00x_* 文件:
    actual_hash = sha256sum(evidence/E00x_*.*)
    recorded_hash = metadata/E00x_*.json → file_ref.file_hash_hex
    manifest_hash = manifest.json → evidence_items[x].integrity.file_hash_sha256
  → 三者必须完全一致

Step 4: 逐条证据 — TSA 签名验证
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  对每个证据:
    openssl ts -verify \
      -in proofs/tsa_responses/E00x.tsr \
      -data evidence/E00x_*.* \
      -CAfile proofs/tsa_certs/tsa_ca_chain.pem
  → "Verification: OK" = TSA 签名有效

Step 5: 逐条证据 — TSA 哈希绑定验证
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  从 TSR 中提取 messageImprint.hashedMessage:
    openssl ts -reply -in E00x.tsr -text | grep "Message data"
  → TSR 内的哈希 = 文件实际哈希 = 元数据记录哈希
  → 证明: "TSA 认证的就是这个文件，不是别的文件"

Step 6: 逐条证据 — TSA 时间提取
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  openssl ts -reply -in E00x.tsr -text | grep "Time stamp:"
  → 提取 TSA 签发的可信时间
  → 比对 metadata/E00x.json 中的 granted_time_utc 是否一致
  → 证明: "该文件在此时间之前已存在"

Step 7: TSA 证书链验证
━━━━━━━━━━━━━━━━━━━━━
  openssl verify -CAfile system_root_ca.pem proofs/tsa_certs/tsa_ca_chain.pem
  → 确认 TSA 证书由受信根 CA 签发
  → 确认 TSA 证书未过期、未被吊销

Step 8: 元数据 record_hash 验证
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  对每个 metadata/E00x.json:
    1. 读取 integrity.hash_input_fields 列表
    2. 按列表提取字段值
    3. 按键排序 JSON 序列化 (sort_keys, separators=(',',':'))
    4. SHA-256 计算
  → 计算结果 = integrity.record_hash_hex
  → 证明: "元数据本身未被篡改"

Step 9: 审计链完整性验证
━━━━━━━━━━━━━━━━━━━━━━
  读取 audit/chain_of_custody.json
  从第 1 条开始逐条验证:
    - prev_log_hash 是否等于上一条的 this_log_hash
    - 重算 this_log_hash（对 entry 中除 this_log_hash 外的全字段 JSON 序列化后 SHA-256）
    - sequence_no 是否严格连续
  → 全链无断裂 = 操作记录未被篡改

Step 10: 交叉验证汇总
━━━━━━━━━━━━━━━━━━━━
  检查以下等式链是否成立:

  sha256(原始文件)
    = metadata.file_ref.file_hash_hex        (Step 3)
    = manifest.evidence_items.file_hash       (Step 3)
    = TSR.messageImprint.hashedMessage        (Step 5)
    = checksums.sha256 中的记录               (Step 2)

  如果 10 步全部通过:
  → 结论: 证据文件自采集以来内容未被修改，
          在 TSA 签发时间之前已存在，
          采集和操作全过程有完整审计记录
```

---

## 6. 导出流程（App 内部执行步骤）

```
用户点击"导出证据包"
     │
     ▼
Step A: 前置验证
  对案件内所有 EvidenceItem 执行 S07:VerifyEvidenceRecord
  如有任一失败 → 阻断导出，提示用户
     │ 全部通过
     ▼
Step B: 创建临时目录
  {temp}/EvidencePackage_{case_id_short}_{date}/
     │
     ▼
Step C: 逐条证据处理
  for each EvidenceItem:
    C1. S05:DecryptFile → 解密到 evidence/ 目录
    C2. 验证解密后哈希 = file_ref.file_hash_hex
    C3. 生成 metadata JSON → metadata/ 目录
    C4. 复制 TSR 文件 → proofs/tsa_responses/
    C5. 按序号重命名所有文件
     │
     ▼
Step D: 生成全局文件
  D1. 导出 audit/chain_of_custody.json（从 SQLCipher 提取）
  D2. 生成 manifest.json
  D3. 生成 verify/checksums.sha256
  D4. 生成 verify/verify.sh + verify.bat
  D5. 生成 verify/verification_guide.html
  D6. 生成 证据报告.pdf
  D7. 生成 README.txt
     │
     ▼
Step E: 包签名
  E1. 计算包内所有文件的联合哈希
  E2. 生成 package_signature.json
     │
     ▼
Step F: 打包
  F1. ZIP 压缩（不加密——证据包本身就是要给第三方看的）
  F2. 计算 ZIP 文件的 SHA-256
     │
     ▼
Step G: 清理
  G1. 安全擦除临时目录中的解密文件
  G2. AuditLog: "EXPORT_COMPLETE"
     │
     ▼
Step H: 输出
  H1. 保存到用户选择的位置（分享、AirDrop、文件App）
  H2. UI 显示导出成功 + ZIP 文件哈希值
```

---

## 7. package_signature.json

```json
{
  "signature_version": "1.0.0",
  "export_id": "uuid",
  "export_time_utc": "2026-02-08T08:00:00.000Z",

  "package_hash": {
    "algorithm": "SHA-256",
    "scope": "包内所有文件（除 package_signature.json 自身）",
    "method": "按文件相对路径字母序排列，逐个计算 SHA-256，所有哈希值拼接后再 SHA-256",
    "hash_hex": "final_package_hash..."
  },

  "file_hashes": {
    "README.txt": "hash...",
    "证据报告.pdf": "hash...",
    "evidence/E001_photo_20260101_143205.jpg": "hash...",
    "manifest.json": "hash...",
    "...": "..."
  },

  "generator": {
    "app": "EvidenceVault",
    "version": "1.0.3",
    "build": "42"
  }
}
```
