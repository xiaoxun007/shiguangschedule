# 10 关键类索引

按模块分组的关键类与函数速查，便于定位。文件路径均为相对仓库根。

## App 入口

| 类/函数 | 文件 | 职责 |
| --- | --- | --- |
| `MyApplication` | [MyApplication.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/MyApplication.kt) | Application，Hilt/WorkManager 初始化、启动同步、拷贝离线仓库 |
| `MainActivity` | [MainActivity.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/MainActivity.kt) | Compose 宿主，起始页路由 |
| `AppNavigation` / `ScreenContent` | [Navigation.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/Navigation.kt) | Destination 路由表与转场动画分发 |
| `Destination` / `ShiguangNavMetadata` | [Navigation.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/Navigation.kt) | 类型安全导航目的地定义 |
| `AddEditCourseChannel` / `PresetCourseData` | [AppChannels.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/navigation/AppChannels.kt) | 跨页面一次性数据传递 |

## 数据库（data/db/main）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `MainAppDatabase` | [MainAppDatabase.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/MainAppDatabase.kt) | 主库（v5），`isInitialized` 启动闸门 |
| `CourseTable` / `Course` / `CourseWeek` / `TimeSlot` / `CourseTableConfig` | 同目录 | Room 实体 |
| `*Dao` | 同目录 | 各实体 DAO |
| `MIGRATION_1_2` / `MIGRATION_2_3` / `ALL_MIGRATIONS` | [DatabaseMigrations.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/DatabaseMigrations.kt) | Room 迁移脚本 |
| `Converters` | [Converters.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/Converters.kt) | `Set<String>` ⇔ JSON TypeConverter |
| `MainAppDatabase.RemoveAppSettingsSpec` | [MainAppDatabase.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/MainAppDatabase.kt) | 4→5 删表 AutoMigrationSpec |

## 数据库（data/db/widget）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `WidgetDatabase` | [WidgetDatabase.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/widget/WidgetDatabase.kt) | 快照库（v3） |
| `WidgetCourse` / `WidgetAppSettings` | 同目录 | 快照实体 |
| `WidgetCourseDao` / `WidgetAppSettingsDao` | 同目录 | 快照 DAO |

## DI（data/di）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `DatabaseModule` | [DatabaseModule.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/di/DatabaseModule.kt) | 提供双库实例与全部 DAO |
| `DataStoreModule` | [DataStoreModule.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/di/DataStoreModule.kt) | 提供 4 个 DataStore（含 Proto） |

## Repository（data/repository）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `AppSettingsRepository` | [AppSettingsRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/AppSettingsRepository.kt) | 全局设置 SSOT + 周次/开学日算法 |
| `CourseTableRepository` | [CourseTableRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/CourseTableRepository.kt) | 课表/课程 CRUD + 调课三模式 |
| `TimeSlotRepository` | [TimeSlotRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/TimeSlotRepository.kt) | 时间段管理 |
| `StyleSettingsRepository` | [StyleSettingsRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/StyleSettingsRepository.kt) | 课表样式 Proto DataStore |
| `WidgetRepository` | [WidgetRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/WidgetRepository.kt) | 小组件数据 + `dataUpdatedFlow` |
| `CourseConversionRepository` | [CourseConversionRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/CourseConversionRepository.kt) | 导入导出转换（Web/JSON/ICS/日历） |
| `CourseImportExport` | [CourseImportExport.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/CourseImportExport.kt) | JSON/CBOR 模型与序列化器 |
| `BackupRepository` | [BackupRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/BackupRepository.kt) | 多模块备份包导出/恢复 |
| `GitRepository` / `GitRepositoryImpl` | [GitRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/GitRepository.kt) | 仓库服务，委托 GitUpdater |
| `SchoolRepository` / `SchoolHistoryRepository` | [SchoolRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/SchoolRepository.kt) | 学校索引解析 + 访问历史 |
| `ApiConfigRepository` | [ApiConfigRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/ApiConfigRepository.kt) | WebDAV 配置（密码加解密） |
| `ContributionRepository` | [ContributionRepository.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/ContributionRepository.kt) | 贡献者列表 |

## 同步（data/sync）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `SyncManager` | [SyncManager.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/sync/SyncManager.kt) | 启动闸门 + 事件总线消费 |
| `WidgetDataSynchronizer` | [WidgetDataSynchronizer.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/sync/WidgetDataSynchronizer.kt) | 主库→快照库同步（`syncFlow` / `syncNow` / `performSync`） |

## 小组件（widget）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `*NativeProvider`（4 个） | [widget/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/) | AppWidgetProvider，转发回调 |
| `*NativeRenderer`（4 个 object） | 同各子目录 | 纯函数 `render(snapshot): RemoteViews` |
| `WidgetUpdateHelper.updateAllWidgets` | [WidgetUpdateHelper.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/WidgetUpdateHelper.kt) | 统一刷新分发 |
| `WorkManagerHelper` | [WorkManagerHelper.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/WorkManagerHelper.kt) | 周期任务注册/取消 |
| `WidgetUiUpdateWorker` / `FullDataSyncWorker` | [WidgetSyncWorkers.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/WidgetSyncWorkers.kt) | 两个 `@HiltWorker` |

## 服务（service）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `CourseNotificationWorker` | [CourseNotificationWorker.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/service/CourseNotificationWorker.kt) | 课程提醒闹钟注册（50010–50110） |
| `DndSchedulerWorker` | [DndSchedulerWorker.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/service/DndSchedulerWorker.kt) | 勿扰闹钟注册（50001/50002）+ `enqueueWork` |
| `CourseAlarmReceiver` | [CourseAlarmReceiver.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/service/CourseAlarmReceiver.kt) | 闹钟到点处理：通知 / `toggleMode` |

## 工具（tool）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `GitUpdater` | [GitUpdater.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/GitUpdater.kt) | JGit 三阶段延迟写入 + 灯塔校验 |
| `LogProgressMonitor` | [LogProgressMonitor.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/LogProgressMonitor.kt) | JGit 进度回调 |
| `IcsExportTool` | [IcsExportTool.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/IcsExportTool.kt) | ICS 生成 + 日历批量插入 |
| `CalendarAccountManager` | [CalendarAccountManager.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/CalendarAccountManager.kt) | 系统日历账户管理 |
| `UpdateChecker` | [UpdateTool.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/UpdateTool.kt) | 应用更新检查 |
| `FileSharingUtils.shareFile` | [FileSharingUtils.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/FileSharingUtils.kt) | 文件分享 |

## 网络（data/api）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `ApiDateImporter` | [ApiDateImporter.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/api/date/ApiDateImporter.kt) | 节假日 API（Ktor）→ `skippedDates` |
| `WebDavConfig` / `WebDavClient` | [data/api/webdav/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/api/webdav/) | WebDAV 配置与客户端 |

## 教务适配（ui/schoolselection）

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `AndroidBridge` | [AndroidBridge.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/AndroidBridge.kt) | JS-Native 桥（`@JavascriptInterface`） |
| `WebViewScreen` / `WebViewModel` | [web/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/) | WebView 承载页 |
| `WebCompatDelegate` / `WebViewRequestInterceptor` / `WebPostInterceptJS` | [web/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/) | WebView 兼容与 POST 拦截 |
| `WebUiEvent` / `WebDialogHost` | [web/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/) | JS 触发的 Compose 弹窗 |
| `SchoolSelectionListScreen` / `AdapterSelectionScreen` | [list/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/list/) | 学校/适配器选择 |

## UI / 主题

| 类 | 文件 | 职责 |
| --- | --- | --- |
| `ShiguangScheduleTheme` | [Theme.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/theme/Theme.kt) | 主题入口（动态取色/自定义主色） |
| `WeeklyScheduleScreen` / `WeeklyScheduleViewModel` | [ui/schedule/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schedule/) | 周课表 |
| `TodayScheduleScreen` / `TodayScheduleViewModel` | [ui/today/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/today/) | 今日课表 |
| `SettingsScreen` / `SettingsViewModel` | [ui/settings/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/settings/) | 设置首页与全局设置状态 |
