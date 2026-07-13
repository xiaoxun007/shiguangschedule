# 06 服务与同步

负责课程提醒、勿扰自动化、主库→快照库同步与后台任务调度。

## 组件清单

| 组件 | 类型 | 职责 |
| --- | --- | --- |
| [CourseNotificationWorker](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/service/CourseNotificationWorker.kt) | `@HiltWorker` | 清空 50010–50110 旧闹钟后，按"提前 N 分钟"为未来 7 天课程批量注册 `setExactAndAllowWhileIdle` 精确闹钟（指向 `CourseAlarmReceiver`） |
| [DndSchedulerWorker](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/service/DndSchedulerWorker.kt) | `@HiltWorker` | 清空 50001/50002 后，查找下一节课起止时间并注册两个闹钟（上课开勿扰、下课关勿扰），自带 `enqueueWork()` |
| [CourseAlarmReceiver](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/service/CourseAlarmReceiver.kt) | `@AndroidEntryPoint` Receiver | 统一处理课程提醒闹钟（发通知 + 刷新小组件）与勿扰闹钟（`toggleMode` 切换 DND/静音，下课结束时重排 `DndSchedulerWorker`） |
| [SyncManager](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/sync/SyncManager.kt) | `@Singleton` | 监听 `MainAppDatabase.isInitialized` 就绪后启动 `WidgetDataSynchronizer.syncFlow`；500ms debounce 消费 `dataUpdatedFlow` 触发通知/勿扰 Worker |
| [WidgetDataSynchronizer](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/sync/WidgetDataSynchronizer.kt) | `@Inject` | 主库→快照库同步器，`combine(课程, 时间段, 配置)` 响应式计算未来 7 天 `WidgetCourse`；另提供 `syncNow()` 供定期 Worker 调用 |

## 课程提醒通知链路

```
触发源（三类）
 ├─ 数据变更：主库变化 → WidgetDataSynchronizer 写快照库
 │            → WidgetRepository.dataUpdatedFlow
 │            → SyncManager (debounce 500ms) → triggerNotificationWorker()
 ├─ 设置变更：NotificationSettingsViewModel 直接入队
 └─ 应用启动：MyApplication.onCreate → syncManager.startAllSynchronizers()
                         │
                         ▼
       CourseNotificationWorker.doWork()
   ── cancelAllAlarms() 清空 50010–50110
   ── 取未来 7 天课程（过滤 isSkipped），计算 remindTime = 开始 - N 分钟
   ── 仅对未来的时间点 setExactAndAllowWhileIdle(RTC_WAKEUP)
                         │
                         ▼ AlarmManager 到点
                CourseAlarmReceiver (ACTION_COURSE_REMIND)
   ── goAsync() + IO 协程
   ── showNotification()（高优先级通道、BigTextStyle、关闭 Action、promoted ongoing）
   ── removeAlarmIdFromPrefs() + updateAllWidgets()
```

- unique work 名：`CourseNotificationWorker_Sync_Update`（来自 SyncManager）与 `..._Settings_Update`（来自设置页），均 `REPLACE`。内部先整段清槽再重建，幂等安全。
- 通知 ID = `EXTRA_ALARM_SLOT_ID`（50010–50110），每节课通知独立。

## 勿扰 / 静音自动化

```
触发入队（三类）
 ├─ SyncManager 消费 dataUpdatedFlow
 ├─ CourseAlarmReceiver 收到 DND_ACTION_END（下课）后重排
 └─ NotificationSettingsScreen 设置变更
                         │
                         ▼
       DndSchedulerWorker.doWork()
   ── cancelDndAlarms() 清 50001/50002
   ── findNextDndAlarmTimes(7天) 找最近课程开始/结束
   ── setExactAndAllowWhileIdle 注册两个闹钟：
        50001 = DND_ACTION_START（上课）
        50002 = DND_ACTION_END  （下课）
                         │
                         ▼ AlarmManager 到点
                CourseAlarmReceiver
   ── DND_ACTION_START → toggleMode(true)
   ── DND_ACTION_END   → toggleMode(false) → DndSchedulerWorker.enqueueWork() 排下一次
```

`toggleMode` 依据 `AutoControlMode`：

- `DND`：`NotificationManager.setInterruptionFilter(PRIORITY/ALL)`
- `SILENT`：`AudioManager.ringerMode(SILENT/NORMAL)`

均要求 `isNotificationPolicyAccessGranted`（未授予仅告警不阻断）。

> **闹钟号段物理隔离**：DND 用 50001/50002，课程提醒用 50010–50110，便于各自整段清理，避免切课表时闹钟残留。

## 主库 → 快照库同步

存在三条链路，最终收敛到 `WidgetDataSynchronizer.performSync`：

1. **响应式同步（核心，进程内）**：`SyncManager` 监听 `isInitialized` → 启动 `syncFlow` = `appSettings` → `flatMapLatest(currentCourseTableId)` → `combine(课程, 时间段, 配置)` → `performSync`。
2. **定期全量同步（兜底，跨进程）**：小组件 `onEnabled` → `WorkManagerHelper.schedulePeriodicWork` → `FullDataSyncWorker`（1 天）→ `syncNow()`。
3. **UI 刷新链路（不触发数据同步）**：`WidgetUiUpdateWorker`（15 分钟）→ 仅 `updateAllWidgets`，保证过分钟时视图随时间推进。

`performSync` 逻辑：依据 `semesterStartDate / semesterTotalWeeks / firstDayOfWeek` 计算周次与星期，从今天（或开学日）起展开 7 天窗口，逐课生成 `WidgetCourse`（自定义时间或时间段映射、`isSkipped` 标记），最后 `deleteAll + insertAll + updateAllWidgets`。无配置时清空快照表并写兜底设置。

**反向反馈环**：`performSync` 写快照库 → `WidgetRepository` 发 `dataUpdatedFlow` → `SyncManager` 反向触发 `CourseNotificationWorker` 与 `DndSchedulerWorker` 重排闹钟，形成"主库变化 → 快照刷新 → 闹钟重排"闭环。

## 后台任务调度总览

| 任务 | 触发方式 | 周期/策略 | 闹钟号段 |
| --- | --- | --- | --- |
| 课程提醒重排 | 快照库更新（debounce 500ms）/ 设置变更 | OneTime, REPLACE | 注册 50010–50110 |
| 勿扰重排 | 快照库更新 / 下课广播 / 设置变更 | OneTime, REPLACE | 注册 50001/50002 |
| 精确闹钟到点 | `AlarmManager` 投递 | 单次广播 | 50001–50110 |
| 周期 UI 刷新 | 小组件启用时注册 | 15 分钟, KEEP | — |
| 周期全量同步 | 小组件启用时注册 | 1 天, UPDATE | — |
| 响应式数据同步 | 主库 Flow 变化 | 持续监听 | — |

**总体调度思路**：以 `MainAppDatabase.isInitialized` 为启动闸门，以 `dataUpdatedFlow` 为事件总线；响应式 Flow 负责实时性，WorkManager 一次性任务负责"重排闹钟"副作用，WorkManager 周期任务负责跨进程兜底；`setExactAndAllowWhileIdle` 保证 Doze 下准时唤醒；固定号段 + 整段清理策略避免残留。
