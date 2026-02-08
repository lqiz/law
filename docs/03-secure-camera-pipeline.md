# EvidenceVault v1.0 — 安全相机流水线规格

> 核心目标：从快门按下到文件落盘，全程无可篡改窗口。
> 任何"拍完再改"的企图都会导致哈希链断裂。

---

## 1. 总览：快门到落盘的 14 步流水线

```
用户按下快门
     │
     ▼
┌─────────────────────────── 同步阶段（<2s 内完成）──────────────────────────┐
│ Step 1:  锁定环境快照（NTP 时间 + GPS + 设备指纹）                         │
│ Step 2:  系统相机 API 捕获原始帧                                          │
│ Step 3:  注入不可逆水印层                                                 │
│ Step 4:  合成最终图像 → 内存中的 JPEG bytes                               │
│ Step 5:  计算 SHA-256（对内存 bytes，非磁盘文件）                          │
│ Step 6:  生成 AES-256-GCM 密钥/IV → 加密 bytes                           │
│ Step 7:  原子写入：加密文件 + 元数据 → 磁盘                               │
│ Step 8:  写入 AuditLog（创建条目）                                        │
└────────────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────── 异步阶段（后台完成）────────────────────────────┐
│ Step 9:  请求 RFC 3161 TSA 时间戳                                         │
│ Step 10: 写回 timestamp_proof 到 EvidenceItem                             │
│ Step 11: 重算 record_hash（纳入 TSA 字段）                                │
│ Step 12: 写入 AuditLog（固化完成）                                        │
│ Step 13: 清除内存中的明文 bytes                                           │
│ Step 14: UI 反馈 → 固化成功 ✓                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 每一步详细规格

### Step 1: 锁定环境快照

**时机**：快门按下的第一个回调，在图像捕获之前。

**动作**：
```
1.1  读取 NTP 校准时间（使用 App 启动时缓存的 NTP offset）
1.2  读取设备本地时钟
1.3  计算 clock_drift_ms = ntp_time - local_clock
1.4  读取 GPS（使用最近一次已授权的位置，不阻塞等待新定位）
1.5  读取设备型号、OS 版本、App 版本、网络类型、电量
1.6  将以上全部写入 CaptureContext 结构体，加盖 frozen 标记
```

**平台能力**：

| 能力 | iOS | Android |
|-----|-----|---------|
| NTP 时间 | `TrueTime` 库（缓存 offset） | `TrueTime` for Android |
| GPS | `CLLocationManager.location`（最近缓存） | `FusedLocationProviderClient.lastLocation` |
| 设备型号 | `utsname.machine` | `Build.MODEL` |
| OS 版本 | `UIDevice.systemVersion` | `Build.VERSION.RELEASE` |
| 网络类型 | `NWPathMonitor` | `ConnectivityManager` |
| 电量 | `UIDevice.batteryLevel` | `BatteryManager` |

**Flutter 实现**：
```
device_info_plus      → 设备型号、OS 版本
geolocator            → GPS
connectivity_plus     → 网络类型
battery_plus          → 电量
true_time (自封装)    → NTP offset
```

**防篡改设计**：
- CaptureContext 一旦 frozen，任何字段均不可修改
- clock_drift_ms 记录了设备时钟偏移，事后鉴定可交叉验证
- 若 GPS 未授权，字段为 null，不伪造

---

### Step 2: 系统相机 API 捕获原始帧

**动作**：
```
2.1  调用系统相机 API 获取最高质量原始图像数据
2.2  获取原始 EXIF 数据（系统写入的，非 App 构造的）
2.3  原始图像 bytes 保持在内存中，不写入磁盘
```

**平台能力**：

| 能力 | iOS | Android |
|-----|-----|---------|
| 相机捕获 | `AVCapturePhotoOutput` | `CameraX ImageCapture` |
| 原始帧获取 | `photoOutput(_:didFinishProcessingPhoto:)` → `AVCapturePhoto.fileDataRepresentation()` | `ImageCapture.takePicture()` → `ImageProxy` |
| EXIF 读取 | `CGImageSource` / `PHAsset` | `ExifInterface` |
| 视频帧 | `AVCaptureMovieFileOutput` | `CameraX VideoCapture` |

**Flutter 实现**：
```
camera                → 相机预览 + 拍照
本项目自封装 MethodChannel → 获取原始 bytes（不经过 Flutter image codec）
```

**防篡改设计**：
- 不使用 `ImagePicker`（它会经过系统相册，有被中间修改的窗口）
- 直接从相机硬件回调获取 bytes，全程在内存中操作
- 不保存到系统相册（避免其他 App 访问）

---

### Step 3: 注入不可逆水印层

**动作**：
```
3.1  根据 Step 1 的 CaptureContext 生成水印内容
3.2  将水印烧录到图像像素中（不可分离）
3.3  水印分为两层：可见水印 + 隐式指纹
```

**水印生成规则**（详见第 3 节）：

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│                   [照片主体内容]                      │
│                                                     │
│                                                     │
│                                                     │
│                                                     │
│                                                     │
│                                                     │
├─────────────────────────────────────────────────────┤
│ ▎可见水印条（底部半透明黑底白字）                      │
│ ▎                                                   │
│ ▎ EvidenceVault                                     │
│ ▎ 2026-02-08 14:32:05 CST (UTC+8)                  │
│ ▎ 31.2304°N, 121.4737°E ± 8m                       │
│ ▎ iPhone 15 Pro · iOS 18.2                          │
│ ▎ ID: a3f7...c921                                   │
│ ▎                                                   │
└─────────────────────────────────────────────────────┘
```

**防篡改设计**：
- 水印直接写入像素矩阵，不是 EXIF/metadata 层（EXIF 可被工具剥离）
- 水印在 JPEG 压缩前注入，是图像内容的一部分
- 修改水印 = 修改像素 = 哈希变化 = 完整性校验失败

---

### Step 4: 合成最终图像

**动作**：
```
4.1  将水印后的图像编码为 JPEG（quality=95，平衡清晰度和大小）
4.2  写入 EXIF：仅保留相机硬件参数（焦距、ISO、曝光）
4.3  不写入 GPS 到 EXIF（GPS 信息仅存在于加密的 CaptureContext 中）
4.4  结果：内存中的 final_jpeg_bytes
```

**为什么不存 RAW / PNG**：
- RAW 文件过大（20-50MB），移动端存储压力
- PNG 无损但体积大，且照片场景无必要
- JPEG quality=95 视觉无损，体积合理（3-8MB）
- 法院对 JPEG 照片证据有广泛采信先例

**防篡改设计**：
- 不写 GPS 到 EXIF：防止第三方工具读取 EXIF 获取位置隐私
- GPS 存在加密的 CaptureContext 中，仅导出时可见

---

### Step 5: 计算 SHA-256 哈希

**时机**：紧接 Step 4，对内存中的 `final_jpeg_bytes` 计算，不是对磁盘文件。

**动作**：
```
5.1  input  = final_jpeg_bytes（内存）
5.2  output = SHA-256(input) → 64 字符 hex string
5.3  同时记录 file_size_bytes = input.length
```

```dart
// Dart 伪代码
import 'package:crypto/crypto.dart';
final digest = sha256.convert(finalJpegBytes);
final fileHashHex = digest.toString(); // "a3f7...c921"
```

**为什么在内存中算而不是写盘后算**：

```
 ✗ 错误做法：写入磁盘 → 读取磁盘 → 计算哈希
   风险：写入和读取之间存在时间窗口，文件可能被篡改（root 设备/恶意进程）

 ✓ 正确做法：内存中的 bytes → 计算哈希 → 加密写入磁盘
   保证：哈希对应的 bytes 与加密的 bytes 是完全相同的内存对象
```

**这是整个流水线防篡改的关键点**：哈希计算发生在明文 bytes 存在于内存的唯一时刻，之后 bytes 要么被加密，要么被清除，没有"拍完再改"的窗口。

---

### Step 6: 加密

**动作**：
```
6.1  从 Keystore/Keychain 获取主密钥引用（key_ref）
6.2  生成随机 IV（12 bytes / 96 bits）
6.3  AES-256-GCM 加密：
     - plaintext  = final_jpeg_bytes（与 Step 5 使用的是同一个内存对象）
     - key        = 主密钥
     - iv         = 随机 IV
     - aad        = file_hash_hex（将哈希绑定到密文，防止密文替换攻击）
     - output     = encrypted_bytes + auth_tag (16 bytes)
6.4  记录 iv_hex, auth_tag_hex
```

**密钥管理方案**（详见第 4 节）：

| 层级 | 名称 | 存储位置 | 用途 |
|------|------|---------|------|
| L1 | 用户 PIN/生物识别 | 用户记忆/生物特征 | 解锁 App |
| L2 | Master Key (MK) | Android Keystore / iOS Keychain | 加解密 DEK |
| L3 | Data Encryption Key (DEK) | 加密存储在 SQLCipher 中 | 加解密证据文件 |

**平台能力**：

| 能力 | iOS | Android |
|-----|-----|---------|
| 硬件密钥存储 | Secure Enclave + Keychain | StrongBox / TEE + Keystore |
| 密钥不可导出 | `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | `setIsStrongBoxBacked(true)` |
| 生物识别绑定 | `kSecAccessControlBiometryCurrentSet` | `setUserAuthenticationRequired(true)` |
| AES-GCM 硬件加速 | Apple A-series AES 引擎 | ARMv8 Cryptography Extension |

**Flutter 实现**：
```
flutter_secure_storage     → Keychain/Keystore 抽象
pointycastle               → AES-256-GCM（纯 Dart 实现，跨平台一致）
自封装 MethodChannel        → 调用原生 Keystore/Keychain API（更精细控制）
```

---

### Step 7: 原子写入磁盘

**动作**：
```
7.1  构建 EvidenceItem JSON（除 timestamp_proof 和 integrity 外的所有字段）
7.2  计算临时 record_hash（timestamp_proof.status = "pending"）
7.3  数据库事务 BEGIN
7.4    写入加密文件到沙箱：  {evidence_id}.evault
7.5    写入 EvidenceItem 到 SQLCipher
7.6  数据库事务 COMMIT
7.7  验证：读取文件前 16 bytes 确认写入成功
```

**原子性保证**：
```
如果 7.4 成功但 7.5 失败 → 事务回滚 → 删除孤立的 .evault 文件
如果 7.5 成功但 7.4 失败 → 事务回滚 → 不存在不一致状态
两者都成功 → COMMIT → 证据完整落盘
```

**文件存储路径**：
```
{app_sandbox}/
└── evidence_store/
    ├── {case_id}/
    │   ├── {evidence_id}.evault          # 加密的证据文件
    │   ├── {evidence_id}.evault.meta     # 加密参数（iv, tag, key_ref）
    │   └── ...
    └── ...
```

**平台能力**：

| 能力 | iOS | Android |
|-----|-----|---------|
| 沙箱路径 | `NSDocumentDirectory` | `getFilesDir()` |
| 文件保护 | `NSFileProtectionComplete` | `EncryptedFile` (Jetpack Security) |
| 数据库加密 | SQLCipher iOS | SQLCipher Android |

---

### Step 8: 写入 AuditLog

**动作**：
```
8.1  记录操作：action = "CAPTURE_AND_SEAL_START"
8.2  记录关联：evidence_id, case_id
8.3  生成 log_hash，链接 prev_log_hash
8.4  写入 audit_log 表
```

---

### Step 9: 请求 TSA 时间戳（异步）

**动作**：
```
9.1  构造 RFC 3161 TimeStampReq：
     - messageImprint.hashAlgorithm = SHA-256
     - messageImprint.hashedMessage = file_hash_hex（Step 5 的输出）
     - nonce = 随机 64-bit 整数
     - certReq = true（要求 TSA 返回证书）
9.2  HTTP POST → TSA 服务器
9.3  解析 TimeStampResp
9.4  验证返回的哈希 = 我方发送的哈希
9.5  验证 nonce 一致
9.6  提取 granted_time_utc
```

**TSA 服务器选型**：
```
首选：联合信任时间戳服务 (TSA.cn)
  - 国内司法采信率最高
  - 通过国家密码管理局审批
  - 支持 RFC 3161

备选：国家授时中心 (NTSC)
  - 国家级时间源

开发/测试：FreeTSA.org
  - 免费，仅用于开发阶段
```

**离线降级**：
```
如果网络不可用：
  9.x  timestamp_proof.status = "pending"
  9.x  记录 ntp_time_utc 作为临时时间参考
  9.x  加入重试队列，网络恢复后自动补时间戳
  9.x  AuditLog 记录 "TSA_REQUEST_DEFERRED"
```

---

### Step 10-12: 写回 + 重算 + 日志

**动作**：
```
10.1  将 TSA 响应写入 EvidenceItem.timestamp_proof
10.2  timestamp_proof.status = "granted"
11.1  重算 record_hash（此时 TSA 字段已纳入）
11.2  更新 integrity.record_hash_hex
12.1  AuditLog: action = "CAPTURE_AND_SEAL_COMPLETE"
```

---

### Step 13-14: 清除 + 反馈

**动作**：
```
13.1  将内存中的 final_jpeg_bytes 置零并释放
13.2  将内存中的明文密钥材料置零
13.3  触发 Dart GC（无法强制，但已置零，即使 GC 延迟也无明文残留）
14.1  UI 显示固化完成状态
14.2  显示摘要：时间、位置、文件大小、哈希前 8 位
```

**平台能力**：

| 能力 | iOS | Android |
|-----|-----|---------|
| 安全内存清除 | `memset_s` / `Data(count:).withUnsafeMutableBytes` | `Arrays.fill(byteArray, 0)` |

---

## 3. 水印生成规则详细规格

### 3.1 可见水印

**位置**：图像底部，高度 = 图像高度 × 8%，半透明黑底（alpha=0.7）。

**内容模板**（5 行）：
```
Line 1: "EvidenceVault"                                      ← 品牌标识
Line 2: "{yyyy-MM-dd HH:mm:ss} {timezone_abbr} (UTC{offset})" ← NTP 校准时间
Line 3: "{lat}°N/S, {lng}°E/W ± {accuracy}m"                 ← GPS（无授权则显示 "位置未授权"）
Line 4: "{device_model} · {os_version}"                       ← 设备标识
Line 5: "ID: {evidence_id 前 8 位}...{后 4 位}"               ← 证据短 ID
```

**示例**：
```
EvidenceVault
2026-02-08 14:32:05 CST (UTC+8)
31.2304°N, 121.4737°E ± 8m
iPhone 15 Pro · iOS 18.2
ID: a3f7b2c1...c921
```

**样式规格**：
```
字体：系统等宽字体（iOS: Menlo, Android: monospace）
字号：图像宽度 / 40（自适应）
颜色：#FFFFFF，alpha=1.0
背景：#000000，alpha=0.7
左边距：图像宽度 × 2%
行间距：字号 × 1.4
```

### 3.2 隐式指纹（v1.0 简化方案）

**方案**：在 JPEG EXIF 的 `UserComment` 字段中写入签名数据。

```json
{
  "ev_version": "1.0",
  "evidence_id": "完整 UUID",
  "file_hash_prefix": "SHA-256 前 16 字符",
  "ntp_timestamp": "ISO 8601 UTC"
}
```

**为什么 v1.0 不做 LSB 隐写**：
- LSB 隐写在 JPEG 有损压缩后不可靠
- 实现复杂度高，与 MVP 优先原则冲突
- EXIF UserComment + 可见水印 + SHA-256 哈希已构成充分的证明链
- v2.0 可考虑基于 DCT 系数的鲁棒水印

### 3.3 水印时间来源优先级

```
优先级 1: NTP 校准时间（App 启动时同步，精度 ±50ms）
优先级 2: GPS 时间（卫星授时，精度 ±100ns，但需 GPS 授权）
优先级 3: 设备本地时钟（最低可信度，记录 drift 供事后比对）

实际使用: 取优先级 1，同时记录优先级 3 及两者差值
```

---

## 4. 文件加密方案完整规格

### 4.1 密钥层级架构

```
┌─────────────────────────────────────────────────┐
│           Layer 1: User Authentication          │
│         PIN (6+ digits) / Biometrics            │
│                     │                           │
│                     ▼ 解锁                      │
│    ┌────────────────────────────────┐            │
│    │  Layer 2: Master Key (MK)     │            │
│    │  AES-256, 存于 Keystore       │            │
│    │  不可导出，硬件保护            │            │
│    │         │                     │            │
│    │         ▼ 解密                │            │
│    │  ┌─────────────────────┐     │            │
│    │  │ Layer 3: DEK Pool   │     │            │
│    │  │ (Data Encryption    │     │            │
│    │  │  Keys)              │     │            │
│    │  │                     │     │            │
│    │  │ DEK_1 → Case_A     │     │            │
│    │  │ DEK_2 → Case_B     │     │            │
│    │  │ ...                 │     │            │
│    │  │ 加密存储于 SQLCipher │     │            │
│    │  └─────────────────────┘     │            │
│    └────────────────────────────────┘            │
└─────────────────────────────────────────────────┘
```

### 4.2 密钥生命周期

**MK（Master Key）生成**：
```
时机：App 首次启动，用户设置 PIN/生物识别后
生成：Keystore/Keychain 内部生成（密钥材料不离开安全硬件）
属性：
  - 算法: AES-256
  - 用途: ENCRYPT | DECRYPT
  - 不可导出: true
  - 需要用户认证: true（PIN 或生物识别）
  - 认证有效期: 30 秒（每次操作需重新认证）
```

**DEK（Data Encryption Key）生成**：
```
时机：创建新案件时
生成：CSPRNG 生成 32 bytes 随机密钥
存储：使用 MK 加密后存入 SQLCipher 的 key_store 表
关联：每个 Case 一个 DEK
```

**平台 API 对照**：

| 操作 | iOS | Android |
|------|-----|---------|
| 生成 MK | `SecKeyCreateRandomKey(kSecAttrKeySizeInBits: 256)` | `KeyGenerator.getInstance("AES", "AndroidKeyStore")` |
| 硬件保护 | Secure Enclave（A7+） | StrongBox（Titan M）/ TEE |
| 生物识别绑定 | `SecAccessControl(.biometryCurrentSet)` | `setUserAuthenticationRequired(true)` |
| 加密操作 | `SecKeyCreateEncryptedData` | `Cipher.getInstance("AES/GCM/NoPadding")` |
| 密钥不可导出 | `kSecAttrIsPermanent: true` | `setKeyEntry` 无 `getEncoded()` |

### 4.3 单文件加密流程

```
输入:
  plaintext_bytes    = final_jpeg_bytes (内存中)
  dek                = 案件对应的 DEK (从 SQLCipher 读取, 用 MK 解密)
  file_hash_hex      = Step 5 的输出

加密:
  iv                 = CSPRNG(12 bytes)               // 每次加密必须新 IV
  aad                = UTF8(file_hash_hex)             // 附加认证数据
  (ciphertext, tag)  = AES-256-GCM(dek, iv, plaintext_bytes, aad)

输出:
  encrypted_file     = iv (12B) || ciphertext || tag (16B)    // 单文件格式
  iv_hex             = hex(iv)
  auth_tag_hex       = hex(tag)
```

**AAD（Additional Authenticated Data）设计**：
```
AAD = file_hash_hex

作用: 将文件哈希绑定到密文
效果: 如果攻击者替换了加密文件（密文替换攻击），
      解密时 GCM 认证会失败，因为 AAD 不匹配
```

### 4.4 解密验证流程

```
输入:
  encrypted_file     = 磁盘上的 .evault 文件
  dek                = DEK（经 MK 解密）
  expected_hash_hex  = EvidenceItem.file_ref.file_hash_hex

解密:
  从文件读取: iv (前 12B), ciphertext (中间), tag (末 16B)
  aad                = UTF8(expected_hash_hex)
  plaintext_bytes    = AES-256-GCM-Decrypt(dek, iv, ciphertext, tag, aad)

  如果 GCM tag 验证失败 → 中止，报告 "TAMPERED"

完整性双重验证:
  actual_hash        = SHA-256(plaintext_bytes)
  assert actual_hash == expected_hash_hex     // 二次确认

  如果不一致 → 中止，报告 "INTEGRITY_MISMATCH"（理论上不应发生）
```

---

## 5. Hash 计算的时机与范围

### 5.1 哈希计算节点总览

| 时机 | 计算对象 | 输出字段 | 目的 |
|------|---------|---------|------|
| **Step 5** 拍照后立即 | `final_jpeg_bytes`（含水印的完整 JPEG） | `file_ref.file_hash_hex` | 文件内容指纹，全流程锚点 |
| **Step 7** 落盘时 | EvidenceItem 所有 `[H]` 字段（无 TSA） | `integrity.record_hash_hex`（临时） | 确保元数据完整 |
| **Step 11** TSA 返回后 | EvidenceItem 所有 `[H]` 字段（含 TSA） | `integrity.record_hash_hex`（最终） | 包含时间证明的完整记录哈希 |
| **AuditLog 每条** | 日志条目全字段 | `this_log_hash` | 操作审计链 |
| **导出时** | 每个文件 + manifest | `hash_manifest.json` | 导出包完整性 |

### 5.2 哈希范围精确定义

**file_hash_hex 的计算范围**：
```
输入 = 完整 JPEG 文件 bytes（包含 EXIF header + 可见水印像素 + JPEG 压缩数据）
不包含 = 文件系统元数据（创建时间、权限等）
算法 = SHA-256
```

**record_hash_hex 的计算范围**：
```
输入 = EvidenceItem 中所有标记 [H] 的字段（见 02-evidence-item-schema.md 第 4 节）
序列化 = JSON，键按路径字母序排列，separators=(',', ':')，ensure_ascii=False
算法 = SHA-256

注意：GPS 为 null 时，gps.* 字段不参与（避免 null 序列化歧义）
注意：timestamp_proof 为 pending 时，TSA 字段不参与
      为 granted 后重算，TSA 字段纳入
```

---

## 6. 防"拍完再改"的完整防线

```
时间线:
  T0                T1              T2            T3           T4
  快门按下          原始帧获取       水印注入       哈希计算      加密写盘
  │                │               │             │            │
  ├─── Step 1 ────┤── Step 2-3 ──┤── Step 4 ──┤── Step 5 ──┤── Step 6-7 ──→
  │                │               │             │            │
  环境锁定          内存中           内存中         哈希锁定      密文锁定
```

**6 层防篡改机制**：

| # | 防线 | 攻击场景 | 防御方式 |
|---|------|---------|---------|
| 1 | 无磁盘中间态 | 修改磁盘上的明文文件 | 全程内存操作，明文不落盘 |
| 2 | 哈希先于加密 | 加密后替换密文 | file_hash 在加密前计算，作为 GCM AAD 绑定到密文 |
| 3 | GCM 认证标签 | 篡改加密后的文件 | auth_tag 验证失败 → 解密中止 |
| 4 | 水印烧录像素 | 修改 EXIF 伪造时间 | 时间在像素中，改 EXIF 不影响水印，改水印则哈希变 |
| 5 | TSA 时间戳 | 事后伪造拍摄时间 | TSA 签发时间由 CA 机构保证，不可伪造 |
| 6 | AuditLog 哈希链 | 删除或修改操作记录 | 链式哈希，任何删改导致链断裂 |

**攻击者如果要"拍完再改"，需要同时**：
```
1. 突破 AES-256-GCM 加密（计算不可行）
2. 或获取 DEK 密钥（需要生物识别 + Keystore 硬件保护）
3. 即使获取 DEK 并修改文件，file_hash 不匹配
4. 即使修改 file_hash，record_hash 不匹配
5. 即使修改 record_hash，TSA 时间戳中的哈希不匹配
6. 即使以上全部绕过，AuditLog 哈希链断裂
7. TSA 时间戳由 CA 私钥签名，无法伪造
```

**结论**：在不攻破 TSA 机构私钥的前提下，篡改是不可行的。

---

## 7. 录音/录像的差异处理

| 差异点 | 照片 | 录音 | 录像 |
|-------|------|------|------|
| 捕获方式 | 单帧 | 流式（持续录制） | 流式（音视频） |
| 水印 | 可见像素水印 | 不可加可见水印 | 首尾帧加水印 |
| 哈希时机 | 拍照后立即 | 录制结束后 | 录制结束后 |
| 中间保护 | N/A | 加密分段缓冲 | 加密分段缓冲 |
| 文件格式 | JPEG | AAC (m4a 容器) | MP4 (H.264+AAC) |
| 元数据注入 | EXIF UserComment | M4A/MP4 metadata atom | MP4 metadata atom |

**录音/录像的流式加密方案**：
```
录制中每 N 秒（默认 30s）:
  1. 将当前缓冲区 flush 到临时加密分段文件
  2. 对分段计算哈希并记录
  3. 内存缓冲区清零

录制结束:
  1. 合并所有分段 → 完整文件 bytes（内存中）
  2. 计算整体 file_hash
  3. 验证分段哈希链
  4. 加密完整文件 → .evault
  5. 删除临时分段文件

目的：防止录制中 App 被杀时丢失全部数据
```
