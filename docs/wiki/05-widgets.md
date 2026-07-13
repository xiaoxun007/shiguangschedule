# 05 小组件系统

小组件目录：[app/src/main/java/.../widget/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/)，提供 4 种桌面小组件，采用统一的**瘦 Provider + 纯函数 Renderer** 模式。

## 四种小组件

| 类型 | 尺寸 | 展示形式 |
| --- | --- | --- |
| **tiny** | 2×1 | 仅"下一节课"气泡：课名/时间/地点/剩余节数 |
| **compact** | 2×2 | 今日剩余 → 明日预告 → 无课三态紧凑列表 |
| **double_days** | 4×2 | 左右双栏：今日剩余 vs 明日全部 |
| **list_vertical** | 4×2（仅垂直可缩放） | 垂直长列表三态切换，开始/结束时间分列 |

四种均使用 `widget_loading_placeholder.xml` 作为 `initialLayout`，共享 `widget_item_course_common.xml`、`widget_item_course_list_node.xml`、`widget_divider_horizontal.xml` 等子布局。系统 `updatePeriodMillis` 均为 30 分钟（系统下限）。元数据 XML 位于 `app/src/main/res/xml/shiguangschedule_*_widget.xml`，并在 [AndroidManifest.xml](../../../app/src/main/AndroidManifest.xml) 注册为 `APPWIDGET_UPDATE` Receiver。

## Provider / Renderer 协作模式

```
系统 onUpdate / Worker ──▶ *NativeProvider.onUpdate
                                   │
                                   ▼
              WidgetUpdateHelper.updateAllWidgets(context)
                  (组装统一 WidgetSnapshot protobuf)
                                   │
                ┌──────────┬───────┴───────┬──────────┐
                ▼          ▼               ▼          ▼
              Tiny     Compact       DoubleDays   ListVertical
           Renderer.render(context, snapshot) → RemoteViews
                                   │
                                   ▼
              AppWidgetManager.updateAppWidget(componentName, rv)
```

- **NativeProvider**（`AppWidgetProvider` 子类）：仅接收生命周期回调。`onUpdate` 转发到 `updateAllWidgets()`；`onEnabled`（首个实例添加）调用 `WorkManagerHelper.schedulePeriodicWork`；`onDisabled`（最后一个移除）调用 `cancelAllWork`。不持有数据、不构建 RemoteViews。
- **NativeRenderer**（`object`）：暴露纯函数 `render(context, snapshot): RemoteViews`，完成全部 UI 逻辑（状态重置防残留、点击跳转 MainActivity、假期/有课/无课三分支判定、课程过滤排序、颜色映射、动态 `addView`）。无状态可被任意线程安全调用。

四种 Renderer 共享同一份 `WidgetSnapshot`，避免重复查库。`WidgetSnapshot` 由 [widget_snapshot.proto](../../../app/src/main/proto/widget_snapshot.proto) 定义（courses + current_week + style）。

## 调度三件套

| 组件 | 职责 |
| --- | --- |
| [WidgetUpdateHelper](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/WidgetUpdateHelper.kt)（`updateAllWidgets`） | 统一刷新分发中心：从 `WidgetRepository` 拉今日+明日课程、当前周数、网格样式（带超时保底），组装 `WidgetSnapshot`，依次调用四种 Renderer 渲染并应用（规格间 300ms 延迟，单规格失败不影响其他） |
| [WorkManagerHelper](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/WorkManagerHelper.kt) | 注册/取消两个周期任务：`WidgetUiUpdateWorker_Periodic`（15 分钟，KEEP）与 `FullDataSyncWorker_Periodic`（1 天，UPDATE） |
| [WidgetSyncWorkers](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/widget/WidgetSyncWorkers.kt) | `WidgetUiUpdateWorker`（15 分钟仅刷新 UI）、`FullDataSyncWorker`（1 天，注入 `WidgetDataSynchronizer.syncNow()` 全量同步后刷新） |

## WidgetDatabase

详见 [03 数据层](./03-data-layer.md#widgetdatabase小组件快照库)。`WidgetCourse` 为按日期展平的渲染快照（复合主键 `id-date`），`WidgetAppSettings` 为单行学期配置。与主库物理隔离，由 `WidgetDataSynchronizer` 单向投影。

## 数据流与触发源

```
[事实来源] MainAppDatabase
   Course / CourseWeek / TimeSlot / CourseTableConfig / AppSettings
                  │  (响应式 syncFlow / 每日 syncNow)
                  ▼
        WidgetDataSynchronizer.performSync
   ── 展平 Course+CourseWeek+TimeSlot，按未来 7 天物化
   ── 计算周数、解析时间、标记 skippedDates
                  │
                  ▼
[快照库] WidgetDatabase (widget_courses / widget_semester_start_date)
                  │  (Flow 查询 + 周数计算)
                  ▼
          WidgetRepository
                  │  (带超时拉取 + 样式合并)
                  ▼
        WidgetUpdateHelper.updateAllWidgets
   ── 组装 WidgetSnapshot (protobuf)
                  │
          ┌───────┼───────┬───────────┐
          ▼       ▼       ▼           ▼
       Tiny  Compact  DoubleDays  ListVertical
     Renderer.render → RemoteViews → 桌面刷新
```

**三类触发源汇入同一 `updateAllWidgets` 入口：**

1. **系统触发**：`onUpdate`（APPWIDGET_UPDATE 广播 / 30 分钟系统周期）。
2. **周期触发**：`WidgetUiUpdateWorker`（15 分钟，仅刷新 UI）/ `FullDataSyncWorker`（1 天，先全量同步再刷新）。
3. **响应式触发**：`WidgetDataSynchronizer.syncFlow` 监听主库变化 → 重写快照库 → 末尾自调 `updateAllWidgets`。

## 关键类速查

| 类 | 职责 |
| --- | --- |
| `*NativeProvider`（4 个） | AppWidgetProvider，转发回调、管理 WorkManager 任务生命周期 |
| `*NativeRenderer`（4 个 object） | 纯函数渲染 `WidgetSnapshot` → `RemoteViews` |
| `WidgetUpdateHelper` | 统一组装快照并分发到四种 Renderer |
| `WorkManagerHelper` | 注册/取消 15 分钟 UI 刷新与 1 天全量同步周期任务 |
| `WidgetUiUpdateWorker` / `FullDataSyncWorker` | 两个 `@HiltWorker`，分别负责 UI 刷新与全量同步 |
| `WidgetRepository` | 封装快照库访问、当前周次流、`dataUpdatedFlow` 事件 |
| `WidgetDataSynchronizer` | 主库→快照库反规范化同步器（响应式 + 一次性双模式） |
| `SyncManager` | 等待主库就绪后启动同步器并派发通知/勿扰 Worker（详见 [06](./06-services-sync.md)） |
