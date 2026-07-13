# 03 数据层

数据层遵循 **SSOT（单一事实来源）** 原则：`MainAppDatabase` 为权威数据源，`WidgetDatabase` 为渲染快照，`DataStore` 持久化偏好与样式。所有对外访问经 `Repository`（Hilt `@Singleton`）封装。

## Room 双数据库

### MainAppDatabase（主库，事实来源）

定义于 [MainAppDatabase.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/MainAppDatabase.kt)，数据库名 `main_app_database`，version 5，`exportSchema = true`（schema 存于 `app/schemas/`）。

实体关系：

```
CourseTable (课表元数据)
   │ 1
   │
   ├──< Course (课程)            ──< CourseWeek (课程-周次 多对多)
   │ 1        (courseTableId)        (courseId)
   │
   ├──< TimeSlot (时间段)         (courseTableId, 联合主键 number+courseTableId)
   │ 1
   └──1 CourseTableConfig (课表配置, courseTableId 唯一)
```

所有子表对 `CourseTable` 均为 `ForeignKey(onDelete = CASCADE)`。

| 实体 | 表名 | 关键字段 | 说明 |
| --- | --- | --- | --- |
| [CourseTable](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/CourseTable.kt) | `course_tables` | id, name, createdAt | 课表元数据 |
| [Course](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/Course.kt) | `courses` | id, courseTableId, name, day, startSection/endSection, isCustomTime, customStartTime/customEndTime, colorInt | 课程；支持节次模式与自定义时间模式（`isCustomTime=true` 时忽略节次） |
| [CourseWeek](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/CourseWeek.kt) | `course_weeks` | courseId, weekNumber（联合主键） | 课程与周次的多对多关联 |
| [TimeSlot](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/TimeSlot.kt) | `time_slots` | number, startTime, endTime, courseTableId, alias（联合主键 number+courseTableId） | 节次时间段 |
| [CourseTableConfig](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/CourseTableConfig.kt) | `course_table_config` | courseTableId, showWeekends, semesterStartDate, semesterTotalWeeks, defaultClassDuration/BreakDuration, firstDayOfWeek | 课表专属配置（一对一） |

**初始化**：数据库 `onCreate` 回调中写入默认课表（"我的课表"）、默认配置（20 周、45 分钟课时、10 分钟课间、周首日周一）与 13 节默认时间段；写完后置 `isInitialized = true`。`onOpen` 也置 true（用于已存在数据库的冷启动）。

**迁移**：[DatabaseMigrations.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/DatabaseMigrations.kt) 定义 `MIGRATION_1_2`（独立 `course_table_config` 表）、`MIGRATION_2_3`（课程自定义时间字段），并由 `AutoMigration` 处理 3→4、4→5（4→5 删除旧 `app_settings` 表，偏好迁至 DataStore）。[Converters.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/main/Converters.kt) 提供 `Set<String>` ⇔ JSON 的 TypeConverter（跳过日期集合）。

### WidgetDatabase（小组件快照库）

定义于 [WidgetDatabase.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/widget/WidgetDatabase.kt)，数据库名 `widget_database`，version 3，`fallbackToDestructiveMigration`。与主库物理隔离，不共享表/DAO。

| 实体 | 表名 | 说明 |
| --- | --- | --- |
| [WidgetCourse](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/widget/WidgetCourse.kt) | `widget_courses` | 按日期展平的课程快照，主键 `"${course.id}-$dateString"`，仅含渲染必需字段 |
| [WidgetAppSettings](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/db/widget/WidgetAppSettings.kt) | `widget_semester_start_date` | 单行（id=1）学期配置，供周数推算 |

数据由 `WidgetDataSynchronizer` 单向从主库投影写入，不回写主库。

## DataStore

定义于 [DataStoreModule.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/di/DataStoreModule.kt)，采用 Preferences 与 Proto 双轨：

| DataStore | 模式 | Named | 用途 |
| --- | --- | --- | --- |
| `appSettingsDataStore` | Preferences | `AppSettings` | 全局设置（提醒、勿扰、起始页、主题、跳过日期等） |
| `schoolHistoryDataStore` | Preferences | `SchoolHistory` | 学校选择历史（本科/研究生/通用工具三类） |
| `apiConfigDataStore` | Preferences | `ApiConfig` | WebDAV 等 API 配置（密码经 AndroidKeyStore AES/GCM 加密） |
| `scheduleGridStyleDataStore` | **Proto**（`ScheduleGridStyleProto`） | — | 课表网格样式（Wire 生成） |

## Repository 职责

所有 Repository 为 Hilt `@Singleton`（`ContributionRepository`/`SchoolRepository` 为 `object`），封装对应 DAO/DataStore 并承载业务逻辑。

| Repository | 职责 |
| --- | --- |
| [AppSettingsRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/AppSettingsRepository.kt) | 全局设置 SSOT，协调 DataStore 偏好与 `CourseTableConfig`，提供设置读写、课表配置 CRUD 及周次/开学日反推算法 |
| [CourseTableRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/CourseTableRepository.kt) | 课表与课程核心 CRUD，建表（含默认时间段/配置）、课程 upsert/删除、调课三模式（合并/覆盖/交换）、按日期/周次查询 |
| [TimeSlotRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/TimeSlotRepository.kt) | 时间段查询流、批量插入、整表替换 |
| [StyleSettingsRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/StyleSettingsRepository.kt) | 基于 Proto DataStore 管理课表网格样式，原子化 setter、字节导出/恢复、小组件联动刷新 |
| [WidgetRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/WidgetRepository.kt) | 小组件数据查询、批量写入、设置读写、当前周次流；通过 Channel 发出 `dataUpdatedFlow` |
| [CourseConversionRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/CourseConversionRepository.kt) | 课表导入导出转换：Web 导入（`importCoursesFromList`/`importCourseConfig`/`importTimeSlots`）、JSON 互转、ICS 导出、系统日历同步 |
| [CourseImportExport](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/CourseImportExport.kt) | 纯数据模型与序列化器（JSON/CBOR），是 Bridge 与 ConversionRepo 之间的数据契约层 |
| [BackupRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/BackupRepository.kt) | 多模块备份包（CBOR 课程 + Proto 样式 + 版本信封）的原子导出与带校验恢复 |
| [GitRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/GitRepository.kt) | 仓库服务接口及实现，委托 `GitUpdater` 执行 JGit 拉取（见 [07](./07-tools-import.md)） |
| [SchoolRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/SchoolRepository.kt) | 从 `filesDir/repo/index/school_index.pb`（Wire）解析学校索引；同文件 `SchoolHistoryRepository` 持久化访问历史 |
| [ContributionRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/ContributionRepository.kt) | 从 Assets 读取 `contributors_data/contributors.json` 反序列化贡献者列表 |
| [ApiConfigRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/ApiConfigRepository.kt) | WebDAV 配置持久化（密码加解密），提供 `WebDavClient` 工厂 |

## 业务模型（data/model）

| Model | 说明 |
| --- | --- |
| [AppSettingsModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/model/AppSettingsModel.kt) | 全局设置模型 + DataStore 键集，含 `AutoControlMode`(DND/静音)、`StartScreen`、`AppThemeMode` 枚举与 `fromPreferences` 解析器 |
| [ScheduleGridStyle](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/model/ScheduleGridStyle.kt) | 课表网格样式 Compose 模型（12 套默认双色），含 Proto ⇔ Compose 双向转换 `toCompose()`/`toProto()` |
| [RepoInfo](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/model/RepoInfo.kt) | Git 仓库源信息，含 `RepoType` 枚举（官方/公开 Fork/私有/自定义） |
| [SchoolHistoryModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/model/SchoolHistoryModel.kt) | 学校选择历史模型（三类分类记录） |
| [ContributionList](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/model/ContributionList.kt) | 贡献者 JSON 根结构（`app_dev` + `jiaowu_adapter`） |

## DI 模块（data/di）

| Module | 提供内容 |
| --- | --- |
| [DatabaseModule](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/di/DatabaseModule.kt) | `MainAppDatabase`、`WidgetDatabase` 单例，及全部 DAO（Course/TimeSlot/CourseTable/CourseWeek/CourseTableConfig/WidgetCourse/WidgetAppSettings） |
| [DataStoreModule](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/di/DataStoreModule.kt) | 4 个 DataStore（含 Proto 样式 DataStore），用 `@Named` 区分 Preferences 实例 |

> Repository 本身通过构造函数注入 DAO/DataStore，由 Hilt 自动构建，无需单独 `@Provides`。

## Proto 定义（app/src/main/proto/）

| Proto | 生成包 | 用途 |
| --- | --- | --- |
| [school_index.proto](../../../app/src/main/proto/school_index.proto) | `school_index` | 教务适配索引：`SchoolIndex`/`School`/`Adapter` + `AdapterCategory` 枚举 |
| [schedule_style.proto](../../../app/src/main/proto/schedule_style.proto) | `data.model.schedule_style` | 课表网格样式 `ScheduleGridStyleProto`（尺寸/颜色池/开关/边框/模式） |
| [widget_snapshot.proto](../../../app/src/main/proto/widget_snapshot.proto) | — | 小组件快照 `WidgetSnapshot`（courses + current_week + style），import schedule_style.proto |

由 Wire 插件生成 Kotlin 类，配置见 [build.gradle.kts](../../../app/build.gradle.kts) 的 `wire { }` 块。
