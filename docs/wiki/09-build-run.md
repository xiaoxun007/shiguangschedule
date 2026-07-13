# 09 构建与运行

## 环境要求

| 项 | 要求 |
| --- | --- |
| Android Studio / AGP | AGP 9.2.1 |
| JDK | 21（`jvmToolchain(21)`） |
| Kotlin | 2.4.0 |
| compileSdk / targetSdk | 37 |
| minSdk | 26（Android 8.0） |
| Gradle | 通过 `gradle/wrapper/gradle-wrapper.properties` 指定 |

## Flavor 与产物

构建配置见 [app/build.gradle.kts](../../../app/build.gradle.kts)，`productFlavors` 维度 `version`：

| Flavor | applicationId | versionName 后缀 | 图标背景 | 定位 |
| --- | --- | --- | --- | --- |
| `dev` | `...shiguangschedule.dev` | `-dev` | 红色 | 开发者测试，可与正式版共存 |
| `prod` | `...shiguangschedule` | 无 | 蓝色 | 普通用户，开启安全验证 |

BuildConfig 开关差异详见 [01 项目概览](./01-overview.md#版本与-flavor)。

**ABI 分包**：启用 `splits.abi`，输出 `armeabi-v7a` / `arm64-v8a` / `x86_64` 三个独立 APK（`isUniversalApk = false`，排除 mips/armeabi/riscv64/x86）。

**Release 构建**：`isMinifyEnabled = true` + `isShrinkResources = true`，使用 `proguard-android-optimize.txt` 与 [proguard-rules.pro](../../../app/proguard-rules.pro)；签名复用 `debug` 签名配置（仓库未内置正式签名）。

**Room schema 导出**：`ksp { arg("room.schemaLocation", "$projectDir/schemas") }`，schema JSON 存于 [app/schemas/](../../../app/schemas/)。

## 常用构建命令

使用 Gradle Wrapper（`./gradlew`，Windows 用 `gradlew.bat`）。CI 环境加 `--no-daemon`。

```bash
# 清理
./gradlew clean

# Debug 构建（开发版）
./gradlew :app:assembleDevDebug

# Debug 构建（正式版）
./gradlew :app:assembleProdDebug

# Release 构建（开发版）
./gradlew :app:assembleDevRelease

# Release 构建（正式版，产物在 app/build/outputs/apk/prod/release/）
./gradlew :app:assembleProdRelease

# 安装到已连接设备
./gradlew :app:installDevDebug
./gradlew :app:installProdDebug

# 单元测试
./gradlew :app:testDevDebugUnitTest

# 仪器化测试（需连接设备/模拟器）
./gradlew :app:connectedDevDebugAndroidTest
```

> 产物路径：`app/build/outputs/apk/<flavor>/<buildType>/app-<flavor>-<buildType>-<abi>.apk`。

## 运行与调试

1. **首次启动**：`MyApplication.onCreate` 会清理 `cacheDir/share_temp` 临时文件、启动 `SyncManager`、协程拷贝 `assets/offline_repo` 到 `filesDir/repo/` 作为离线兜底。
2. **数据库初始化**：`MainAppDatabase` 首次创建时写入默认课表"我的课表"、默认配置（20 周）与 13 节默认时间段，置 `isInitialized = true`。
3. **同步启动**：`SyncManager` 监听 `isInitialized` 就绪后启动 `WidgetDataSynchronizer.syncFlow`，并 500ms debounce 监听 `dataUpdatedFlow` 触发通知/勿扰 Worker。
4. **起始页**：由 `AppSettings.startScreen` 决定（周课表 / 今日课表）。

## 权限说明

[AndroidManifest.xml](../../../app/src/main/AndroidManifest.xml) 声明的关键权限：

| 权限 | 用途 |
| --- | --- |
| `INTERNET` / `ACCESS_NETWORK_STATE` | 网络访问（Git 拉取、节假日 API、WebDAV、更新检查） |
| `SCHEDULE_EXACT_ALARM` | 课程提醒精确闹钟 |
| `POST_NOTIFICATIONS` | 课程提醒通知 |
| `ACCESS_NOTIFICATION_POLICY` | 勿扰/静音自动化（`setInterruptionFilter`/`ringerMode`） |
| `RECEIVE_BOOT_COMPLETED` | 开机后恢复闹钟（预留） |
| `POST_PROMOTED_NOTIFICATIONS` | Android 16 promoted ongoing 通知 |
| `READ_CALENDAR` / `WRITE_CALENDAR` | 系统日历同步与 ICS 导入 |

> 勿扰/静音自动化需用户在系统设置授予"勿扰权限"（`isNotificationPolicyAccessGranted`），未授予仅告警不阻断。精确闹钟需用户在系统设置授权（Android S+ `canScheduleExactAlarms()`）。

## 注册的组件

| 类型 | 组件 | 说明 |
| --- | --- | --- |
| Application | `.MyApplication` | `@HiltAndroidApp` |
| Activity | `.MainActivity` | `singleTask`，LAUNCHER |
| Receiver | `.service.CourseAlarmReceiver` | 课程提醒/勿扰闹钟 |
| Receiver | `.widget.tiny/compact/double_days/list_vertical.*NativeProvider` | 4 种小组件 |
| Provider | `androidx.core.content.FileProvider`（`${applicationId}.fileprovider`） | 文件分享 |
| Provider | `androidx.startup.InitializationProvider` | `tools:node="remove"` 移除，改由 Hilt 管理 |
