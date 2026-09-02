# Webnovel Writer · Code Wiki

> 长篇网文创作插件的完整代码文档。本 Wiki 面向维护者与贡献者，覆盖项目架构、模块职责、关键类与函数、依赖关系以及运行方式。

## 项目一句话

**Webnovel Writer** 是一个跑在 Claude Code 上的长篇网文创作插件（v6.2.1，GPL-3.0），通过 **Skills（斜杠命令）+ Agents（子代理）+ Python 数据链 + RAG 检索** 四层结构，让 AI 写到几百章仍能记住设定、接住伏笔、守住大纲——它是一套面向长篇连载的**一致性系统**，不是一次性生成器。

## 快速导航

| 文档 | 内容 | 适合谁读 |
|------|------|----------|
| [01-项目概述与快速开始](01-项目概述与快速开始.md) | 项目定位、核心能力、技术栈、仓库结构、安装运行 | 所有人 |
| [02-整体架构](02-整体架构.md) | 分层架构、真源划分、Story System 合同体系、写章流水线、投影机制 | 架构理解 |
| [03-模块与目录结构](03-模块与目录结构.md) | 全仓库目录树注释与各目录职责 | 所有人 |
| [04-Skills 命令详解](04-Skills命令详解.md) | 8 个斜杠命令的流程、参数、引用资料与约束 | 使用者/贡献者 |
| [05-Agents 与 Hooks 详解](05-Agents与Hooks详解.md) | 4 个子代理的输入输出 schema、2 个运行时 Hook 的防护逻辑 | 贡献者 |
| [06-CLI 工具链详解](06-CLI工具链详解.md) | `webnovel.py` 统一入口、全部子命令映射、项目根定位机制 | 贡献者 |
| [07-核心数据模块详解](07-核心数据模块详解.md) | 合同引擎、章节提交链、状态双写、SQLite 索引、投影体系 | 核心贡献者 |
| [08-记忆与 RAG 检索详解](08-记忆与RAG检索详解.md) | 长期记忆分层、BM25/向量/图谱检索、写前上下文组装 | 核心贡献者 |
| [09-Dashboard 详解](09-Dashboard详解.md) | FastAPI 只读后端 30 个 API、React 前端 6 页面、SSE 实时刷新 | 前端/全栈贡献者 |
| [10-依赖关系详解](10-依赖关系详解.md) | 运行时依赖、模块依赖图、数据文件血缘 | 架构理解 |
| [11-运行测试与发布指南](11-运行测试与发布指南.md) | 环境搭建、pytest（覆盖率 ≥90%）、Dashboard 开发、发版流程 | 贡献者 |

## 推荐阅读顺序

1. **[01-项目概述与快速开始](01-项目概述与快速开始.md)** —— 了解这个项目是什么、装完怎么跑。
2. **[02-整体架构](02-整体架构.md)** —— 建立"合同 → 提交 → 投影 → 只读视图"的核心心智模型。
3. **[03-模块与目录结构](03-模块与目录结构.md)** —— 对照目录树找到每个概念的落点。
4. 按需深入：写章链路读 **[04](04-Skills命令详解.md) → [05](05-Agents与Hooks详解.md) → [07](07-核心数据模块详解.md)**；检索链路读 **[08](08-记忆与RAG检索详解.md)**；可视化读 **[09](09-Dashboard详解.md)**。
5. 动手改代码前读 **[10-依赖关系](10-依赖关系详解.md)** 与 **[11-运行测试与发布](11-运行测试与发布指南.md)**。

## 核心概念速查

| 概念 | 一句话解释 | 详见 |
|------|-----------|------|
| Story System | `.story-system/` 目录下"合同种子 + 章节提交 + 事件审计"的**唯一事实主链** | [02-整体架构](02-整体架构.md) |
| CHAPTER_COMMIT | 一章写完后的结构化"事实入账单"，`accepted` 才驱动投影 | [07-核心数据模块](07-核心数据模块详解.md) |
| 投影（Projection） | 从 accepted commit 派生 state.json / index.db / summaries / memory / vectors 五路只读视图 | [07-核心数据模块](07-核心数据模块详解.md) |
| write-gate | prewrite / precommit / postcommit 三道关卡，阻断不合格的写作与提交 | [02-整体架构](02-整体架构.md) |
| 三层记忆 | working / episodic / semantic 记忆包，按任务预算注入写前上下文 | [08-记忆与RAG](08-记忆与RAG检索详解.md) |
| RAG 多级降级 | graph_hybrid / hybrid → RRF → rerank；无 Embedding Key 自动退 BM25 | [08-记忆与RAG](08-记忆与RAG检索详解.md) |
| Strand Weave | 主线(Quest 60%) / 感情(Fire 20%) / 世界观(Constellation 20%) 三线节奏约束 | [02-整体架构](02-整体架构.md) |
| Dashboard | FastAPI + React 的只读可视化面板，watchdog + SSE 实时刷新 | [09-Dashboard](09-Dashboard详解.md) |

## 关键入口

| 入口 | 位置 | 说明 |
|------|------|------|
| 插件清单 | `.claude-plugin/marketplace.json` / `webnovel-writer/.claude-plugin/plugin.json` | Marketplace 安装与版本（6.2.1） |
| CLI 统一入口 | `webnovel-writer/scripts/webnovel.py` | 所有命令行工具的唯一入口 |
| Dashboard 启动 | `webnovel-writer/dashboard/`（`python -m dashboard`） | 默认 `127.0.0.1:8765` |
| 测试配置 | `pytest.ini`（根目录） | 覆盖率门槛 90% |
| 官方文档 | `docs/`（README → architecture → guides → operations） | 仓库自带文档中心 |
