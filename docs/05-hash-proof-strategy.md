# EvidenceVault v1.0 — 哈希存证与时间戳策略

> 核心命题：如何在不自建基础设施的前提下，证明"某个哈希值在某个时间点之前已经存在"。
> 这是整个证据体系"时间不可抵赖"的唯一支撑。

---

## 1. 四种存证通道分析

### 1.1 通道 A：RFC 3161 TSA（可信时间戳）

```
原理:
  App 发送 SHA-256 哈希 → TSA 服务器用自己的私钥对 (哈希 + 当前时间) 签名
  → 返回 TimeStampToken（TST）
  → 任何人可用 TSA 公钥验证：该哈希在该时间前已存在

网络交互:
  App ──HTTP POST──→ TSA 服务器
       (仅发送 32 字节哈希值 + nonce)
  App ←─HTTP 200──── TSA 服务器
       (返回 TST, ~2-4 KB)
```

| 维度 | 评估 |
|------|------|
| 司法采信度 | ⭐⭐⭐⭐⭐ 中国司法实践中采信率最高，联合信任已有数万判例 |
| 法律依据 | 《电子签名法》第 8 条、最高法《关于互联网法院审理案件若干问题的规定》第 11 条 |
| 实现复杂度 | 低：HTTP POST + ASN.1 解析 |
| 成本 | 联合信任：约 0.5-2 元/次；开发测试：FreeTSA 免费 |
| 隐私保护 | ⭐⭐⭐⭐⭐ 只发哈希，不发原文 |
| 响应时间 | 200-800ms |
| 离线能力 | 无（必须联网） |
| 验证方式 | `openssl ts -verify` 标准工具链，无需专用软件 |

**国内有资质的 TSA 服务商**：

| 服务商 | 资质 | 接口标准 | 司法采信记录 |
|--------|------|---------|-------------|
| 联合信任 (TSA.cn) | 国家密码管理局审批、CMA/CNAS 认证 | RFC 3161 | 全国法院广泛采信，含最高法案例 |
| 国家授时中心 (NTSC) | 中科院直属，国家法定时间源 | RFC 3161 | 权威性最高，但商用接口有限 |
| 数字认证 (BJCA) | 北京 CA 中心 | RFC 3161 + 自有 API | 北京地区法院高采信率 |

### 1.2 通道 B：司法链（天平链 / 北京互联网法院链等）

```
原理:
  App 发送 SHA-256 哈希 → 司法链 API → 哈希写入法院认可的联盟链
  → 返回上链凭证（链上交易 ID + 区块高度 + 区块时间戳）
  → 法院可通过自有节点直接验证

网络交互:
  App ──HTTPS──→ 存证平台 API（如天平链 API）
       (发送哈希 + 业务元数据)
  App ←─HTTPS──── 存证平台
       (返回 txHash, blockHeight, blockTimestamp, certificate_url)
```

| 维度 | 评估 |
|------|------|
| 司法采信度 | ⭐⭐⭐⭐⭐ 法院自建链，直接打通审判系统，部分法院可"一键验证" |
| 法律依据 | 最高法《关于互联网法院审理案件若干问题的规定》第 11 条第 2 款 |
| 实现复杂度 | 中：需要对接各链 SDK/API，各链接口不统一 |
| 成本 | 天平链：企业认证后按量计费，约 1-5 元/次 |
| 隐私保护 | ⭐⭐⭐⭐ 只发哈希，但需要实名注册开发者账号 |
| 响应时间 | 1-10 秒（出块时间） |
| 离线能力 | 无 |
| 验证方式 | 通过司法链官方验证页面或 API |

**主要司法链平台**：

| 平台 | 运营方 | 覆盖范围 | 接入方式 |
|------|--------|---------|---------|
| 天平链 | 北京互联网法院 | 全国（但北京地区最强） | REST API + SDK |
| 至信链 | 腾讯 + 中国网安 | 广州互联网法院等 | REST API |
| 蚂蚁链版权链 | 蚂蚁集团 | 杭州互联网法院等 | REST API |

### 1.3 通道 C：公链（Ethereum 等）

```
原理:
  App 构造一笔交易，将哈希写入交易的 data 字段
  → 交易被矿工打包上链
  → 区块时间戳 + 全球共识 = 时间证明

网络交互:
  App ──JSON-RPC──→ Infura/Alchemy 节点
       (发送签名交易，data 字段包含哈希)
  App ←─JSON-RPC──── 节点
       (返回 txHash → 等待确认 → blockNumber, blockTimestamp)
```

| 维度 | 评估 |
|------|------|
| 司法采信度 | ⭐⭐⭐ 有采信先例（杭州互联网法院 2018 年），但非普遍 |
| 法律依据 | 同上第 11 条，但法官对公链理解程度参差不齐 |
| 实现复杂度 | 中高：需管理私钥、gas 费、交易确认 |
| 成本 | ETH Gas 费波动大，单次约 0.5-5 美元；Layer2 可降至 <0.01 美元 |
| 隐私保护 | ⭐⭐⭐ 哈希公开可查，虽不含原文但可关联用户地址 |
| 响应时间 | 15s - 5min（ETH 出块 + 确认） |
| 离线能力 | 无 |
| 验证方式 | Etherscan 公开可查，任何人可独立验证 |

**⚠️ 法律风险提示**：
```
1. 2021 年国务院十部委通知明确"虚拟货币相关业务属于非法金融活动"
2. App 直接操作 ETH 交易可能触发合规风险
3. 推荐做法：仅将公链作为辅助验证渠道，不作为主要存证方式
4. 如果使用，通过第三方合规存证服务间接上链（如蚂蚁链、至信链已打通公链锚定）
```

### 1.4 通道 D：本地模拟（开发测试用）

```
原理:
  App 本地生成一个模拟 TSA 响应，用本地生成的密钥对签名
  → 格式与真实 TSA 完全一致，但签名者不是可信 CA
  → 仅用于开发、测试、演示

特点:
  - 零网络依赖
  - 零成本
  - 零司法效力
  - 但数据结构和验证流程与生产环境完全一致
```

---

## 2. 版本演进路线

```
                    司法效力
                      ▲
                      │
         v2.0        │     ● TSA + 司法链 + 公链锚定
         天平链       │    ╱  （三重证明，最高可信度）
                      │   ╱
         v1.1        │  ● TSA + 司法链
         司法链       │ ╱   （双重证明）
                      │╱
         v1.0       ● TSA
         TSA         ╱│    （单一但已足够，司法采信率高）
                    ╱  │
         dev      ●    │
         本地模拟  │    │
                  │    └──────────────────────────────→ 实现复杂度
```

| 版本 | 存证通道 | 理由 |
|------|---------|------|
| **dev** | 通道 D（本地模拟） | 开发测试，零依赖 |
| **v1.0** | 通道 A（RFC 3161 TSA） | 司法采信率最高、实现最简单、成本最低、标准化验证工具链 |
| **v1.1** | A + B（TSA + 司法链） | 增加法院直接可验证的链上存证，双重保险 |
| **v2.0** | A + B + C（TSA + 司法链 + 公链锚定） | 全球可验证性，通过合规第三方间接上公链 |

**v1.0 选择 TSA 的决定性理由**：

```
1. 一次 HTTP 调用解决问题，无需管理钱包/私钥/Gas
2. openssl ts -verify 即可验证，法官助理 / 鉴定机构人员可直接操作
3. 联合信任 TSA 在中国已有超过十万份判决书引用
4. RFC 3161 是 IETF 国际标准，不依赖任何特定商业平台
5. 成本可控：即使每天固化 100 条证据，月成本 < 200 元
```

---

## 3. 多通道调度架构

即使 v1.0 只用 TSA，架构层面预留多通道能力，做到**加通道不改上层代码**：

```
                ┌──────────────────────────────┐
                │      EvidenceItem            │
                │  timestamp_proof: HashProof  │
                │  (支持多个 proof)             │
                └──────────────┬───────────────┘
                               │
                ┌──────────────▼───────────────┐
                │    HashProofManager          │
                │    (通道路由 + 重试 + 降级)    │
                └──┬───────┬───────┬───────┬───┘
                   │       │       │       │
              ┌────▼──┐┌───▼───┐┌──▼──┐┌───▼────┐
              │ TSA   ││ 司法链 ││ 公链 ││ 本地   │
              │Channel││Channel││Chan. ││Mockup  │
              └───────┘└───────┘└─────┘└────────┘

调度策略（v1.0 配置）:
  primary:   TSA（联合信任）
  fallback:  TSA（FreeTSA，备用）
  secondary: null（v1.1 开启司法链）
  dev_mode:  LocalMockup
```

---

## 4. HashProof 对象结构

### 4.1 统一结构（支持所有通道类型）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://evidencevault.local/schemas/hash-proof/v1.0.0",
  "title": "HashProof",
  "description": "哈希存证证明对象 — 支持 TSA / 司法链 / 公链 / 本地模拟",
  "type": "object",
  "required": [
    "proof_id",
    "proof_version",
    "channel",
    "status",
    "input",
    "output",
    "created_at"
  ],
  "additionalProperties": false,
  "properties": {

    "proof_id": {
      "type": "string",
      "format": "uuid",
      "description": "[H] 证明对象唯一 ID"
    },

    "proof_version": {
      "const": "1.0.0",
      "description": "[H] 结构版本号"
    },

    "channel": {
      "type": "string",
      "enum": ["tsa_rfc3161", "judicial_chain", "public_chain", "local_mockup"],
      "description": "[H] 存证通道类型"
    },

    "status": {
      "type": "string",
      "enum": [
        "pending",
        "requesting",
        "granted",
        "failed",
        "retry_scheduled"
      ],
      "description": "[D] 当前状态"
    },

    "input": {
      "type": "object",
      "description": "发送给存证服务的输入",
      "required": ["hash_algorithm", "hash_hex", "nonce"],
      "additionalProperties": false,
      "properties": {
        "hash_algorithm": {
          "const": "SHA-256",
          "description": "[H] 哈希算法"
        },
        "hash_hex": {
          "type": "string",
          "pattern": "^[a-f0-9]{64}$",
          "description": "[H] 被存证的哈希值（= EvidenceItem.file_ref.file_hash_hex）"
        },
        "nonce": {
          "type": "string",
          "description": "[H] 防重放随机数"
        }
      }
    },

    "output": {
      "type": "object",
      "description": "存证服务返回的证明数据（按通道类型不同结构不同）",
      "oneOf": [
        { "$ref": "#/$defs/tsa_output" },
        { "$ref": "#/$defs/judicial_chain_output" },
        { "$ref": "#/$defs/public_chain_output" },
        { "$ref": "#/$defs/local_mockup_output" }
      ]
    },

    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "[H] 证明请求发起时间（本地时钟）"
    },

    "granted_at": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "[H][D] 存证服务签发的可信时间（通道方时间，非本地时间）"
    },

    "retry_count": {
      "type": "integer",
      "default": 0,
      "description": "重试次数"
    },

    "last_error": {
      "type": ["string", "null"],
      "description": "最近一次错误信息"
    }
  },

  "$defs": {

    "tsa_output": {
      "type": "object",
      "description": "RFC 3161 TSA 输出",
      "required": [
        "channel_type",
        "tsa_url",
        "tsr_base64",
        "tsa_cert_chain_pem",
        "tsa_serial_number",
        "granted_time_utc",
        "response_status"
      ],
      "additionalProperties": false,
      "properties": {
        "channel_type": {
          "const": "tsa_rfc3161"
        },
        "tsa_url": {
          "type": "string",
          "format": "uri",
          "description": "[H][V] TSA 服务地址"
        },
        "tsr_base64": {
          "type": "string",
          "description": "[V] 完整 TimeStampResp Base64（可直接用 openssl 验证）"
        },
        "tsa_cert_chain_pem": {
          "type": "string",
          "description": "[V] TSA 签名证书链 PEM 格式"
        },
        "tsa_serial_number": {
          "type": "string",
          "description": "[V] TSA 序列号"
        },
        "granted_time_utc": {
          "type": "string",
          "format": "date-time",
          "description": "[H][D] TSA 签发的可信时间"
        },
        "response_status": {
          "type": "integer",
          "description": "[V] TSA 响应状态码（0=granted, 1=grantedWithMods, 2=rejection）"
        }
      }
    },

    "judicial_chain_output": {
      "type": "object",
      "description": "司法链输出（天平链 / 至信链等）",
      "required": [
        "channel_type",
        "chain_name",
        "api_endpoint",
        "tx_hash",
        "block_height",
        "block_timestamp_utc",
        "certificate_url"
      ],
      "additionalProperties": false,
      "properties": {
        "channel_type": {
          "const": "judicial_chain"
        },
        "chain_name": {
          "type": "string",
          "enum": ["tianping", "zhixin", "antchain_copyright"],
          "description": "[H][V] 司法链名称"
        },
        "api_endpoint": {
          "type": "string",
          "format": "uri",
          "description": "[V] API 端点"
        },
        "tx_hash": {
          "type": "string",
          "description": "[H][V] 链上交易哈希"
        },
        "block_height": {
          "type": "integer",
          "description": "[H][V] 区块高度"
        },
        "block_timestamp_utc": {
          "type": "string",
          "format": "date-time",
          "description": "[H][D] 区块时间戳"
        },
        "certificate_url": {
          "type": "string",
          "format": "uri",
          "description": "[D] 存证证书在线查验 URL"
        },
        "raw_response_base64": {
          "type": "string",
          "description": "[V] 原始 API 响应 Base64（留存完整性）"
        }
      }
    },

    "public_chain_output": {
      "type": "object",
      "description": "公链输出（v2.0）",
      "required": [
        "channel_type",
        "chain_id",
        "tx_hash",
        "block_number",
        "block_timestamp_utc"
      ],
      "additionalProperties": false,
      "properties": {
        "channel_type": {
          "const": "public_chain"
        },
        "chain_id": {
          "type": "string",
          "description": "[H][V] 链标识（如 'ethereum_mainnet', 'polygon'）"
        },
        "tx_hash": {
          "type": "string",
          "pattern": "^0x[a-f0-9]{64}$",
          "description": "[H][V] 交易哈希"
        },
        "block_number": {
          "type": "integer",
          "description": "[H][V] 区块号"
        },
        "block_timestamp_utc": {
          "type": "string",
          "format": "date-time",
          "description": "[H][D] 区块时间戳"
        },
        "explorer_url": {
          "type": "string",
          "format": "uri",
          "description": "[D] 区块链浏览器查验 URL"
        },
        "contract_address": {
          "type": ["string", "null"],
          "description": "[V] 如果通过合约存证，合约地址"
        }
      }
    },

    "local_mockup_output": {
      "type": "object",
      "description": "本地模拟输出（仅开发测试）",
      "required": [
        "channel_type",
        "mock_granted_time_utc",
        "mock_signature_hex",
        "warning"
      ],
      "additionalProperties": false,
      "properties": {
        "channel_type": {
          "const": "local_mockup"
        },
        "mock_granted_time_utc": {
          "type": "string",
          "format": "date-time",
          "description": "模拟签发时间（设备本地时钟，不可信）"
        },
        "mock_signature_hex": {
          "type": "string",
          "description": "本地密钥签名（格式与 TSA 一致，但不可信）"
        },
        "warning": {
          "const": "LOCAL_MOCKUP_NO_LEGAL_VALIDITY",
          "description": "强制警告标识：此证明无任何法律效力"
        }
      }
    }
  }
}
```

### 4.2 EvidenceItem 中的挂载方式

```
原 02-evidence-item-schema.md 中 timestamp_proof 是单一对象。
现在升级为 proofs 数组，支持多通道：
```

```json
{
  "hash_proofs": {
    "type": "array",
    "items": { "$ref": "hash-proof/v1.0.0" },
    "minItems": 0,
    "description": "一条证据可以有多个存证证明（不同通道）"
  },
  "primary_proof_id": {
    "type": ["string", "null"],
    "format": "uuid",
    "description": "[H] 主证明 ID（参与 record_hash 计算的那个）"
  }
}
```

**哈希计算规则更新**：
```
record_hash 只纳入 primary_proof 的 [H] 字段。
secondary proofs 作为辅助验证，不影响 record_hash。
这样新增通道不会破坏已有证据的完整性。
```

---

## 5. 第三方验证指南（按通道类型）

### 5.1 TSA 验证（v1.0 标配）

```bash
# 环境: 任何安装了 openssl 的机器

# 1. 准备文件
#    evidence.jpg       — 原始证据文件（从证据包 evidence/ 目录获取）
#    response.tsr       — TSA 响应文件（从证据包 certificates/ 目录获取）
#    tsa_ca.pem         — TSA 根证书（从证据包 certificates/ 目录获取）

# 2. 验证文件哈希
sha256sum evidence.jpg
# 输出: a3f7b2c1...c921  evidence.jpg
# 比对 hash_manifest.json 中记录的哈希值

# 3. 验证 TSA 时间戳
openssl ts -verify \
  -in response.tsr \
  -data evidence.jpg \
  -CAfile tsa_ca.pem

# 期望输出: Verification: OK

# 4. 查看 TSA 签发时间
openssl ts -reply -in response.tsr -text
# 输出中的 "Time stamp" 行即为可信时间

# 验证通过的含义:
# → evidence.jpg 的 SHA-256 哈希值
# → 在 TSA 签发时间之前已存在
# → TSA 签名由受信 CA 签发，不可伪造
```

### 5.2 司法链验证（v1.1）

```
# 1. 访问天平链验证页面
#    https://tiantong.court.gov.cn/verify（示例）

# 2. 输入 tx_hash
#    或上传 证据包中的 chain_certificate.json

# 3. 页面返回:
#    ✓ 链上存证时间: 2026-02-08 14:32:05
#    ✓ 存证哈希: a3f7b2c1...c921
#    ✓ 区块高度: 1,234,567
#    ✓ 存证状态: 有效

# 4. 本地验证文件哈希
sha256sum evidence.jpg
# 比对链上记录的哈希值
```

### 5.3 公链验证（v2.0）

```
# 1. 访问区块链浏览器
#    https://etherscan.io/tx/{tx_hash}

# 2. 查看交易 Input Data 字段
#    解码后应包含: SHA-256 哈希值

# 3. 比对:
#    链上哈希 == sha256sum evidence.jpg == hash_manifest.json 中的值

# 4. 区块时间戳即为存证时间
```

---

## 6. HashProofManager 调度逻辑

```
┌────────────────────────────────────────────────────────────────────┐
│                     HashProofManager                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  requestProof(file_hash_hex):                                      │
│                                                                    │
│    1. 从配置读取 enabled_channels:                                  │
│       v1.0:  ["tsa_rfc3161"]                                       │
│       v1.1:  ["tsa_rfc3161", "judicial_chain"]                     │
│       v2.0:  ["tsa_rfc3161", "judicial_chain", "public_chain"]     │
│       dev:   ["local_mockup"]                                      │
│                                                                    │
│    2. 按优先级依次请求:                                              │
│                                                                    │
│       ┌─── primary (第一个成功的即为主证明) ───┐                     │
│       │                                       │                    │
│       │  try TSA (联合信任)                     │                    │
│       │    ├─ 成功 → primary_proof ✓            │                    │
│       │    └─ 失败 → try TSA (FreeTSA 备用)     │                    │
│       │              ├─ 成功 → primary_proof ✓  │                    │
│       │              └─ 失败 → mark pending     │                    │
│       │                 加入重试队列             │                    │
│       └───────────────────────────────────────┘                    │
│                                                                    │
│       ┌─── secondary (异步，不阻塞主流程) ────┐                     │
│       │                                       │                    │
│       │  if 司法链 enabled:                    │                    │
│       │    async request → 成功则追加到 proofs  │                    │
│       │                                       │                    │
│       │  if 公链 enabled:                      │                    │
│       │    async request → 成功则追加到 proofs  │                    │
│       └───────────────────────────────────────┘                    │
│                                                                    │
│  retryPending():                                                   │
│    定时任务（每 5 分钟），重试所有 pending/failed 的 proof            │
│    指数退避: 5min → 15min → 45min → 2h → 6h → 停止（标记 failed）   │
│                                                                    │
│  verifyProof(proof: HashProof) → VerifyResult:                     │
│    根据 channel 类型分发到对应验证器                                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 7. 离线降级策略

```
场景: 用户在无网络环境下（地下室、电梯、飞行模式）录音取证

时间线:
  T0  用户按下拍照/录音
  T1  采集完成，尝试 TSA 请求
  T2  网络不可用，TSA 请求失败

降级流程:
  1. HashProof.status = "pending"
  2. 记录三个辅助时间锚点:
     a. NTP 时间（App 启动时缓存的 offset，即使离线也可算出近似 NTP 时间）
     b. 设备本地时钟
     c. GPS 时间（如果 GPS 可用，卫星授时不依赖网络）
  3. EvidenceItem 正常创建，record_hash 中 TSA 字段为 pending 状态
  4. UI 显示: "✓ 证据已本地固化   ⏳ 时间戳待联网后获取"
  5. 加入重试队列
  6. AuditLog: "TSA_REQUEST_DEFERRED, reason=network_unavailable"

网络恢复后:
  7. 后台自动重试 TSA 请求
  8. 成功后更新 HashProof.status = "granted"
  9. 重算 record_hash（纳入 TSA 字段）
  10. AuditLog: "TSA_REQUEST_COMPLETED, delay_sec={T_granted - T_created}"

⚠️ 司法效力影响:
  - TSA 时间 = 联网补戳时间，不是拍摄时间
  - 但 NTP 缓存时间 + GPS 时间 + 设备时钟三者交叉验证
    可以辅助论证拍摄时间的合理性
  - 延迟越短，辅助论证越强

  建议 App UI 提示:
  "离线期间采集的证据，建议在 24 小时内联网完成时间戳认证，
   以获得最强法律效力"
```

---

## 8. 成本估算

| 方案 | 单次成本 | 月 100 条 | 月 1000 条 | 年度预算 |
|------|---------|----------|-----------|---------|
| 本地模拟 | ¥0 | ¥0 | ¥0 | ¥0 |
| TSA（联合信任） | ¥0.5-2 | ¥50-200 | ¥500-2000 | ¥6K-24K |
| 天平链 | ¥1-5 | ¥100-500 | ¥1K-5K | ¥12K-60K |
| ETH 主网 | ¥3-35 | ¥300-3500 | 不现实 | — |
| ETH L2 (Polygon) | ¥0.01-0.1 | ¥1-10 | ¥10-100 | ¥120-1200 |

**v1.0 成本模型**：
```
TSA 成本由用户无感承担（内含在 App 订阅/一次性付费中）
假设人均月采集 30 条证据:
  TSA 成本 ≈ 30 × ¥1 = ¥30/月/用户
  App 订阅定价 ¥15-30/月 或 ¥198/年 即可覆盖
  或：免费用户限额 5 条/月，付费无限制
```

---

## 9. 配置文件结构

```json
{
  "hash_proof_config": {
    "mode": "production",
    "enabled_channels": ["tsa_rfc3161"],
    "primary_channel": "tsa_rfc3161",

    "tsa_rfc3161": {
      "providers": [
        {
          "name": "联合信任",
          "url": "https://tsa.tsa.cn/",
          "priority": 1,
          "timeout_ms": 5000,
          "ca_cert_bundled": true
        },
        {
          "name": "FreeTSA (备用/测试)",
          "url": "https://freetsa.org/tsr",
          "priority": 2,
          "timeout_ms": 8000,
          "ca_cert_bundled": true
        }
      ]
    },

    "judicial_chain": {
      "enabled": false,
      "provider": "tianping",
      "api_endpoint": "https://api.tianping.court.gov.cn/v1/evidence",
      "app_id": "配置时填入",
      "app_secret_ref": "keystore://judicial_chain_secret"
    },

    "public_chain": {
      "enabled": false,
      "chain": "polygon",
      "rpc_url": "https://polygon-rpc.com",
      "contract_address": "0x..."
    },

    "retry_policy": {
      "max_retries": 5,
      "backoff_minutes": [5, 15, 45, 120, 360],
      "give_up_after_hours": 24
    },

    "offline_fallback": {
      "use_cached_ntp_offset": true,
      "use_gps_time": true,
      "prompt_user_to_connect_within_hours": 24
    }
  }
}
```
