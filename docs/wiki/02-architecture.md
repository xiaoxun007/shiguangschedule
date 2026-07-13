# 02 整体架构

## 分层架构

项目为单 module Android 应用，采用 **MVVM + 单向数据流**，整体分为以下几层：

```
┌─────────────────────────────────────────────────────────┐
│                       App 入口层                          │
│  MyApplication (Hilt/WorkManager 初始化/离线仓库拷贝)      │
│  MainActivity (Compose 宿主 / 起始页路由)                 │
│  Navigation.kt (Destination 路由表 + 转场动画)            │
├─────────────────────────────────────────────────────────┤
│                        UI 层 (Compose)                   │
│  ui/schedule  ui/today  ui/settings(子包)  ui/schoolsel  │
│  ui/components(通用组件)  ui/theme(主题)                  │
│  各 Screen Composable + 对应 ViewModel (Hilt)            │
├─────────────────────────────────────────────────────────┤
│                     数据 / 业务层                         │
│  data/repository (SSOT 仓库, @Singleton)                 │
│  data/sync (SyncManager / WidgetDataSynchronizer)        │
│  data/model (业务模型 + 存储键)                           │
├─────────────────────────────────────────────────────────┤
│                       存储层                              │
│  data/db/main  (MainAppDatabase: 课表/课程/时间段/配置)   │
│  data/db/widget(WidgetDatabase: 渲染快照)                │
│  DataStore (Preferences + Proto 双轨)                    │
├─────────────────────────────────────────────────────────┤
│              小组件 / 服务 / 工具 / DI                     │
│  widget(4种小组件 Provider/Renderer)                     │
│  service(通知/勿扰/闹钟 Worker+Receiver)                 │
│  tool(Git更新/ICS导出/日历/分享/应用更新)                 │
│  data/di(Hilt Module: Database/DataStore)               │
└─────────────────────────────────────────────────────────┘
```

## 关键架构特征

1. **单一事实来源（SSOT）**：`MainAppDatabase` 为课表数据的唯一权威来源；`WidgetDatabase` 仅为渲染优化的反规范化快照，由 `WidgetDataSynchronizer` 单向投影，不回写。
2. **响应式驱动**：UI 与小组件均订阅 Room/DataStore 的 `Flow`，数据变化自动传播；`WidgetDataSynchronizer.syncFlow` 监听主库三路 Flow 合并触发同步。
3. **事件总线闭环**：`WidgetRepository.dataUpdatedFlow`（CONFLATED Channel）作为内部事件总线，`SyncManager` 以 500ms debounce 消费它，触发通知/勿扰 Worker 重排闹钟，形成"主库变化 → 快照刷新 → 闹钟重排"闭环。
4. **启动闸门**：`MainAppDatabase.isInitialized`（StateFlow）作为同步启动闸门，确保种子数据写入后才启动 `WidgetDataSynchronizer`。
5. **瘦 Provider + 纯函数 Renderer**：小组件 Provider 仅转发生命周期回调，渲染逻辑集中在无状态 `Renderer.render()`，四种小组件共享同一份 `WidgetSnapshot`。

## 数据流概要

```
用户操作 / 教务导入
      │
      ▼
Repository (写) ──▶ MainAppDatabase (Room Flow)
      │                       │
      │                       ▼  (combine 三路 Flow)
      │              WidgetDataSynchronizer.syncFlow
      │                       │ (展平 + 7天物化)
      │                       ▼
      │              WidgetDatabase (快照)
      │                       │
      │                       ▼ dataUpdatedFlow
      │              SyncManager (debounce 500ms)
      │                       │
      │          ┌────────────┴────────────┐
      │          ▼                         ▼
      │  CourseNotificationWorker   DndSchedulerWorker
      │  (注册精确闹钟 50010-50110)  (注册闹钟 50001/50002)
      │          │                         │
      │          └────────────┬────────────┘
      │                       ▼ AlarmManager 到点
      │              CourseAlarmReceiver
      │              (发通知 / toggle 勿扰)
      │
      ▼ (UI Flow 订阅)
Compose Screen (ViewModel)         Widget Renderer
                                   (updateAllWidgets)
```

## 目录结构

仅列出与源码理解相关的关键目录，省略资源/构建产物。

```
shiguangschedule/
├── app/
│   ├── build.gradle.kts                 # 模块构建配置（Flavor/依赖/ Wire）
│   ├── proguard-rules.pro
│   ├── schemas/                          # Room 导出的 schema JSON (v3/4/5)
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml       # 权限、Activity、4 个小组件 Receiver
│       │   ├── assets/
│       │   │   ├── git_repos.json        # 内置仓库源清单
│       │   │   └── offline_repo/         # 离线兜底索引 + school.js
│       │   ├── java/com/xingheyuzhuan/shiguangschedule/
│       │   │   ├── MyApplication.kt      # Application（Hilt/同步/离线仓库）
│       │   │   ├── MainActivity.kt       # Compose 宿主
│       │   │   ├── Navigation.kt         # Destination 路由 + 转场动画
│       │   │   ├── navigation/AppChannels.kt
│       │   │   ├── data/
│       │   │   │   ├── api/              # 节假日 API / WebDAV 客户端
│       │   │   │   ├── db/main/          # 主数据库实体/DAO/迁移
│       │   │   │   ├── db/widget/        # 小组件快照数据库
│       │   │   │   ├── di/               # Hilt Module (Database/DataStore)
│       │   │   │   ├── model/            # 业务模型与存储键
│       │   │   │   ├── repository/       # SSOT 仓库
│       │   │   │   └── sync/             # SyncManager / WidgetDataSynchronizer
│       │   │   ├── service/              # 通知/勿扰 Worker、闹钟 Receiver
│       │   │   ├── tool/                 # Git/ICS/日历/分享/更新工具
│       │   │   ├── ui/
│       │   │   │   ├── components/       # 通用 Compose 组件
│       │   │   │   ├── schedule/         # 周课表
│       │   │   │   ├── today/            # 今日课表
│       │   │   │   ├── settings/         # 设置（多个子包）
│       │   │   │   ├── schoolselection/  # 学校选择 + WebView 适配
│       │   │   │   └── theme/            # 颜色/主题/字体
│       │   │   └── widget/               # 4 种小组件 + 调度
│       │   ├── proto/                    # Wire proto 定义
│       │   └── res/                      # 布局/ drawable / xml / values
│       └── dev/ , release/ , test/       # Flavor 与测试源集
├── gradle/libs.versions.toml            # Version Catalog
├── build.gradle.kts                     # 根构建文件
├── settings.gradle.kts
└── README.md
```

> UI 层的设置子包按功能拆分：`time / backup / themesettings / coursemanagement / update / coursetables / conversion / style / notification / additional / course / contribution / quickactions`，详见 [04 UI 层](./04-ui-layer.md)。
