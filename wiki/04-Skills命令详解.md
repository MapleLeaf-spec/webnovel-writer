# 04 · Skills 命令详解

> 8 个斜杠命令定义在 `webnovel-writer/skills/<name>/SKILL.md`。SKILL.md 与共享 references 是"行为闸门"——Claude Code 执行时的强制流程；CSV 是"知识库"——按需 BM25 检索。

## 1. 命令总览

| 命令                               | 用途       | 核心关卡/闸门                   | 关键 references                               |
| -------------------------------- | -------- | ------------------------- | ------------------------------------------- |
| `/webnovel-init`                 | 深度初始化    | 充分性闸门 6 条 + 用户拍板          | worldbuilding/ + creativity/ + genre-tropes |
| `/webnovel-plan <卷>`             | 卷纲规划     | 冲突阻断（BLOCKER）             | outlining/ + 大纲模板                           |
| `/webnovel-write <章>`            | 写章流水线    | 充分性闸门 7 条 + 三道 write-gate | polish/anti-ai/style + writing/             |
| `/webnovel-review <范围>`          | 章节审查     | blocking 用户裁决             | core-constraints + review-schema            |
| `/webnovel-query <词>`            | 状态查询（只读） | 真源优先级定位                   | system-data-flow + advanced/foreshadowing   |
| `/webnovel-learn "<经验>"`         | 经验沉淀     | 只追加不删除                    | 无                                           |
| `/webnovel-dashboard`            | 只读面板     | 只读 + localhost 安全         | 无                                           |
| `/webnovel-doctor [--chapter N]` | 项目体检     | 只读、不修复                    | 无                                           |

## 2. /webnovel-init —— 深度初始化

**一句话**：分阶段问答收集创作信息 → 通过"充分性闸门" → 执行 `webnovel.py init` 生成项目骨架与 Story System 合同种子。

### 阶段流程

| 步骤       | 内容       | 要点                                               |
| -------- | -------- | ------------------------------------------------ |
| Step 1   | 预检与上下文加载 | 校验入口脚本、加载最小参考、输出"已知/待收集清单"                       |
| Step 1.5 | 灵感来源（可选） | 拆书必须走 Agent 调 `deconstruction-agent`，用户确认后才可变形采用 |
| Step 2   | 故事核与商业定位 | 书名/题材（支持 A+B 复合）/规模/一句话故事/核心冲突                   |
| Step 3   | 角色骨架     | 主角欲望-缺陷-结构、感情线、反派分层与镜像对抗                         |
| Step 4   | 金手指      | 类型/风格/可见度/不可逆代价（必须有代价）/成长节奏                      |
| Step 5   | 世界观与力量规则 | 世界规模/力量体系/势力/社会阶层                                |
| Step 6   | 创意约束包    | 反套路库 ≤2 个 → 2-3 套创意包（卖点+规则+硬约束）→ 五维评分            |
| Step 7   | 一致性复述    | 初始化摘要草案（故事核/主角核/金手指核/世界核/创意约束核）让用户拍板             |

**充分性闸门（6 条）** → 执行生成（`init` + idea\_bank + Patch 总纲 + `story-system` 生成 MASTER\_SETTING）→ 验证与交付（state/设定集/总纲/idea\_bank/MASTER\_SETTING 存在性）→ 作者友好三段式最终报告。

## 3. /webnovel-plan —— 卷纲规划

**一句话**：基于总纲增量补齐设定 → 卷节拍表 → 卷时间线 → 卷纲 → 批量章纲 → 设定写回 → 刷新 Story System 合同。

**优先级链**：用户 > 总纲 > 时间线 > 默认流程 > reference。

10 步流程要点：

1. 加载 state.json/总纲/genre + 跨卷状态（近 5 章摘要、query-entity-state、get-open-loops）
2. 增量补齐设定基线（世界观/力量体系/主角卡/反派）
3. 选卷确认范围
4. **卷节拍表**：中段反转必填、危机链 ≥3 次递增（Promise → Catalyst → 危机链 → 中段反转 → All Is Lost → Payoff+新钩子）
5. **卷时间线表**：时间体系、D-N 倒计时
6. **卷纲骨架**：Strand 分布、爽点密度、伏笔规划 + 跨卷一致性检查
7. **批量章纲**（默认 10 章/批，范围 8-12）：每章含目标/阻力/代价/时间锚点/爽点/钩子 + 结构化节点 **CBN×1、CPNs×2\~4、CEN×1**；相邻章 CEN→CBN 必须逻辑承接
8. 新增设定写回设定集（冲突标 `BLOCKER` 阻断等裁决）
9. 验证 + `第{N}卷-总纲写回.json` + `master-outline-sync` + `update-state`
10. 刷新合同：`story-system --chapter --persist --emit-runtime-contracts`（禁止占位 query）

## 4. /webnovel-write —— 写章流水线（核心）

**一句话**：产出可发布章节，完整执行"上下文 → 起草 → 审查 → 润色 → 提交 → 备份"。

参数：`[章号] [--fast|--minimal]`（`--fast` 轻量审查；`--minimal` 跳过审查写 no-review artifact）。

### 完整步骤链

| 阶段     | 动作                                                                                                                                              | 关键校验                                                                      |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| 准备     | `preflight` → `where` 解析 PROJECT\_ROOT → `placeholder-scan`                                                                                     | 项目健康 + 无占位残留                                                              |
| 准备     | 从详细大纲解析真实 `CHAPTER_GOAL` → `story-system --chapter --persist --emit-runtime-contracts` → **prewrite write-gate**                                | 必备 MASTER\_SETTING/volume/chapter.review，缺失即阻断                            |
| Step 1 | **Agent 调 context-agent** 生成写作任务书                                                                                                               | 固定五段排序，记录 SubagentRun                                                     |
| Step 2 | 起草正文                                                                                                                                            | 只依据任务书，纯正文无占位符，围绕 CBN→CPNs→CEN                                            |
| Step 3 | **Agent 调 reviewer**（只返回严格 JSON）→ 主流程 Write 落盘 → `review-pipeline` 标准化                                                                          | blocking 定点修复或用户裁决，只跑一轮                                                   |
| Step 4 | 润色：修复非 blocking → 风格适配 → 排版 → Anti-AI 终检                                                                                                        | 只改表达不改事实；`anti_ai_force_check=fail` 不进 Step 5                             |
| Step 5 | 5.1 Agent 调 **data-agent** 产出三份 artifact → 5.2 **precommit write-gate** + 只读 git diff → `chapter-commit` → 5.3 **postcommit write-gate** 验证五路投影 | blocking>0 / missed\_nodes / pending → rejected；失败隔离用 `projections retry` |
| Step 6 | Git 备份：`backup --chapter`                                                                                                                       | 以 PROJECT\_ROOT 为准，禁止裸全量 add                                              |

**充分性闸门（7 条）**：正文非空、审查落库、blocking 已处理、Anti-AI pass、accepted commit + 投影完成、chapter\_status=committed、三道 write-gate 均通过。另有 run-ledger 断点续跑（重复执行从失败点继续）。

### 写作任务书五段结构（context-agent 输出）

① 开篇委托（书名/章号/标题/一句话目标）→ ② 这章的故事（摘要、目标/阻力、CBN/CPNs/CEN、必须覆盖/禁区、RAG 线索）→ ③ 这章的人物（状态/驱动力/说话倾向）→ ④ 怎么写更顺（风格/节奏裁决的自然语言指导）→ ⑤ 收在哪里（结尾感觉与未完感）。

## 5. /webnovel-review —— 章节审查

流程 8 步：解析项目根（缺 state.json 阻断）→ 缺合同先刷新 runtime 合同 → 按需加载参考 → 加载投影状态与正文 → **Agent 调统一 reviewer**（reviewer 只返回 JSON，主流程写盘）→ `review-pipeline --save-metrics` 生成报告并落库 → `update-state --add-review` 写兼容审查记录 → `blocking=true` 时 AskUserQuestion 裁决（立即修复 / 保存报告稍后处理）。

**红线**：禁止主流程伪造结论或 `overall_score`；报告与 metrics 只由 review-pipeline 产出。

## 6. /webnovel-query —— 状态查询（只读）

识别查询类型 → 按数据源优先级定位真源（写前真源 `.story-system/` → 写后真源 accepted commit → 投影层 memory-contract/state.json/index.db）→ 调用最窄工具（query-entity-state / query-relationships / query-rules / get-open-loops；静态设定直接 Grep+Read）→ 按模板输出（概要/详细信息/一致性检查）。同时加载 reference 不超过 2 个。

## 7. /webnovel-learn —— 经验沉淀

项目根 guard → 读 `progress.current_chapter` → 归类 `pattern_type`（hook/pacing/dialogue/payoff/emotion/format/other）→ 必须 `project-memory add-pattern` 命令写入（禁止手写 JSON）。约束：只追加不删除；同 pattern\_type+description 跳过；损坏 JSON 不写脏数据。

## 8. /webnovel-dashboard —— 只读面板

确认 dashboard 模块 → 解析项目根 → 校验 `frontend/dist/index.html` 构建产物 → `python -m dashboard.server --project-root` 启动 → 验证 `/api/story-runtime/health`。安全边界：纯只读、文件访问限 PROJECT\_ROOT、默认仅监听 localhost。不默认安装依赖。

## 9. /webnovel-doctor —— 项目体检

`project-status --format summary` 短状态 → `doctor` 阶段感知检查（按 runtime 推导的项目阶段解释缺失项，不把新 init 项目按老项目检查）。只读、不自动修复、不装依赖、不展示 API Key。

## 10. SKILL.md 文件约定

- frontmatter 含 `name` / `description`（触发词）/ `allowed-tools`（限制子代理可用工具面）

- 正文 = 步骤链 + **充分性闸门**（满足才继续）+ 输出格式契约 + **作者友好报告契约**（已完成/部分完成/需要你处理/未完成 四态）

- 失败处理：最小回滚、可信断点（run\_ledger）、三类报告（产生文件/异常耗时/下一步建议）

