# EvidenceVault v1.0 — Flutter 项目架构与依赖选型

---

## 1. 项目目录结构

```
evidence_vault/
├── android/                            # Android 原生层
│   └── app/src/main/kotlin/
│       └── com/evidencevault/
│           ├── CameraPlugin.kt         # 原生相机控制
│           ├── AudioRecorderPlugin.kt  # 原生录音控制
│           └── KeystorePlugin.kt       # Android Keystore 操作
├── ios/                                # iOS 原生层
│   └── Runner/
│       ├── CameraPlugin.swift          # AVCapturePhoto 控制
│       ├── AudioRecorderPlugin.swift   # AVAudioEngine 控制
│       └── KeychainPlugin.swift        # iOS Keychain 操作
├── lib/
│   ├── main.dart                       # 入口
│   ├── app.dart                        # MaterialApp + 路由配置
│   │
│   ├── core/                           # 核心引擎层（与 UI 无关）
│   │   ├── crypto/
│   │   │   ├── hash_engine.dart        # S01: SHA-256 计算
│   │   │   ├── aes_engine.dart         # S04/S05: AES-256-GCM 加解密
│   │   │   └── key_manager.dart        # 三层密钥管理
│   │   ├── timestamp/
│   │   │   ├── tsa_client.dart         # S03: RFC 3161 TSA 客户端
│   │   │   ├── hash_proof_manager.dart # 多通道调度 + 重试
│   │   │   └── asn1_builder.dart       # ASN.1 请求/响应解析
│   │   ├── evidence/
│   │   │   ├── evidence_builder.dart   # S06: 组装 EvidenceRecord
│   │   │   ├── evidence_verifier.dart  # S07: 全链路校验
│   │   │   └── integrity_hasher.dart   # record_hash 计算
│   │   ├── audit/
│   │   │   └── audit_logger.dart       # S10: 链式审计日志
│   │   ├── export/
│   │   │   ├── package_builder.dart    # S08: 证据包生成
│   │   │   ├── pdf_generator.dart      # PDF 报告生成
│   │   │   └── verify_script_gen.dart  # verify.sh/bat 生成
│   │   └── wipe/
│   │       └── secure_wiper.dart       # S09: 安全擦除
│   │
│   ├── data/                           # 数据层
│   │   ├── database/
│   │   │   ├── app_database.dart       # SQLCipher 初始化 + 迁移
│   │   │   ├── dao/
│   │   │   │   ├── case_dao.dart
│   │   │   │   ├── evidence_dao.dart
│   │   │   │   ├── hash_proof_dao.dart
│   │   │   │   ├── audio_segment_dao.dart
│   │   │   │   ├── audit_log_dao.dart
│   │   │   │   └── export_record_dao.dart
│   │   │   └── migrations/
│   │   │       └── migration_v1.dart
│   │   ├── models/                     # 数据模型（与 DB 表对应）
│   │   │   ├── case_model.dart
│   │   │   ├── evidence_item_model.dart
│   │   │   ├── hash_proof_model.dart
│   │   │   ├── audio_segment_model.dart
│   │   │   └── audit_log_model.dart
│   │   └── repositories/              # 仓储层（聚合 DAO 操作）
│   │       ├── case_repository.dart
│   │       ├── evidence_repository.dart
│   │       └── export_repository.dart
│   │
│   ├── services/                       # 业务服务层
│   │   ├── capture_service.dart        # 采集编排（协调相机/录音 + 固化流水线）
│   │   ├── seal_service.dart           # 固化编排（哈希 → 加密 → TSA → 组装）
│   │   ├── ntp_service.dart            # NTP 时间同步
│   │   ├── device_meta_service.dart    # S02: 设备元数据采集
│   │   └── auth_service.dart           # App 解锁（PIN + 生物识别）
│   │
│   ├── features/                       # UI 功能模块（按页面组织）
│   │   ├── auth/                       # 解锁 / 设置 PIN
│   │   │   ├── lock_screen.dart
│   │   │   ├── pin_setup_screen.dart
│   │   │   └── auth_provider.dart
│   │   ├── home/                       # 首页（案件列表）
│   │   │   ├── home_screen.dart
│   │   │   └── home_provider.dart
│   │   ├── case_detail/               # 案件详情（证据列表 + 时间线）
│   │   │   ├── case_detail_screen.dart
│   │   │   ├── evidence_timeline.dart
│   │   │   └── case_detail_provider.dart
│   │   ├── capture/                   # 采集（相机 / 录音 / 导入）
│   │   │   ├── camera_screen.dart
│   │   │   ├── audio_recorder_screen.dart
│   │   │   ├── import_screen.dart
│   │   │   └── capture_provider.dart
│   │   ├── evidence_detail/           # 证据详情查看
│   │   │   ├── evidence_detail_screen.dart
│   │   │   ├── photo_viewer.dart
│   │   │   ├── audio_player.dart
│   │   │   └── evidence_detail_provider.dart
│   │   ├── export/                    # 导出
│   │   │   ├── export_screen.dart
│   │   │   └── export_provider.dart
│   │   ├── guide/                     # 取证引导
│   │   │   ├── guide_list_screen.dart
│   │   │   └── guide_detail_screen.dart
│   │   └── settings/                  # 设置
│   │       ├── settings_screen.dart
│   │       └── security_settings_screen.dart
│   │
│   ├── shared/                        # 共享 UI 组件
│   │   ├── widgets/
│   │   │   ├── evidence_card.dart
│   │   │   ├── status_badge.dart
│   │   │   ├── watermark_overlay.dart
│   │   │   └── seal_progress.dart
│   │   └── theme/
│   │       └── app_theme.dart
│   │
│   └── utils/                         # 工具函数
│       ├── constants.dart
│       ├── extensions.dart
│       └── formatters.dart
│
├── test/                              # 测试
│   ├── core/                          # 核心引擎单元测试
│   │   ├── hash_engine_test.dart
│   │   ├── aes_engine_test.dart
│   │   ├── integrity_hasher_test.dart
│   │   └── tsa_client_test.dart
│   ├── data/                          # DAO 测试
│   └── integration/                   # 集成测试
│       ├── seal_pipeline_test.dart
│       └── export_pipeline_test.dart
│
├── pubspec.yaml
└── analysis_options.yaml
```

---

## 2. 分层架构

```
┌──────────────────────────────────────────────┐
│              Features (UI)                    │
│         Screen + Provider (Riverpod)         │
├──────────────────────────────────────────────┤
│              Services                         │
│     业务编排（CaptureService, SealService）    │
├──────────────────────────────────────────────┤
│            Core Engine                        │
│   纯逻辑（Hash, AES, TSA, Audit, Export）     │
│   ← 此层可独立于 Flutter 运行单元测试 →       │
├──────────────────────────────────────────────┤
│         Data (Repository + DAO)              │
│         SQLCipher + File System              │
├──────────────────────────────────────────────┤
│       Platform Channels (原生插件)            │
│    Camera / AudioRecorder / Keystore         │
└──────────────────────────────────────────────┘

依赖规则:
  Features → Services → Core + Data
  Core 不依赖 Data（通过接口注入）
  Data 不依赖 Features
  Platform Channels 被 Services 层调用
```

---

## 3. 依赖选型

### 3.1 核心依赖

| 依赖包 | 版本 | 用途 | 为什么选它 |
|--------|------|------|-----------|
| `flutter_riverpod` | ^2.x | 状态管理 | 类型安全、可测试、provider 自动销毁 |
| `crypto` | ^3.x | SHA-256 哈希 | Dart 官方维护，纯 Dart |
| `pointycastle` | ^3.x | AES-256-GCM + RSA | 纯 Dart 密码学库，跨平台一致 |
| `sqflite_sqlcipher` | ^3.x | SQLCipher 数据库 | SQLCipher 加密 + sqflite API |
| `path_provider` | ^2.x | 获取沙箱路径 | Flutter 官方插件 |
| `uuid` | ^4.x | UUIDv4 生成 | 证据/案件/日志 ID |

### 3.2 采集相关

| 依赖包 | 版本 | 用途 | 备注 |
|--------|------|------|------|
| `camera` | ^0.11.x | 相机预览 + 拍照 | Flutter 官方插件；分段控制需自封装 MethodChannel |
| `file_picker` | ^8.x | 文件/截图导入 | 系统文件选择器 |
| `image` | ^4.x | 水印注入（像素操作） | 纯 Dart 图像处理 |

### 3.3 设备信息

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `device_info_plus` | ^10.x | 设备型号、OS 版本 |
| `geolocator` | ^12.x | GPS 定位 |
| `connectivity_plus` | ^6.x | 网络状态检测 |
| `battery_plus` | ^6.x | 电量读取 |

### 3.4 安全相关

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `flutter_secure_storage` | ^9.x | Keychain/Keystore 简单读写 |
| `local_auth` | ^2.x | 生物识别（指纹/面容） |

### 3.5 UI 与导出

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `go_router` | ^14.x | 声明式路由 |
| `pdf` | ^3.x | 纯 Dart PDF 生成 |
| `archive` | ^3.x | ZIP 打包 |
| `share_plus` | ^9.x | 系统分享（AirDrop 等） |
| `just_audio` | ^0.9.x | 录音回放 |
| `intl` | ^0.19.x | 日期/数字国际化格式 |

### 3.6 自封装原生插件（MethodChannel）

| 插件 | 平台 | 用途 | 为什么不用现有包 |
|------|------|------|----------------|
| `ev_camera` | iOS + Android | 原始帧获取 + 分段控制 | 需要获取原始 bytes，不经过系统相册 |
| `ev_audio_recorder` | iOS + Android | PCM 流 + 精确分段 | 需要精确控制 flush 时机和分段边界 |
| `ev_keystore` | iOS + Android | MK 生成 + DEK 加解密 | flutter_secure_storage 不支持 GCM |
| `ev_ntp` | iOS + Android | NTP 时间同步 | 需要缓存 offset 供离线使用 |
| `ev_screen_guard` | iOS + Android | 防截屏/录屏 | FLAG_SECURE / UIScreen capturedDidChange |

---

## 4. pubspec.yaml 核心部分

```yaml
name: evidence_vault
description: 职场证据固化工具
version: 1.0.0+1
publish_to: none

environment:
  sdk: '>=3.2.0 <4.0.0'
  flutter: '>=3.16.0'

dependencies:
  flutter:
    sdk: flutter

  # 状态管理
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.3.0

  # 路由
  go_router: ^14.0.0

  # 密码学
  crypto: ^3.0.3
  pointycastle: ^3.9.0

  # 数据库
  sqflite_sqlcipher: ^3.1.0
  path_provider: ^2.1.0

  # 设备信息
  device_info_plus: ^10.1.0
  geolocator: ^12.0.0
  connectivity_plus: ^6.0.0
  battery_plus: ^6.0.0

  # 安全
  flutter_secure_storage: ^9.2.0
  local_auth: ^2.2.0

  # 采集
  camera: ^0.11.0
  file_picker: ^8.0.0
  image: ^4.2.0

  # 导出
  pdf: ^3.11.0
  archive: ^3.6.0
  share_plus: ^9.0.0

  # UI 与工具
  intl: ^0.19.0
  uuid: ^4.4.0
  just_audio: ^0.9.39

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0
  riverpod_generator: ^2.4.0
  build_runner: ^2.4.0
  mockito: ^5.4.0
  sqflite_common_ffi: ^2.3.0  # 桌面端测试用
```

---

## 5. 关键实现约束

### 5.1 线程模型

```
Main Isolate (UI)
  ├── UI 渲染
  ├── Provider 状态管理
  └── 轻量级操作

Background Isolate (compute)
  ├── SHA-256 哈希计算（大文件）
  ├── AES-256-GCM 加解密
  ├── PDF 生成
  ├── ZIP 打包
  └── 证据链完整性校验

Platform Thread (MethodChannel)
  ├── 相机原始帧获取
  ├── 录音 PCM 流
  └── Keystore/Keychain 操作

规则:
  - 超过 10ms 的计算必须放到 Background Isolate
  - 哈希和加密通过 Dart 的 Isolate.run() 执行
  - 相机和录音通过 EventChannel 实时推送数据
```

### 5.2 错误处理

```dart
// 统一错误类型
sealed class EVError {
  const EVError(this.code, this.message);
  final String code;
  final String message;
}

class CaptureError extends EVError { ... }    // 采集失败
class SealError extends EVError { ... }       // 固化失败
class CryptoError extends EVError { ... }     // 加解密失败
class TSAError extends EVError { ... }        // TSA 请求失败
class IntegrityError extends EVError { ... }  // 完整性校验失败
class StorageError extends EVError { ... }    // 存储读写失败
class ExportError extends EVError { ... }     // 导出失败
```

### 5.3 测试策略

```
core/ 层:
  - 100% 单元测试覆盖
  - 纯 Dart，不依赖 Flutter，可在 CI 上直接 dart test
  - 使用已知向量验证 SHA-256 / AES-GCM 正确性

data/ 层:
  - DAO 使用 sqflite_common_ffi 在桌面端测试
  - Repository 使用 mock DAO

services/ 层:
  - 使用 mock core + mock data 集成测试
  - 重点测试固化流水线的完整流程

features/ 层:
  - Widget 测试关键交互（拍照按钮 → 固化进度 → 完成提示）
  - 不追求 UI 层高覆盖率（MVP 优先）
```
