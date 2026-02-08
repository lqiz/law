# EvidenceVault v1.0 — 安全录音模块规格

> 录音与拍照的本质差异：拍照是**瞬时**的（快门 → 一帧），录音是**持续**的（可能几十分钟甚至数小时）。
> 这带来三个独有挑战：中途崩溃丢数据、录完裁剪篡改、长文件哈希性能。

---

## 1. 问题域分析

| 挑战 | 拍照（已解决） | 录音（本文解决） |
|------|--------------|----------------|
| 数据生命周期 | 毫秒级，一帧完成 | 分钟~小时级，持续流入 |
| 崩溃风险 | 极低（操作瞬时） | 高（长时间录制随时可能崩溃/来电/被杀） |
| 篡改手段 | 修改像素 | 裁剪头尾、删除中间段、拼接、变速 |
| 内存压力 | 一帧（3-8MB） | 不可能全部在内存（1 小时 ≈ 60MB AAC） |
| 实时性要求 | 无（拍完处理） | 必须实时写入，不能等录完 |

**结论**：录音不可能照搬拍照的"全程内存 → 最后写盘"模式。必须设计分段写入方案。

---

## 2. 两种方案对比

### 方案 A：固定时间窗分片 + 链式哈希（Chained Segments）

```
录音流 ──┬── Segment 0 (0~30s)  ─→ Hash₀ ─→ Encrypt₀ ─→ 写盘
         ├── Segment 1 (30~60s) ─→ Hash₁(含 Hash₀) ─→ Encrypt₁ ─→ 写盘
         ├── Segment 2 (60~90s) ─→ Hash₂(含 Hash₁) ─→ Encrypt₂ ─→ 写盘
         └── ...
         结束 ─→ 生成 Manifest（含所有段哈希链）─→ 最终 record_hash
```

**核心思路**：每 N 秒切一段，每段独立加密写盘，段间通过链式哈希绑定顺序。

### 方案 B：流式哈希 + 加密分段落盘（Streaming Hash）

```
录音流 ──→ SHA-256 hasher（持续喂入）──→ 每 N 秒从 hasher 导出中间状态
              同时 ↓
         加密缓冲区 ──→ 每 N 秒 flush ──→ 加密追加写入单一 .evault 文件
              同时 ↓
         检查点文件 ──→ 每 N 秒写入 hasher 状态 + 文件偏移量

         结束 ─→ hasher.finalize() ─→ 最终 file_hash
```

**核心思路**：哈希器和加密器始终在线，音频数据流过即处理，不切断文件。

---

## 3. 方案 A 详细设计：固定时间窗分片 + 链式哈希

### 3.1 分片策略

```
分片时长: 30 秒（可配置，v1.0 固定 30s）
文件格式: 每段为独立的 AAC in M4A 容器
编码参数: AAC-LC, 128kbps, 44.1kHz, mono
单段大小: ≈ 480KB
```

**为什么 30 秒**：
- 太短（5s）：段数过多，I/O 开销大，哈希链冗长
- 太长（5min）：崩溃时丢失数据量大
- 30s 平衡：崩溃最多丢 30s，每小时仅 120 段

### 3.2 链式哈希结构

```
Segment 0:
  audio_hash₀  = SHA-256(segment_0.m4a bytes)
  chain_hash₀  = SHA-256("GENESIS" || audio_hash₀ || segment_index=0 || ntp_timestamp₀)

Segment 1:
  audio_hash₁  = SHA-256(segment_1.m4a bytes)
  chain_hash₁  = SHA-256(chain_hash₀ || audio_hash₁ || segment_index=1 || ntp_timestamp₁)

Segment N:
  audio_hashₙ  = SHA-256(segment_n.m4a bytes)
  chain_hashₙ  = SHA-256(chain_hashₙ₋₁ || audio_hashₙ || segment_index=N || ntp_timestampₙ)

最终:
  file_hash    = SHA-256(全部分段合并后的完整音频 bytes)
  manifest_hash = chain_hashₙ (最后一段的链哈希，包含了所有前序段的信息)
```

### 3.3 完整录制流程

```
┌────────────────────── 录制前 ──────────────────────┐
│                                                    │
│ P1. 用户按下录音按钮                                │
│ P2. 锁定 CaptureContext（S02: 设备 + NTP + GPS）   │
│ P3. 生成 evidence_id (UUIDv4)                      │
│ P4. 创建录音会话目录:                               │
│     evidence_store/{case_id}/{evidence_id}_session/ │
│ P5. 初始化 segment_index = 0                       │
│ P6. 初始化 prev_chain_hash = "GENESIS"             │
│ P7. AuditLog: "RECORDING_START"                    │
│                                                    │
└────────────────────────────────────────────────────┘
          │
          ▼
┌────────────────────── 录制中（循环）───────────────────┐
│                                                       │
│ R1. 开始录制到内存缓冲区（系统录音 API → PCM 流）      │
│ R2. 实时 PCM → AAC 编码（硬件编码器）                  │
│ R3. 到达 30 秒边界：                                  │
│     R3.1  停止当前分段编码，获取 segment_bytes         │
│     R3.2  audio_hash = SHA-256(segment_bytes)         │
│     R3.3  chain_hash = SHA-256(                       │
│              prev_chain_hash ||                       │
│              audio_hash ||                            │
│              segment_index ||                         │
│              ntp_timestamp                            │
│           )                                           │
│     R3.4  加密 segment_bytes:                         │
│           iv = random(12 bytes)                       │
│           aad = audio_hash                            │
│           (ciphertext, tag) = AES-256-GCM(...)        │
│     R3.5  原子写入:                                   │
│           → {evidence_id}_session/seg_{index}.evault  │
│           → 更新 checkpoint.json                      │
│     R3.6  清除 segment_bytes 内存                     │
│     R3.7  prev_chain_hash = chain_hash                │
│     R3.8  segment_index++                             │
│ R4. 继续下一段录制                                    │
│                                                       │
│ [崩溃恢复点: 读取 checkpoint.json 可恢复已完成的段]     │
│                                                       │
└───────────────────────────────────────────────────────┘
          │
          ▼
┌────────────────────── 录制结束 ────────────────────────┐
│                                                        │
│ E1. 处理最后一个不满 30s 的分段（同 R3 流程）           │
│ E2. 合并所有解密后的分段 → 完整音频 bytes（内存/流式） │
│ E3. file_hash = SHA-256(完整音频 bytes)                │
│ E4. 加密完整文件 → {evidence_id}.evault                │
│ E5. 生成 SegmentManifest（见下方）                     │
│ E6. 请求 TSA 时间戳（对 file_hash）                    │
│ E7. 组装 EvidenceItem (S06)                            │
│ E8. 删除会话目录（分段临时文件）                        │
│ E9. AuditLog: "RECORDING_SEAL_COMPLETE"                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 3.4 SegmentManifest 数据结构

```json
{
  "evidence_id": "uuid",
  "total_segments": 5,
  "segment_duration_sec": 30,
  "total_duration_sec": 142,
  "encoding": {
    "codec": "AAC-LC",
    "bitrate_kbps": 128,
    "sample_rate_hz": 44100,
    "channels": 1
  },
  "segments": [
    {
      "index": 0,
      "offset_sec": 0.0,
      "duration_sec": 30.0,
      "size_bytes": 481280,
      "audio_hash_hex": "a1b2c3...",
      "chain_hash_hex": "d4e5f6...",
      "ntp_timestamp_utc": "2026-02-08T14:32:05.000Z",
      "encrypt_iv_hex": "...",
      "encrypt_tag_hex": "..."
    },
    {
      "index": 1,
      "offset_sec": 30.0,
      "duration_sec": 30.0,
      "...": "..."
    }
  ],
  "final_chain_hash_hex": "最后一段的 chain_hash（全链锚点）",
  "file_hash_hex": "完整合并文件的 SHA-256（= EvidenceItem.file_ref.file_hash_hex）"
}
```

### 3.5 崩溃恢复机制

**checkpoint.json**（每完成一段更新）：

```json
{
  "evidence_id": "uuid",
  "case_id": "uuid",
  "capture_context": { "...frozen..." },
  "completed_segments": 3,
  "prev_chain_hash_hex": "当前链头",
  "segments": [
    { "index": 0, "audio_hash_hex": "...", "chain_hash_hex": "...", "file": "seg_0.evault" },
    { "index": 1, "audio_hash_hex": "...", "chain_hash_hex": "...", "file": "seg_1.evault" },
    { "index": 2, "audio_hash_hex": "...", "chain_hash_hex": "...", "file": "seg_2.evault" }
  ],
  "status": "recording",
  "last_checkpoint_utc": "2026-02-08T14:33:35.000Z"
}
```

**恢复流程**：
```
App 重新启动:
  1. 扫描 evidence_store 下所有 *_session/ 目录
  2. 读取 checkpoint.json
  3. 如果 status = "recording":
     → 弹窗: "发现一段未完成的录音 (已录制 {N} 段约 {N*30} 秒)，是否保存已录部分？"
     → 用户选择"保存":
        a. 跳过合并（仅有分段，没有完整文件）
        b. 合并已有分段 → 计算 file_hash → 加密 → TSA
        c. SegmentManifest 标记: "recording_interrupted": true
        d. EvidenceItem.mutable.user_note 自动追加:
           "[系统] 录音因应用中断而提前结束，已保存前 {N*30} 秒"
     → 用户选择"丢弃":
        a. SecureWipe 整个 session 目录
        b. AuditLog: "RECORDING_INTERRUPTED_DISCARDED"
```

### 3.6 方案 A 优劣

```
✅ 优点:
   - 崩溃恢复天然支持：每段独立加密落盘，最多丢 30s
   - 实现直觉简单：切段 → 哈希 → 加密 → 写盘，4 步循环
   - 链式哈希证明段序完整：插入/删除/乱序任一操作都会破坏链
   - 分段可独立验证：第三方可逐段校验
   - 内存压力可控：只需保持 30s 音频在内存（≈480KB）

❌ 缺点:
   - 分段边界存在微小断裂（1-2 个 AAC 帧的编码差异）
   - 结束时需合并全部分段为完整文件（额外 I/O）
   - 段数过多时 manifest 较大（1 小时 = 120 段）
   - 分段时间精度受编码器 flush 机制影响
```

---

## 4. 方案 B 详细设计：流式哈希 + 加密追加写入

### 4.1 核心思路

不切断音频文件，全程维护一个连续的加密文件流和一个持续更新的哈希器。

```
                                ┌──────────────┐
PCM 音频流 ──→ AAC 编码器 ──→ │              │──→ SHA-256 Hasher (持续喂入)
                               │  加密缓冲区   │
                               │  (ring buffer)│──→ AES-256-CTR 加密流 ──→ 追加写入 .evault
                               │              │
                               └──────────────┘
                                       │
                                 每 30s 快照 ──→ checkpoint (hasher 中间状态)
```

### 4.2 加密模式选择

**方案 B 不能用 GCM**：
```
GCM 是分组认证加密，必须知道完整明文才能计算 auth_tag。
流式录音场景下，录制结束前无法计算 tag → 无法做流式认证加密。

替代方案: AES-256-CTR (加密) + HMAC-SHA-256 (认证)
  - CTR 模式支持流式加密，无需 padding
  - 录制结束后对整个密文计算 HMAC 作为认证标签
  - 这是 Encrypt-then-MAC 范式
```

### 4.3 流式哈希与检查点

```python
# 伪代码：流式哈希器的检查点机制

class StreamingHasher:
    def __init__(self):
        self.hasher = SHA256()
        self.bytes_fed = 0

    def feed(self, chunk: bytes):
        """每收到一块编码后的音频数据就喂入"""
        self.hasher.update(chunk)
        self.bytes_fed += len(chunk)

    def snapshot(self) -> dict:
        """导出哈希器中间状态用于崩溃恢复"""
        return {
            "hasher_state": self.hasher.copy().hexdigest_partial(),  # 中间状态
            "bytes_fed": self.bytes_fed,
            "snapshot_time_utc": now_utc()
        }

    def finalize(self) -> str:
        """录制结束，输出最终哈希"""
        return self.hasher.hexdigest()
```

**问题：SHA-256 中间状态导出**：
```
标准 SHA-256 API（dart:crypto, CommonCrypto, java.security）
大多不支持导出/恢复中间状态。

解决方案:
  选项 a: 使用 pointycastle 的 SHA-256（Dart 实现，可序列化内部状态）
  选项 b: 分段记录"每 30 秒的累计哈希"（类似 Merkle 链，但非真正的流式恢复）
  选项 c: 不恢复哈希，崩溃后重新从头读取已写入的密文 → 解密 → 重新哈希

v1.0 实际可行的是选项 c：崩溃恢复时重新计算哈希。
```

### 4.4 文件格式

```
.evault 文件内部布局:

┌──────────────────────────────────────┐
│ Header (64 bytes, 明文)              │
│   magic: "EVAULT" (6 bytes)          │
│   version: 0x01 (1 byte)            │
│   mode: 0x02 (stream) (1 byte)      │
│   iv: (16 bytes)                     │
│   reserved: (40 bytes)              │
├──────────────────────────────────────┤
│ Encrypted Audio Stream               │
│   AES-256-CTR(audio_bytes)           │
│   (持续追加写入)                      │
├──────────────────────────────────────┤
│ Trailer (48 bytes, 录制结束时写入)    │
│   hmac: HMAC-SHA-256(密文) (32 bytes)│
│   total_size: (8 bytes)              │
│   checksum: CRC32(header+hmac)       │
│   (8 bytes)                          │
└──────────────────────────────────────┘
```

### 4.5 崩溃恢复

```
崩溃时 Trailer 未写入:
  1. 检测: 文件存在 Header 但无有效 Trailer
  2. 恢复:
     a. 从 Header 读取 IV
     b. 解密已写入的密文部分
     c. 重新计算 file_hash（从头读取全部已有数据）
     d. 重新计算 HMAC，写入 Trailer
     e. 标记 recording_interrupted = true

问题: 如果文件末尾刚好在一个 AES-CTR block 中间被截断:
  → 末尾的不完整 block 必须丢弃
  → 可能丢失最多 16 bytes 的音频数据（约 1ms，可忽略）
```

### 4.6 方案 B 优劣

```
✅ 优点:
   - 无分段边界问题：音频文件天然连续
   - 最终文件就是完整文件：无需合并步骤
   - 单文件管理简单：不需要 session 目录和 manifest

❌ 缺点:
   - 不能用 GCM：被迫使用 CTR+HMAC，安全论证更复杂
   - 崩溃恢复需重新读取全部已有数据重算哈希（长录音时耗时）
   - SHA-256 中间状态序列化依赖特定库实现，跨平台一致性风险
   - 流式写入的文件格式是自定义的，第三方验证需要理解格式
   - 追加写入模式下磁盘 I/O 出错可能损坏整个文件（不像方案 A 只损坏一段）
   - HMAC 只在录制结束时计算：录制过程中无法验证已写入数据的完整性
```

---

## 5. 方案对决

| 维度 | 方案 A（分片链式） | 方案 B（流式追加） | 胜出 |
|------|-------------------|-------------------|------|
| **崩溃安全** | 最多丢 30s，已有段完整可用 | 需重读全文件恢复，可能末尾损坏 | **A** |
| **防裁剪** | 链式哈希，删任一段链即断 | 整文件哈希，只能验全量 | **A** |
| **防插入** | 链式哈希绑定序号，插入则链断 | 整文件哈希，只能验全量 | **A** |
| **防乱序** | segment_index 编入链哈希 | N/A（单文件无此问题） | 平 |
| **分段独立验证** | ✓ 每段可单独验 | ✗ 只能验整体 | **A** |
| **加密方案** | AES-256-GCM（标准认证加密） | AES-256-CTR + HMAC（自组合） | **A** |
| **实现复杂度** | 中等（分段循环 + 合并） | 高（流式哈希状态管理 + 自定义格式） | **A** |
| **第三方验证** | 逐段 sha256sum + 链验证 | 需理解自定义文件格式 | **A** |
| **音频连续性** | 段边界有微小 gap（1-2 帧） | 完全连续 | **B** |
| **长录音性能** | 合并时需读写全量 | 已经是单文件 | **B** |
| **依赖复杂度** | 标准库即可 | 需可序列化 SHA-256 实现 | **A** |

### 结论：v1.0 采用方案 A

理由：
1. **司法场景下分段可验证更有价值**——可以证明"第 3 段（1:00-1:30）的内容未被篡改"
2. **GCM > CTR+HMAC**——GCM 是单一标准原语，安全论证更简单，不易出实现错误
3. **崩溃恢复更可靠**——已有分段直接可用，不需要复杂的恢复逻辑
4. **段边界 gap 不构成实际问题**——AAC 帧长约 23ms，gap 在听感和司法上均不构成影响

v2.0 可考虑方案 B 作为可选模式（长时间录音场景优化）。

---

## 6. 方案 A 的防篡改完整分析

### 6.1 攻击场景逐一覆盖

**攻击 1：录完裁剪头尾**

```
企图: 录了 5 分钟，只想保留中间 2 分钟

防御:
  原始录制: seg_0 → seg_1 → ... → seg_9 (共 10 段)

  如果删除 seg_0 和 seg_9:
    ✗ chain_hash₁ = SHA-256(chain_hash₀ || ...)
      → chain_hash₀ 不存在了 → 链头断裂 → 验证失败
    ✗ manifest.total_segments = 10，实际只有 8 段 → 不匹配
    ✗ file_hash = SHA-256(完整 10 段合并) ≠ SHA-256(8 段合并) → 不匹配

  如果重新计算哈希链和 manifest:
    ✗ TSA 时间戳中记录的 file_hash 是原始 10 段的 → 不匹配
    ✗ TSA 时间戳由 CA 私钥签名 → 无法伪造新的 TSA 响应
```

**攻击 2：删除中间某段**

```
企图: 删除 seg_3（包含不利言论）

防御:
  chain_hash₄ = SHA-256(chain_hash₃ || audio_hash₄ || ...)
  chain_hash₃ 依赖 seg_3 的 audio_hash₃
  → 删除 seg_3 后无法重算 chain_hash₃ → chain_hash₄ 验证失败 → 全链断裂
```

**攻击 3：替换某段内容**

```
企图: 用修改过的音频替换 seg_3

防御:
  新音频 bytes ≠ 原 bytes → audio_hash₃' ≠ audio_hash₃
  → chain_hash₃' ≠ chain_hash₃ → chain_hash₄ 验证失败（因为 chain_hash₄ 依赖原 chain_hash₃）
  → 全链断裂
```

**攻击 4：在中间插入新段**

```
企图: 在 seg_2 和 seg_3 之间插入一段伪造音频

防御:
  插入后 seg_3 变成 seg_4，但 segment_index 编入链哈希
  → 原 chain_hash₃ = SHA-256(... || segment_index=3 || ...)
  → 现在 index=3 指向了伪造段 → chain_hash 与原值不同 → 链断裂
```

**攻击 5：录完后用另一台设备重录**

```
企图: 用相同内容在不同设备重录，替换原始录音

防御:
  CaptureContext 在录制前锁定:
    - device_model / os_version / app_version 不同 → record_hash 不匹配
    - ntp_time_utc 不同 → record_hash 不匹配
    - GPS 可能不同 → record_hash 不匹配
  即使以上全部相同:
    - TSA 时间戳中的 file_hash 对应原始录音 → 与新录音不匹配
```

### 6.2 哈希链验证伪代码

```python
def verify_audio_chain(manifest: dict, segment_files: list[bytes]) -> bool:
    """第三方验证：给定 manifest 和所有分段文件，验证录音完整性"""

    assert len(segment_files) == manifest["total_segments"]

    prev_chain_hash = "GENESIS"

    for i, seg_bytes in enumerate(segment_files):
        seg_info = manifest["segments"][i]

        # 1. 验证分段内容哈希
        actual_audio_hash = sha256(seg_bytes)
        assert actual_audio_hash == seg_info["audio_hash_hex"], \
            f"Segment {i}: audio content tampered"

        # 2. 验证链式哈希
        expected_chain_input = (
            prev_chain_hash +
            actual_audio_hash +
            str(seg_info["index"]) +
            seg_info["ntp_timestamp_utc"]
        )
        actual_chain_hash = sha256(expected_chain_input.encode())
        assert actual_chain_hash == seg_info["chain_hash_hex"], \
            f"Segment {i}: chain hash broken"

        prev_chain_hash = actual_chain_hash

    # 3. 验证最终链哈希
    assert prev_chain_hash == manifest["final_chain_hash_hex"]

    # 4. 验证完整文件哈希
    full_audio = b"".join(segment_files)
    assert sha256(full_audio) == manifest["file_hash_hex"]

    return True
```

---

## 7. 分段边界处理细节

### 7.1 AAC 编码器 flush

```
问题: AAC 编码器内部有缓冲区（通常 1024 samples/帧），
      切段时需要 flush 编码器，可能产生不完整帧。

解决:
  1. 切段前调用 encoder.flush()，获取残留样本的编码数据
  2. 将 flush 数据追加到当前段末尾
  3. 新段从下一个完整 PCM 帧开始
  4. 记录精确的样本数偏移量（sample_offset）到 segment 元数据中

结果: 各段合并后在 AAC 帧级别完全连续，无听感 gap
```

### 7.2 时间精度

```
段边界时间戳精度:
  - 系统 AudioRecord/AVAudioEngine callback 精度: ≈ 5ms
  - NTP offset 精度: ≈ 50ms
  - 段时长误差: ±23ms（一个 AAC 帧）

对于司法场景，秒级精度已经远超需求。
```

---

## 8. 平台实现能力

### 8.1 录音 API

| 能力 | iOS | Android |
|------|-----|---------|
| 录音 API | `AVAudioEngine` | `AudioRecord` |
| AAC 硬件编码 | `AVAudioConverter` (AAC) | `MediaCodec` (AAC) |
| 后台录音 | `UIBackgroundModes: audio` | Foreground Service |
| 来电中断 | `AVAudioSession.interruptionNotification` | `AudioManager.OnAudioFocusChangeListener` |
| 录音权限 | `NSMicrophoneUsageDescription` | `RECORD_AUDIO` permission |

### 8.2 Flutter 实现

```
record                     → 跨平台录音（但不支持分段控制）
自封装 MethodChannel        → 调用原生 AudioRecord/AVAudioEngine（精确分段控制）
```

**v1.0 方案**：必须自封装 MethodChannel，因为：
- 需要精确控制分段时机
- 需要获取原始 PCM 数据用于 flush 控制
- 需要在分段切换时无缝衔接（第三方录音库不支持）

### 8.3 来电/中断处理

```
场景: 用户正在录音，突然来电

处理流程:
  1. 系统发出音频中断通知
  2. 立即 flush 当前分段（同 R3 流程）
  3. 暂停录音，UI 显示 "录音已暂停（来电中断）"
  4. AuditLog: "RECORDING_INTERRUPTED_PHONE_CALL"
  5. 通话结束后，用户可选择:
     a. 继续录音 → 开启新分段（chain 继续）
     b. 结束录音 → 触发 E1-E9 流程

  分段 manifest 中标记:
    "interruptions": [
      {
        "after_segment": 3,
        "reason": "phone_call",
        "duration_sec": 45,
        "timestamp_utc": "2026-02-08T14:35:00Z"
      }
    ]
```

---

## 9. 录音模块与 EvidenceItem 的字段映射

录音类 EvidenceItem 相比照片类增加以下字段：

```json
{
  "classification": {
    "media_type": "audio"
  },

  "file_ref": {
    "original_filename": "recording_20260208_143205.m4a",
    "mime_type": "audio/mp4",
    "...": "与照片一致"
  },

  "audio_metadata": {
    "codec": "AAC-LC",
    "bitrate_kbps": 128,
    "sample_rate_hz": 44100,
    "channels": 1,
    "duration_sec": 142.5,
    "segment_count": 5,
    "segment_duration_sec": 30,
    "recording_interrupted": false,
    "interruptions": [],
    "segment_manifest_hash_hex": "对 SegmentManifest JSON 的 SHA-256",
    "final_chain_hash_hex": "最后一段的链哈希"
  }
}
```

**`audio_metadata` 全部标记为 `[H]`**，参与 `record_hash` 计算。

---

## 10. 存储空间估算

| 时长 | 分段数 | 音频文件大小 | 分段加密开销 | Manifest | 总计 |
|------|--------|------------|-------------|----------|------|
| 1 min | 2 | 960 KB | +64 B × 2 | ~1 KB | ~961 KB |
| 5 min | 10 | 4.8 MB | +64 B × 10 | ~3 KB | ~4.8 MB |
| 30 min | 60 | 28.8 MB | +64 B × 60 | ~15 KB | ~28.8 MB |
| 1 hour | 120 | 57.6 MB | +64 B × 120 | ~30 KB | ~57.6 MB |
| 2 hours | 240 | 115.2 MB | +64 B × 240 | ~60 KB | ~115.2 MB |

加密开销（IV 12B + Tag 16B + padding）可忽略不计。
