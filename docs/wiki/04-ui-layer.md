# 04 UI 层

UI 层基于 **Jetpack Compose + Material 3**，使用 **Navigation3** 做类型安全路由，每个屏幕配套 Hilt `@HiltViewModel`。

## 应用入口

- [MyApplication.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/MyApplication.kt)：`@HiltAndroidApp`，实现 `Configuration.Provider` 注入 `HiltWorkerFactory`；`onCreate` 中清理分享临时文件、启动 `SyncManager`、协程拷贝离线仓库。
- [MainActivity.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/MainActivity.kt)：`@AndroidEntryPoint`，`enableEdgeToEdge`；等待 `SettingsViewModel` 的 `uiState.isReady` 后套用 `ShiguangScheduleTheme` 并按 `startScreen` 设置起始目的地，进入 `AppNavigation`。

## 导航（Navigation3）

路由定义在 [Navigation.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/Navigation.kt) 与 [AppChannels.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/navigation/AppChannels.kt)。

- `Destination`：`@Serializable sealed interface : NavKey`，分为**一级主屏**（无转场动画）与**二级页面**（左右滑动动画）及**带参数动态页**。
- `ShiguangNavMetadata.IsMainScreenKey`：标记一级屏，控制 `transitionSpec` 主屏间无过渡。
- `AppNavigation`：维护 `rememberNavBackStack`，`onNavigate` 对主屏做 `clear + add`（避免栈堆积），二级页 `add`。
- `ScreenContent`：`when (targetDest)` 分发到具体 Composable。

### 一级主屏

| Destination | Composable | 说明 |
| --- | --- | --- |
| `CourseSchedule` | [WeeklyScheduleScreen](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schedule/WeeklyScheduleScreen.kt) | 周课表，左右滑动切周，长按调整课程块 |
| `TodaySchedule` | [TodayScheduleScreen](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/today/TodayScheduleScreen.kt) | 今日课表 |
| `Settings` | [SettingsScreen](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/settings/SettingsScreen.kt) | 设置首页（聚合各配置入口） |

### 二级页面（按 settings 子包）

| 子包 | 屏幕 | 功能 |
| --- | --- | --- |
| `time` | TimeSlotManagementScreen | 时间段管理 |
| `coursetables` | ManageCourseTablesScreen | 多课表管理 |
| `course` | AddEditCourseScreen | 添加/编辑课程 |
| `coursemanagement` | CourseNameListScreen / CourseInstanceListScreen | 全局课程管理 |
| `conversion` | CourseTableConversionScreen | 课表导入导出/ICS/日历同步 |
| `style` | StyleSettingsScreen | 课表样式（格子/颜色/透明度等） |
| `themesettings` | ThemeSettingsScreen | 主题与动态取色 |
| `notification` | NotificationSettingsScreen | 课程提醒/勿扰设置 |
| `backup` | BackupScreen | 备份恢复（JSON/CBOR/WebDAV） |
| `additional` | MoreOptionsScreen / OpenSourceLicensesScreen | 更多选项、开源协议、语言 |
| `contribution` | ContributionScreen | 贡献者列表 |
| `update` | UpdateRepoScreen | 适配仓库更新 |
| `quickactions` | QuickActionsScreen / TweakScheduleScreen / QuickDeleteScreen | 快捷操作（调课/批量删除） |
| `schoolselection` | SchoolSelectionListScreen / AdapterSelectionScreen / WebViewScreen | 学校选择与教务适配导入（详见 [07](./07-tools-import.md)） |

> 起始页由 `AppSettings.startScreen` 决定（`COURSE_SCHEDULE` 或 `TODAY_SCHEDULE`）。

## ViewModel

各屏幕配套 `@HiltViewModel`，通过 `hiltViewModel()`/`viewModels()` 获取，订阅 Repository 的 `Flow` 暴露 `StateFlow<UiState>`。典型：

- [SettingsViewModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/settings/AppSettingsViewModel.kt)：聚合 `AppSettingsRepository`，提供全局设置状态（含 `isReady`、`appSettings`）。
- [WeeklyScheduleViewModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schedule/WeeklyScheduleViewModel.kt)：周课表数据与交互。
- [TodayScheduleViewModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/today/TodayScheduleViewModel.kt)：今日课程。
- [WebViewModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebViewModel.kt)：暴露 `CourseConversionRepository` 与 `TimeSlotRepository` 供 WebView 导入使用。

其余 ViewModel 与对应 Screen 同包，命名一致。

## 通用组件（ui/components）

| 组件 | 用途 |
| --- | --- |
| [NavigationComponents](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/components/NavigationComponents.kt) | 底栏 / 顶部导航 |
| [DatePickerModal](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/components/DatePickerModal.kt) | 日期选择弹层 |
| [CourseTablePickerDialog](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/components/CourseTablePickerDialog.kt) | 课表选择对话框 |
| [AdvancedColorPicker](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/components/AdvancedColorPicker.kt) | 高级颜色选择器 |
| [AlphabetIndexerList](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/components/AlphabetIndexerList.kt) | 字母索引列表 |
| [NumberPicker](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/components/NumberPicker.kt) | 数字选择器 |
| [ImageCropper](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/components/ImageCropper.kt) | 图片裁剪 |

## 跨页面数据传递

[AppChannels.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/navigation/AppChannels.kt) 定义 `AddEditCourseChannel`（`Channel.CONFLATED`）+ `PresetCourseData`，用于从课表长按位置一次性传递预设数据到"添加/编辑课程"页（名称、节次、时间、周次、颜色等）。

## 主题体系（ui/theme）

| 文件 | 职责 |
| --- | --- |
| [Theme.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/theme/Theme.kt) | `ShiguangScheduleTheme` 入口：依据 `AppSettingsModel` 解析深浅模式；Android 12+ 优先动态取色（Material You），否则回退自定义主色；通过 `SideEffect` 配置透明系统栏与图标明暗；用 `LocalIsDarkTheme` 全局广播深色状态 |
| [Color.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/theme/Color.kt) | Material3 基础调色板（浅/深色组）与自定义主色默认值 |
| [Type.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/theme/Type.kt) | Material3 `Typography`，仅自定义 `bodyLarge` |

深色适配覆盖全部页面与小组件，小组件布局有 `layout-night/` 变体。
