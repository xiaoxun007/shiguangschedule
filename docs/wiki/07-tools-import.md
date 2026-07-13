# 07 工具与教务适配

本章覆盖工具层（[tool/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/)）与教务适配导入系统（[ui/schoolselection/](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/)）。

## 工具层（tool/）

| 工具 | 职责 |
| --- | --- |
| [GitUpdater](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/GitUpdater.kt) | 基于 JGit 的适配仓库更新器，三阶段延迟写入：资源/索引暂存 → 协议版本与时间戳去重 → 原子写入 `filesDir/repo/`；非官方仓库先校验灯塔标签 |
| [LogProgressMonitor](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/LogProgressMonitor.kt) | 实现 JGit `ProgressMonitor`，把克隆/拉取进度按百分比回调到日志，供 `GitUpdater` 输出 |
| [IcsExportTool](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/IcsExportTool.kt) | 课表→日历转换引擎：按学期起始日与时间槽推算每节课精确日期，生成 ICS 文本与系统日历 `ContentProviderOperation` 批量插入指令 |
| [CalendarAccountManager](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/CalendarAccountManager.kt) | 单例，通过 `CalendarContract` 查询/创建应用专属本地日历账户，返回日历 ID |
| [UpdateTool](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/UpdateTool.kt) | `UpdateChecker`，OkHttp 从 Gitee/GitHub 拉更新索引 JSON，按 Flavor 与 ABI 匹配下载链接并比较版本号 |
| [FileSharingUtils](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/tool/FileSharingUtils.kt) | 顶层函数 `shareFile`，封装 `Intent.ACTION_SEND` + FileProvider 分享文件 |

> 网络相关：[ApiDateImporter](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/api/date/ApiDateImporter.kt)（Ktor 请求 `timor.tech` 节假日 API，写入 `skippedDates`）、[WebDavClient](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/api/webdav/WebDavClient.kt)（Ktor 实现的 WebDAV 客户端，BasicAuth + MKCOL/PUT/GET，用于云备份）。

## 教务适配导入系统

### 学校选择（list/）

| 文件 | 职责 |
| --- | --- |
| [SchoolSelectionListScreen](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/list/SchoolSelectionListScreen.kt) | 一级学校选择页：类别 Tab（本科专科/研究生/通用工具）、字母索引、搜索、最近访问 |
| [SchoolSelectionViewModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/list/SchoolSelectionViewModel.kt) | 加载学校列表、组合过滤、读写访问历史 |
| [AdapterSelectionScreen](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/list/AdapterSelectionScreen.kt) | 二级适配器选择页：按 schoolId+category 异步加载适配器卡片，点击携带 import_url 与 assetJsPath 进入 WebView |

### WebView 适配（web/）

| 文件 | 职责 |
| --- | --- |
| [WebViewScreen](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebViewScreen.kt) | WebView 承载页：配置 WebView、注入 `AndroidBridge`、地址栏/桌面模式/SSL 对话框、底部"执行导入"按钮（选课表后注入 school.js 执行） |
| [WebViewModel](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebViewModel.kt) | 暴露 `CourseConversionRepository` 与 `TimeSlotRepository` |
| [AndroidBridge](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/AndroidBridge.kt) | **核心 JS-Native 桥**，`@JavascriptInterface` 暴露方法（见下表） |
| [WebViewUtils.kt](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebViewUtils.kt) | `injectAllJavaScript`：在 window 建立 `_resolveAndroidPromise/_rejectAndroidPromise` 与 `AndroidBridgePromise`，把同步 Native 调用包装成 Promise |
| [WebCompatDelegate](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebCompatDelegate.kt) | WebView 配置与客户端包装：enhanceSettings、注入 POST 拦截 JS、桌面 CSS viewport 修复、统一进度回调 |
| [WebViewRequestInterceptor](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebViewRequestInterceptor.kt) | OkHttp 代理 WebView 请求，解决 POST Body 丢失；含 `WebPostBridge`（注入名 "WebPostService"） |
| [WebPostInterceptJS](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebPostInterceptJS.kt) | 注入 JS：劫持 `XMLHttpRequest`/`fetch`/表单 submit，把非 GET Body 经 `WebPostService.register` 注册并附 `X-WebView-Post-Id` |
| [WebUiEvent](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebUiEvent.kt) | sealed interface，定义 JS 触发的三类 UI 事件（ShowAlert/ShowPrompt/ShowSingleSelection） |
| [WebDialogHost](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/ui/schoolselection/web/WebDialogHost.kt) | Composable 宿主，收集 `WebUiEvent` 渲染 AlertDialog，用户操作经 Bridge 解决 JS Promise |

**AndroidBridge 暴露方法**（均通过 promiseId + `AndroidBridgePromise` 异步回调）：

| 方法 | 作用 |
| --- | --- |
| `showToast` | 显示 Toast |
| `showAlert` | 确认弹窗 |
| `showPrompt` | 输入弹窗（支持 JS 验证函数异步校验） |
| `showSingleSelection` | 单选列表弹窗 |
| `saveImportedCourses` | 接收课程 JSON → `CourseConversionRepository.importCoursesFromList` 落库 |
| `saveCourseConfig` | 接收课表配置 → `importCourseConfig` 落库 |
| `savePresetTimeSlots` | 接收预设时间段 → `importTimeSlots` 落库 |
| `notifyTaskCompletion` | 通知 Native 完成，跳转课表页 |

## Git 仓库与离线资源

### 仓库源清单

[git_repos.json](../../../app/src/main/assets/git_repos.json) 内置仓库源：官方 GitHub、Gitee 镜像、ITDONG 镜像（PUBLIC_FORK）、可编辑自定义仓库（CUSTOM）、私有仓库（PRIVATE_REPO），对应 [RepoInfo](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/model/RepoInfo.kt) / `RepoType` 模型。

### GitUpdater 三阶段流程

1. **updateResourceFiles**：clone 或 fetch+hard reset 指定分支的 `resources/` 目录到临时仓库，暂存所有文件（跳过 `adapters.yaml`），目标路径 `filesDir/repo/schools/resources/...`（即 school.js 等适配脚本）。
2. **downloadIndexFile**：clone 远程 `index-pb-release` 分支读取 `school_index.pb`，校验 `protocol_version` 不高于客户端版本（否则致命中止），比较 `version_id` 时间戳去重，暂存新索引。
3. **commitUpdates**：备份旧索引 → 删除重建 `filesDir/repo` → 写入暂存资源 → 写入新索引或恢复备份 → 清理临时目录。

**安全校验**：非官方（非 OFFICIAL）仓库先用 `isLegitimateFork` 通过 `Git.lsRemoteRepository` 检查远程是否含名为 `lighthouse`、SHA 为 `eb49b7c...` 的基准灯塔标签；正式版（`ENABLE_LIGHTHOUSE_VERIFICATION=true`）强制开启。私有仓库通过 `UsernamePasswordCredentialsProvider` 提供凭证。

### 离线兜底

- [offline_repo/index/school_index.pb](../../../app/src/main/assets/offline_repo/index/school_index.pb)：内置 Protobuf 索引种子，App 首次启动由 `MyApplication.initOfflineRepo()` 递归拷贝到 `filesDir/repo/`，作为 GitUpdater 拉取前的兜底数据。
- [offline_repo/schools/resources/GLOBAL_TOOLS/school.js](../../../app/src/main/assets/offline_repo/schools/resources/GLOBAL_TOOLS/school.js)：通用工具适配脚本样例，演示 `AndroidBridgePromise.*` API 用法。

### 索引协议（proto）

[school_index.proto](../../../app/src/main/proto/school_index.proto) 定义：

- `AdapterCategory` 枚举：`GENERAL_TOOL=1` / `BACHELOR_AND_ASSOCIATE=2` / `POSTGRADUATE=3`
- `SchoolIndex`：protocol_version、version_id、repeated School
- `School`：id/name/initial/resource_folder、repeated Adapter
- `Adapter`：adapter_id/adapter_name/category/asset_js_path/optional import_url/description/maintainer

由 [SchoolRepository](../../../app/src/main/java/com/xingheyuzhuan/shiguangschedule/data/repository/SchoolRepository.kt) 用 Wire 解析，提供 `getSchools`/`getAdaptersForSchool`/`getSchoolById`。

## 完整教务导入数据流

```
1. 首次启动  MyApplication.initOfflineRepo()
             assets/offline_repo → filesDir/repo/（离线兜底）

2. 仓库更新（可选）  用户选仓库源 → GitRepositoryImpl → GitUpdater
             灯塔标签校验 → 拉取 resources + school_index.pb
             协议版本/时间戳校验 → 原子写入 filesDir/repo/

3. 学校选择  SchoolSelectionListScreen → SchoolRepository.getSchools(.pb)
             选学校 → AdapterSelectionScreen → getAdaptersForSchool

4. 进入 WebView  携 import_url + assetJsPath → WebViewScreen
             WebCompatDelegate 配置（电脑模式启用 POST 拦截）
             页面加载完成 injectAllJavaScript 建立 Promise 桥

5. 执行导入  点"执行导入" → CourseTablePickerDialog 选课表
             AndroidBridge.setImportTableId → 读取 school.js → evaluateJavascript

6. 脚本运行  school.js 抓取教务数据
             AndroidBridgePromise.showAlert/showPrompt/showSingleSelection 交互
             （WebDialogHost 渲染 Compose 弹窗并 Promise 回调）
             saveImportedCourses/saveCourseConfig/savePresetTimeSlots 传回 JSON

7. 落库  AndroidBridge 用 CourseImportExport.json 解析
        CourseConversionRepository.importCoursesFromList/Config/TimeSlots
        @Transaction 校验 → 清旧 → 批量插入（颜色按课名映射）

8. 完成  notifyTaskCompletion → Toast → 跳转课表页

辅助：ApiDateImporter 拉节假日供 ICS/日历跳过假期
      WebDavClient 支持 CBOR 全盘多课表备份/恢复
      导出侧 exportCourseTableToJson / exportToIcsString / syncCurrentTableToSystemCalendar
```
