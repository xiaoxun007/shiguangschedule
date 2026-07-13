# 01 项目概览

## 项目定位

拾光课程表是一款**面向中国高校师生**的课程表管理工具，核心价值在于通过**社区维护的适配脚本**一键导入各类教务系统课程数据，并提供小组件、提醒、勿扰、备份等完整课表管理能力。项目采用开源模式，鼓励社区开发者参与教务适配与功能贡献。

适配脚本仓库与本项目分离，位于 [shiguang_warehouse](https://github.com/XingHeYuZhuan/shiguang_warehouse)，由 `JGit` 在应用内拉取。

## 功能概览

| 模块 | 能力 |
| --- | --- |
| 课表展示 | 今日课表 / 周课表（左右滑动切周，长按调整课程块） |
| 课表配置 | 多课表管理、独立时间表、周首日（周一/周日）、个性化样式 |
| 课程导入 | 教务适配导入（WebView + JS Bridge）、JSON 导入导出、ICS 导出、系统日历同步 |
| 小组件 | 4 种桌面小组件（tiny / compact / double_days / list_vertical），支持明日预告与深色 |
| 课程提醒 | 提前 N 分钟通知、节假日跳过、勿扰/静音自动化 |
| 备份恢复 | JSON 备份、CBOR 全量备份、WebDAV 云备份 |
| 多语言 | 简体中文、繁体中文、英语 |
| 深色适配 | 全量深色模式，含小组件 |

## 技术栈

| 类别 | 技术 |
| --- | --- |
| UI 框架 | Jetpack Compose（Material 3，含动态取色） |
| 导航 | Navigation3（`androidx.navigation3`） |
| 依赖注入 | Hilt（`com.google.dagger.hilt`） |
| 本地存储 | Room（双数据库）+ DataStore（Preferences + Proto） |
| 序列化 | kotlinx.serialization（JSON / CBOR）+ Wire（Protobuf） |
| 后台任务 | WorkManager + AlarmManager（精确闹钟） |
| 网络 | Ktor 3（WebDAV / 节假日 API）+ OkHttp（更新检查 / WebView 拦截） |
| Git | JGit（拉取适配脚本仓库） |
| 图片加载 | Coil 3 |
| 其他 | aboutLibraries（开源协议页）、AppCompat（语言切换） |

## 版本与 Flavor

项目通过 Gradle `productFlavors`（`version` 维度）区分**开发版（dev）**与**正式版（prod）**，关键差异由 `BuildConfig` 字段注入。详见 [build.gradle.kts](../../../app/build.gradle.kts)。

| BuildConfig 字段 | dev | prod | 含义 |
| --- | --- | --- | --- |
| `CURRENT_FLAVOR_ID` | `"dev"` | `"prod"` | 环境标识 |
| `HIDE_CUSTOM_REPOS` | `false` | `true` | 是否隐藏自定义/私有仓库 |
| `ENABLE_LIGHTHOUSE_VERIFICATION` | `false` | `true` | 是否开启基准灯塔标签校验 |
| `ENABLE_DEV_TOOLS_OPTION_IN_UI` | `true` | `false` | 是否在 UI 显示 DevTools 选项 |
| `ENABLE_ADDRESS_BAR_TOGGLE_BUTTON` | `true` | `false` | 是否显示地址栏切换按钮 |

其他差异：

- **dev**：包名追加 `.dev` 后缀、版本名追加 `-dev`，可与正式版共存；图标为**红色**背景。
- **prod**：默认隐藏自定义仓库并开启安全验证，推荐普通用户使用；图标为**蓝色**背景。

正式版的安全机制（灯塔标签验证）确保用户导入的适配脚本来自经过校验的合法 fork，详见 [07 工具与教务适配](./07-tools-import.md)。
