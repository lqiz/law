# EvidenceVault v1.0 — 安全模块规格

---

## 1. 安全威胁模型

| 威胁 | 场景 | 风险等级 | 攻击者能力 |
|------|------|---------|-----------|
| T1 设备丢失/被盗 | 手机遗失或被扣押 | 高 | 物理访问设备 |
| T2 旁人窥屏 | 公共场合查看证据 | 中 | 视觉观察 |
| T3 强迫解锁 | 被迫提供 PIN 或生物识别 | 高 | 物理胁迫 |
| T4 恶意软件 | 设备上安装了监控软件 | 中 | Root/越狱级别 |
| T5 取证工具提取 | 专业取证设备读取存储 | 高 | 专业取证工具 |
| T6 内存残留 | 解密后数据残留在内存 | 低 | 内存转储 |

---

## 2. App 锁（访问控制）

### 2.1 认证方式层级

```
┌─────────────────────────────────────────┐
│         认证方式优先级                     │
│                                         │
│  Level 1: 生物识别（最便捷）              │
│    iOS: Face ID / Touch ID              │
│    Android: 指纹 / 面部识别              │
│         │                               │
│         │ 失败或不可用                    │
│         ▼                               │
│  Level 2: 6 位 PIN（必须设置）            │
│    所有用户必须设置 PIN 作为底线认证       │
│         │                               │
│         │ 连续 5 次失败                   │
│         ▼                               │
│  Level 3: 冷却期                         │
│    等待 60 秒后可重试                     │
│         │                               │
│         │ 连续 10 次失败                  │
│         ▼                               │
│  Level 4: 紧急响应                       │
│    弹出选项: 继续等待 / 紧急销毁          │
│                                         │
└─────────────────────────────────────────┘
```

### 2.2 PIN 码规格

```
长度: 6 位数字（固定长度）
存储: 不存储 PIN 明文
验证:
  1. PIN → PBKDF2(PIN, salt, iterations=100000) → derived_key
  2. 用 derived_key 尝试解密一个"验证令牌"
  3. 解密成功 = PIN 正确

为什么不用更复杂的密码:
  - 目标用户是普通劳动者，非技术人员
  - 6 位 PIN + 冷却期 + 设备安全芯片 = 暴力破解不可行
  - 10^6 种组合 × 100000 PBKDF2 迭代 × 60 秒冷却 = 实际不可破解
```

### 2.3 生物识别集成

```dart
// 使用 local_auth 包
final isAvailable = await LocalAuthentication().canCheckBiometrics;
final didAuth = await LocalAuthentication().authenticate(
  localizedReason: '解锁 EvidenceVault',
  options: const AuthenticationOptions(
    stickyAuth: true,       // 切换应用后回来不需要重新认证
    biometricOnly: true,    // 仅生物识别，不回退到系统 PIN
  ),
);
```

**平台能力对照**：

| 能力 | iOS | Android |
|------|-----|---------|
| 生物识别 API | LocalAuthentication framework | BiometricPrompt API |
| 面部 | Face ID | 设备依赖（部分支持） |
| 指纹 | Touch ID | Fingerprint API |
| 密钥绑定 | kSecAccessControlBiometryCurrentSet | setUserAuthenticationRequired |
| 生物特征变更检测 | evaluatedPolicyDomainState | 新增指纹时 Keystore 密钥失效 |

### 2.4 锁定时机

| 事件 | 行为 |
|------|------|
| App 启动 | 始终要求认证 |
| 从后台恢复 > 5 分钟 | 要求重新认证 |
| 从后台恢复 ≤ 5 分钟 | 直接恢复（配置项可调） |
| 屏幕锁定 | 标记为需要认证 |
| 查看证据详情 | 不额外认证（App 已解锁即可） |
| 导出证据包 | 额外认证一次（高安全操作） |
| 紧急销毁 | 额外认证一次 + 文字确认 |
| 修改安全设置 | 额外认证一次 |

---

## 3. 数据加密体系

### 3.1 三层密钥完整流程

```
首次使用 App:
━━━━━━━━━━━━

  用户设置 PIN
       │
       ▼
  Step 1: 在 Keystore/Keychain 中生成 Master Key (MK)
          属性: AES-256, 不可导出, 需用户认证
       │
       ▼
  Step 2: 生成 salt (32 bytes random)
          存储到 t_app_config
       │
       ▼
  Step 3: derived_key = PBKDF2(PIN, salt, 100000, SHA-256)
          生成 verify_token = AES-GCM(MK, random_data)
          存储 encrypted_verify_token + verify_nonce 到 t_app_config
       │
       ▼
  App 初始化完成


创建新案件:
━━━━━━━━━━

  Step 1: 生成 DEK = CSPRNG(32 bytes)
       │
       ▼
  Step 2: encrypted_dek = AES-GCM(MK, DEK)
       │
       ▼
  Step 3: 存储到 t_key_store:
          key_id, encrypted_key_base64, case_id
       │
       ▼
  Step 4: 清零内存中的 DEK 明文


拍照/录音时:
━━━━━━━━━━━

  Step 1: 从 t_key_store 读取 encrypted_dek
       │
       ▼
  Step 2: DEK = AES-GCM-Decrypt(MK, encrypted_dek)
          (此操作触发 Keystore 的用户认证检查)
       │
       ▼
  Step 3: 使用 DEK 加密证据文件
       │
       ▼
  Step 4: 清零内存中的 DEK 明文
```

### 3.2 密钥轮换（v1.0 简化方案）

```
v1.0 不主动轮换 DEK（MVP 简化）。

触发被动轮换的场景:
  - 用户修改 PIN → 重新生成 MK → 用新 MK 重新加密所有 DEK
  - 生物特征变更（如录入新指纹）→ iOS: 密钥可能失效 → 需要 PIN 恢复

v2.0 计划:
  - 定期轮换 DEK（每 90 天）
  - 旧 DEK 保留到所有关联证据被导出或删除
```

---

## 4. 防截屏/录屏

### 4.1 实现方案

**Android**:
```kotlin
// 在敏感 Activity 中
window.setFlags(
    WindowManager.LayoutParams.FLAG_SECURE,
    WindowManager.LayoutParams.FLAG_SECURE
)
// 效果: 截屏/录屏时该页面显示为黑色
```

**iOS**:
```swift
// 方式 1: 使用 UITextField 的安全文本输入特性
let secureField = UITextField()
secureField.isSecureTextEntry = true
// 将 secureField 的 layer 作为容器

// 方式 2: 监听截屏通知
NotificationCenter.default.addObserver(
    forName: UIApplication.userDidTakeScreenshotNotification,
    ...) { _ in
    // 截屏后提醒用户（无法阻止，只能事后提醒）
    // 记录 AuditLog
}

// 方式 3: 监听录屏状态
if UIScreen.main.isCaptured {
    // 录屏中 → 隐藏敏感内容
}
```

**Flutter MethodChannel 封装**:
```dart
class ScreenGuard {
  static const _channel = MethodChannel('ev_screen_guard');

  /// 启用防截屏（Android 立即生效，iOS 尽力而为）
  static Future<void> enable() => _channel.invokeMethod('enable');

  /// 禁用防截屏
  static Future<void> disable() => _channel.invokeMethod('disable');
}
```

### 4.2 应用范围

```dart
// 在需要防截屏的页面 initState 中:
@override
void initState() {
  super.initState();
  ScreenGuard.enable();
}

@override
void dispose() {
  ScreenGuard.disable();
  super.dispose();
}
```

---

## 5. 紧急销毁

### 5.1 设计意图

```
场景: 用户处于被胁迫状态（如被不当扣押手机），
      需要快速销毁敏感数据防止泄露。

⚠️ 法律风险提示:
   - 在诉讼进行中销毁证据可能构成"毁灭证据"
   - App 必须在功能入口处明确警告此风险
   - 仅建议在人身安全受威胁时使用
   - AuditLog 的销毁记录本身也会被销毁
```

### 5.2 销毁流程

```
用户路径:
  设置 → 安全设置 → 紧急销毁

  ┌─────────────────────────────────────┐
  │ ⚠️ 紧急销毁                         │
  │                                     │
  │ 此操作将永久删除所有数据，            │
  │ 包括全部案件、证据和操作记录。         │
  │ 删除后不可恢复。                      │
  │                                     │
  │ ⚠ 在诉讼中销毁证据可能违反法律，      │
  │   请仅在人身安全受威胁时使用。         │
  │                                     │
  │ 请输入确认文字:                       │
  │ ┌─────────────────────────────────┐ │
  │ │ 输入 "删除全部数据"              │ │
  │ └─────────────────────────────────┘ │
  │                                     │
  │ 然后验证身份:                        │
  │ [PIN / 生物识别]                     │
  │                                     │
  │    ┌──────────┐  ┌──────────┐      │
  │    │   取消    │  │ 确认销毁  │      │
  │    └──────────┘  └──────────┘      │
  └─────────────────────────────────────┘
```

### 5.3 技术执行

```
确认销毁后执行顺序（不可中断）:
  1. 删除 Keystore/Keychain 中的 Master Key
     → MK 销毁后，所有 DEK 无法解密 → 所有证据文件立即不可读
     → 即使后续步骤失败，数据已不可恢复

  2. 清除 SQLCipher 数据库
     → DROP ALL TABLES
     → VACUUM（重写数据库文件）

  3. 覆写所有 .evault 文件
     → 每个文件用随机数据覆写 3 次
     → 然后删除文件

  4. 删除数据库文件
     → 随机数据覆写 3 次
     → 删除

  5. 清除 SharedPreferences / UserDefaults

  6. 重置 App 状态 → 跳转到 PinSetupScreen（如同全新安装）

估算耗时:
  100 条证据 × 平均 5MB = 500MB
  3 次覆写 × 500MB ÷ 200MB/s = ~7.5 秒
  总计 ≈ 10-15 秒
```

### 5.4 隐蔽紧急销毁（v1.0 可选）

```
方案: 胁迫 PIN

  用户设置一个"胁迫 PIN"（与正常 PIN 不同的另一个 6 位数）
  输入胁迫 PIN → App 表面正常解锁 → 后台静默执行销毁
  → 显示空的案件列表（看起来像刚装的 App）

实现:
  - 设置中启用"紧急备用 PIN"
  - LockScreen 验证时:
    if (pin == normal_pin) → 正常解锁
    if (pin == duress_pin) → 静默销毁 + 显示空状态

⚠️ v1.0 评估: 作为可选功能，默认关闭。
   存在法律灰色地带（可能被认为是帮助毁灭证据的工具设计），
   需法律顾问评估后决定是否上线。
```

---

## 6. 文件系统安全

### 6.1 存储路径安全

```
所有数据存储在 App 沙箱内:

iOS:
  NSDocumentDirectory + NSFileProtectionComplete
  → 设备锁屏后文件自动加密（硬件级）
  → 即使越狱也无法在锁屏状态读取

Android:
  getFilesDir() (内部存储)
  → 其他 App 无法访问（除非 root）
  → 配合 SQLCipher 和文件级加密 = 双重保护

不使用外部存储 / SD 卡（不可控）
```

### 6.2 临时文件管理

```
解密后的临时文件（查看证据时）:
  路径: {temp_dir}/ev_temp_{uuid}.{ext}
  生命周期:
    创建 → 展示给用户 → 用户退出页面 → 覆写 → 删除
  最大存活时间: 5 分钟（定时器强制清除）
  App 启动时: 扫描并清除所有 ev_temp_* 文件
```

---

## 7. 网络安全

```
v1.0 网络通信仅用于 TSA 时间戳:

  传输安全:
    - HTTPS (TLS 1.2+)
    - 证书固定 (Certificate Pinning) → 防中间人攻击
    - 仅发送 32 字节哈希值 + nonce + ASN.1 请求头

  请求内容:
    ✓ 发送: SHA-256 哈希（32 bytes）
    ✗ 不发送: 原始文件、文件名、用户信息、设备信息、GPS

  DNS 安全:
    - 使用 DoH (DNS-over-HTTPS) 防止 DNS 劫持
    - 备用: 直接 IP 连接（硬编码 TSA 服务器 IP 作为 fallback）
```

---

## 8. 完整安全检查清单

| # | 检查项 | 实现方案 | 防御威胁 |
|---|--------|---------|---------|
| 1 | App 访问控制 | PIN + 生物识别 | T1, T2 |
| 2 | 数据库加密 | SQLCipher (AES-256-CBC) | T1, T4, T5 |
| 3 | 文件加密 | AES-256-GCM per file | T1, T4, T5 |
| 4 | 密钥硬件保护 | Keystore / Secure Enclave | T4, T5 |
| 5 | 防截屏 | FLAG_SECURE / screen capture detection | T2 |
| 6 | 内存清零 | 使用后立即清零敏感 bytes | T6 |
| 7 | 临时文件管理 | 超时清除 + 启动时扫描清除 | T1, T4 |
| 8 | 网络安全 | TLS 1.2+ / Certificate Pinning | 中间人攻击 |
| 9 | 紧急销毁 | MK 删除 + 文件覆写 + DB 清除 | T3 |
| 10 | 后台超时锁定 | 5 分钟无操作要求重新认证 | T1, T2 |
| 11 | 错误尝试限制 | 5 次冷却 / 10 次紧急响应 | 暴力破解 |
| 12 | iOS 文件保护 | NSFileProtectionComplete | T1, T5 |
