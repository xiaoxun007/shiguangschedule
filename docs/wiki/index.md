# 拾光课程表 · Code Wiki

本 Wiki 是对 [拾光课程表](https://github.com/XingHeYuZhuan/shiguangschedule)（`shiguangschedule`）源码的结构化解读文档，侧重**整体架构与模块职责**，帮助开发者快速理解项目全貌并上手维护。

> 拾光课程表是一款面向中国高校师生的 Android 课程表管理工具，支持通过社区适配脚本一键导入各类教务系统课程数据，并提供多种桌面小组件、课程提醒、勿扰自动化、ICS 导出与 WebDAV 备份等能力。

## 文档导航

| 章节 | 内容 |
| --- | --- |
| [01 项目概览](./01-overview.md) | 项目定位、技术栈、版本与 Flavor（dev/prod） |
| [02 整体架构](./02-architecture.md) | 分层架构、数据流、目录结构 |
| [03 数据层](./03-data-layer.md) | Room 双数据库、实体关系、DataStore、Repository、DI 模块 |
| [04 UI 层](./04-ui-layer.md) | Navigation3 导航、主要屏幕与 ViewModel、主题体系 |
| [05 小组件系统](./05-widgets.md) | 四种小组件、Provider/Renderer 模式、同步与调度 |
| [06 服务与同步](./06-services-sync.md) | 课程提醒、勿扰自动化、后台任务、数据同步机制 |
| [07 工具与教务适配](./07-tools-import.md) | 工具层、WebView/JS Bridge 导入、Git 仓库、离线资源 |
| [08 依赖关系](./08-dependencies.md) | 第三方库清单与用途 |
| [09 构建与运行](./09-build-run.md) | 环境要求、构建命令、Flavor、签名、产物 |
| [10 关键类索引](./10-key-index.md) | 按模块分组的关键类与函数速查 |

## 阅读建议

- **快速了解项目**：从 [01 项目概览](./01-overview.md) → [02 整体架构](./02-architecture.md) 开始。
- **参与教务适配开发**：重点阅读 [07 工具与教务适配](./07-tools-import.md)。
- **维护小组件 / 后台任务**：重点阅读 [05 小组件系统](./05-widgets.md) 与 [06 服务与同步](./06-services-sync.md)。
- **查找某个类的职责**：直接查阅 [10 关键类索引](./10-key-index.md)。

## 基本信息速览

| 项 | 值 |
| --- | --- |
| 包名 | `com.xingheyuzhuan.shiguangschedule` |
| 最低支持 | Android 8.0（API 26） |
| 目标/编译 SDK | 37 |
| 语言 / JVM | Kotlin 2.4 / JDK 21 |
| 构建系统 | Gradle（Kotlin DSL）+ Version Catalog |
| 版本号 | 1.2.5（versionCode 33） |
