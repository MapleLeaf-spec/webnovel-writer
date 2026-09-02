# 05 · Agents 与 Hooks 详解

## 1. Agent 体系总览

位于 `webnovel-writer/agents/*.md`，共 4 个。它们不是代码，而是 Claude Code 的子代理定义（身份 + 输入 + 执行步骤 + 输出 schema + 边界）。

| Agent                  | 角色             | 被谁调用                                        | 输入                                                   | 输出                             |
| ---------------------- | -------------- | ------------------------------------------- | ---------------------------------------------------- | ------------------------------ |
| `context-agent`        | 写前 research（读） | `/webnovel-write` Step 1                    | `{chapter, project_root, storage_path, state_file}`  | 五段写作任务书（见 04 §4）               |
| `reviewer`             | 事实审查（审）        | `/webnovel-write` Step 3、`/webnovel-review` | `{chapter, chapter_file, project_root, scripts_dir}` | 五维审查 JSON                      |
| `data-agent`           | 事实提取（写）        | `/webnovel-write` Step 5.1                  | `{chapter, chapter_file, project_root}`              | 三份 commit artifact JSON        |
| `deconstruction-agent` | 拆书             | `/webnovel-init` Step 1.5                   | title/source/text/analysis\_mode/init\_goal/genre    | `init_reference_research` JSON |

### 共同边界

- 子代理**只返回结构化结果**（不落盘、不执行 chapter-commit）；写盘/执行由主流程完成

- 子代理不得直接写 state/index/summaries/memory/vectors/projection

- 结果必须经过主流程校验（pydantic / review-pipeline）后才能进入正式链路

## 2. context-agent（写前上下文压缩器）

- **身份**：先 research 再输出任务书；只返回任务书、不暴露系统术语。

- **执行步骤**：

  1. `memory-contract load-context` 取基础包（story\_contracts / recent\_summaries / urgent\_loops / active\_rules / protagonist / memory\_pack / genre\_profile\_excerpt / author\_style\_patterns / style\_contract）
  2. Read 章纲原文 → 确定卷号
  3. 按需深查（query-entity / query-rules / get-timeline / get-reader-signals）
  4. 伏笔处理（`remaining ≤ 5` 或超期必须处理，最多 5 条）
  5. 组装动机/情绪底色/可用能力/文风 → 红线校验（任一 fail 回重组）

- **数据权重**：用户要求 > 章纲原文 / `chapter_directive.goal` > MASTER\_SETTING > reasoning 裁决 > CHAPTER\_COMMIT > CSV

- **降级**：load-context 空 → `extract-context`；contracts 缺失 → 标 legacy fallback；上下文严重不足 → 返回 blocker

## 3. reviewer（章节事实审查员）

- **身份**：章节**事实审查员**，只查 5 个维度（不评分、不评价文笔、不建议情节改动、不剧透大纲、只报有 evidence 的可验证问题）。

- **五维检查**：

| 维度               | 检查重点                    |
| ---------------- | ----------------------- |
| setting（设定一致性）   | 能力 vs 境界、地点 vs 世界观、战力对比 |
| timeline（时间线）    | 时间衔接、倒计时                |
| continuity（叙事连贯） | 上章钩子回应、场景衔接             |
| character（角色一致性） | 对话风格 vs 人设、角色知识边界       |
| logic（逻辑）        | 因果、战力对比、动机合理            |

- **输出 schema（审查 JSON）**：

  - `issues[]`：`severity(critical|high|medium|low)`、`category(setting|timeline|continuity|character|logic)`、`location` / `description` / `evidence` / `fix_hint` / `blocking`

  - `issues_count` / `blocking_count` / `has_blocking`

  - `dimension_results`（5 维逐项结论）+ `summary`

- 无问题也必须显式 `pass`（强制每维度输出结论）；critical 默认 `blocking=true`。

- 注意：后端 `review_schema.py` 的 category 枚举还兼容 `pacing/other`（仅为兼容，审查不主动产出）。

## 4. data-agent（章节事实提取器）

- **身份**：从正文提取结构化信息生成 chapter-commit 所需 artifacts；本文件是三份 artifact schema 的**唯一真源**。

- **执行步骤**：

  1. A 加载：Read 正文 + `index get-core-entities / recent-appearances / get-aliases / get-by-alias`
  2. B 提取与消歧（同一轮完成、不额外调 LLM）：置信度 >0.8 自动采用、0.5-0.8 采用+warning、<0.5 待人工
  3. C 生成三份 JSON 到 `.webnovel/tmp/`
  4. D 摘要（100-150 字，带 frontmatter：chapter/time/location/characters/state\_changes/hook\_type/hook\_strength + 伏笔/承接点）与场景切片（50-100 字/场景）

- **输出格式（三份 artifact）**：

  - `fulfillment_result.json`：顶层四数组 `planned_nodes` / `covered_nodes` / `missed_nodes` / `extra_nodes`

  - `disambiguation_result.json`：顶层 `pending` 数组

  - `extraction_result.json`：顶层直接放 `accepted_events` / `state_deltas` / `entity_deltas` / `entities_appeared` / `scenes` / `summary_text`（可选 `dominant_strand`、`entities_new`）；事件类型 10 种枚举

- 每条埋设伏笔必须同步一条 `open_loop_created` 事件。

## 5. deconstruction-agent（参考书拆解）

- **身份**：把参考文本拆成**可迁移创作模式**与 init 候选，绝不把原作角色/设定/金手指/剧情事实写入新书 canon。

- **模式**：`quick`（黄金三章 → 整体结构 → 拆文报告 1-5 评分 → 2-3 个 init\_candidates） / `deep`（6 阶段：章节解析 → 黄金三章 → 逐章摘要与情节点 → 聚合 → 设定与关系抽象 → 汇总） / `auto`。

- **质量门控**：confidence ≥ 0.85、coverage 85%-95%、overlap ≤ 35%。

- **输出（`init_reference_research`）**：source / analysis\_mode / reader\_promise / opening\_hook\_patterns / cool\_point\_loops / protagonist\_patterns / antagonist\_pressure\_patterns / pacing\_notes / borrowable\_structures / do\_not\_copy / differentiation\_requirements / init\_candidates / quality / resume\_state / canon\_contamination\_warnings。

- **边界**：不写任何文件、不写 idea\_bank.json；采用需 init 主流程 + 用户确认；断点状态放 `resume_state`（**不写** **`_progress.md`**）。

## 6. Hooks 运行时防护

### 6.1 hooks.json 注册表

| 事件             | matcher                  | 脚本                       | timeout |
| -------------- | ------------------------ | ------------------------ | ------- |
| `SessionStart` | `*`                      | `session_start.py`       | 5s      |
| `PreToolUse`   | `Write\|Edit\|MultiEdit` | `guard_runtime_write.py` | 5s      |
| `PreToolUse`   | `Bash`                   | `guard_runtime_write.py` | 5s      |

### 6.2 session\_start.py —— 会话启动短状态

子进程调用 `webnovel.py project-status --format summary`（内部 4s 超时），stdout 裁剪为最多 8 行 / 1000 字符后打印。失败静默（不打断会话）。可用 `WEBNOVEL_DISABLE_SESSION_STATUS_HOOK` 禁用。

### 6.3 guard\_runtime\_write.py —— 运行时写防护

**防护对象（PROTECTED\_SUFFIXES）**：

```text
.story-system/commits/
.webnovel/index.db
.webnovel/vectors.db
.webnovel/memory_scratchpad.json
.webnovel/projection_log.jsonl
```

> 注意：`state.json` **有意不保护**——审计常需批量修复，且其有独立备份重建路径。

**Write/Edit/MultiEdit 防护**：从 tool\_input 提取 `file_path/path/filename` → 路径归一化（兼容 Windows/POSIX）→ 命中保护后缀 → 输出 `permissionDecision: deny` + exit 2，提示改用 runtime 命令。

**Bash 防护**：检测"直接投影写入"命令——保护路径 + 重定向 / `out-file` / `set-content` / `add-content` / `copy-item` / `move-item` / `python` 等写命令，或绕过统一入口直接调 `chapter_commit.py`；白名单放行 `webnovel.py` + `chapter-commit` / `projections retry|replay`。

可用 `WEBNOVEL_DISABLE_RUNTIME_GUARD_HOOK` 禁用。设计目标：**保证 commit/projection 不变量一致，强制写入走 write-gate / chapter-commit / projections 官方链路**。

## 7. 与数据链的对应关系

```text
context-agent ── 只读 ──► memory-contract load-context / extract-context（读路径）
data-agent   ── 产出 ──► .webnovel/tmp/*.json（fulfillment/disambiguation/extraction）
                        └─► chapter-commit（经由 CLI，投影由系统执行）
reviewer     ── 产出 ──► .webnovel/tmp/review_results.json ──► review-pipeline 标准化
deconstruction ── 产出 ─► init_reference_research JSON（用户确认后才变形采用）
```

