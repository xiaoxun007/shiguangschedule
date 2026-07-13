# 08 依赖关系

依赖统一由 Version Catalog [gradle/libs.versions.toml](../../../gradle/libs.versions.toml) 管理。下表按用途分组，列出关键库与作用。

## 构建插件

| 插件 | 版本 | 作用 |
| --- | --- | --- |
| `com.android.application` | AGP 9.2.1 | Android 应用构建 |
| `org.jetbrains.kotlin.plugin.compose` | Kotlin 2.4.0 | Compose 编译器插件 |
| `org.jetbrains.kotlin.plugin.serialization` | 2.4.0 | kotlinx.serialization 编译器插件 |
| `com.google.devtools.ksp` | 2.3.9 | Room/Hilt 符号处理 |
| `com.google.dagger.hilt.android` | 2.60.1 | Hilt 依赖注入 |
| `com.mikepenz.aboutlibraries.plugin.android` | 15.0.3 | 开源协议元数据生成 |
| `com.squareup.wire` | 6.4.5 | Protobuf → Kotlin 代码生成 |

## 运行时依赖

### UI / Compose

| 库 | 作用 |
| --- | --- |
| `androidx.compose:compose-bom`（2026.06.01） | Compose BOM 统一版本 |
| `androidx.compose.material3:material3` | Material 3 组件 |
| `androidx.compose.material:material-icons-extended` | 扩展图标 |
| `androidx.compose.foundation:foundation` | 基础能力 |
| `androidx.compose.animation:animation`（1.11.4） | 动画 |
| `androidx.activity:activity-compose`（1.13.0） | Activity Compose 集成 |
| `androidx.appcompat:appcompat`（1.7.1） | 语言切换等兼容能力 |

### 导航 / 生命周期

| 库 | 作用 |
| --- | --- |
| `androidx.navigation3:navigation3-runtime` / `navigation3-ui`（1.1.4） | Navigation3 类型安全导航 |
| `androidx.lifecycle:lifecycle-viewmodel-navigation3` | ViewModel 与导航集成 |
| `androidx.lifecycle:lifecycle-runtime-ktx`（2.11.0） | 生命周期 |
| `androidx.core:core-ktx`（1.19.0） | Core KTX |

### 依赖注入

| 库 | 作用 |
| --- | --- |
| `com.google.dagger:hilt-android`（2.60.1） | Hilt 运行时 |
| `androidx.hilt:hilt-navigation-compose`（1.4.0） | Compose `hiltViewModel()` |
| `androidx.hilt:hilt-work`（1.4.0） | WorkManager 与 Hilt 集成 |
| `javax.inject:javax.inject`（1） | JSR-330 注解 |

### 存储 / 序列化

| 库 | 作用 |
| --- | --- |
| `androidx.room:room-runtime/ktx/compiler`（2.8.4） | Room 数据库 |
| `androidx.datastore:datastore-core/preferences`（1.2.1） | DataStore 偏好与 Proto |
| `org.jetbrains.kotlinx:kotlinx-serialization-json`（1.11.0） | JSON 序列化 |
| `org.jetbrains.kotlinx:kotlinx-serialization-cbor`（1.11.0） | CBOR 二进制序列化（备份） |
| `com.squareup.wire:wire-runtime`（6.4.5） | Protobuf 运行时 |

### 后台任务

| 库 | 作用 |
| --- | --- |
| `androidx.work:work-runtime-ktx`（2.11.2） | WorkManager 后台调度 |

### 网络

| 库 | 作用 |
| --- | --- |
| `io.ktor:ktor-client-core/okhttp/content-negotiation/logging/auth`（3.5.1） | Ktor 客户端（WebDAV、节假日 API） |
| `io.ktor:ktor-serialization-kotlinx-json` | Ktor JSON 协商 |
| `com.squareup.okhttp3:okhttp`（5.4.0） | OkHttp（更新检查、WebView POST 拦截） |
| `com.squareup.okhttp3:logging-interceptor`（debugImplementation） | 日志拦截器（仅 debug） |

### Git

| 库 | 作用 |
| --- | --- |
| `org.eclipse.jgit:org.eclipse.jgit`（5.13.5） | 拉取适配脚本仓库（排除 httpclient/JavaEWAH/slf4j-api） |
| `org.slf4j:slf4j-api`（2.0.18） + `slf4j-android`（1.7.36） | JGit 日志门面与 Android 绑定 |

### 图片 / 其他

| 库 | 作用 |
| --- | --- |
| `io.coil-kt.coil3:coil-compose`（3.5.0） | Coil 3 图片加载（背景图等） |
| `com.mikepenz:aboutlibraries-compose-m3`（15.0.3） | 开源协议展示页 |

## 依赖关系概览

```
UI 层 ──▶ ViewModel ──▶ Repository ──▶ DAO / DataStore
            │              │                  │
            │              ├──▶ Hilt (注入) ◀──┤
            │              ├──▶ Room (主库/快照库)
            │              ├──▶ DataStore (Preferences/Proto)
            │              ├──▶ Ktor (WebDAV/节假日)
            │              ├──▶ OkHttp (更新/WebView 拦截)
            │              └──▶ JGit (GitUpdater 仓库拉取)
            │
            ├──▶ Navigation3 (路由)
            ├──▶ Compose/Material3 (渲染)
            └──▶ Coil (图片)

后台层 ──▶ WorkManager (周期/一次性任务)
       ──▶ AlarmManager (精确闹钟, 经 CourseAlarmReceiver)

小组件 ──▶ RemoteViews (Renderer 渲染)
       ──▶ WidgetSnapshot (Wire Proto)
       ──▶ WidgetDatabase (快照, 由 WidgetDataSynchronizer 同步)
```

## 测试依赖

| 库 | 作用 |
| --- | --- |
| `junit:junit`（4.13.2） | 单元测试 |
| `androidx.test.ext:junit`（1.3.0） | AndroidX JUnit 扩展 |
| `androidx.test.espresso:espresso-core`（3.7.0） | UI 测试 |
| `androidx.compose.ui:ui-test-junit4` | Compose UI 测试 |
