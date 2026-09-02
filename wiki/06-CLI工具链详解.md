# 06 · CLI 工具链详解

## 1. 统一入口架构

所有命令行工具统一从 `scripts/webnovel.py` 进入：

```bash
python -X utf8 "<CLAUDE_PLUGIN_ROOT>/scripts/webnovel.py" --project-root "<PROJECT_ROOT>" <子命令> [参数]
```

### 两层结构

```text
scripts/webnovel.py                 # 薄封装（约 30 行）
└── data_modules/webnovel.py        # 真正的 argparse CLI（main()，约 300 行）
    ├── 内置 func 处理器           # cmd_where / cmd_preflight / cmd_doctor ...（本文件实现依赖 data_modules）
    ├── _run_data_module(...)       # 进程内 importlib 调用 data_modules.<module>.main()
    └── _run_script(...)            # subprocess 运行 scripts/ 根目录脚本
```

关键辅助函数：

| 函数 | 作用 |
|------|------|
| `_resolve_root()` / `resolve_project_root()` | 解析书项目根（须含 `.webnovel/state.json`） |
| `_run_data_module(module, argv)` | 进程内调用 `data_modules.<module>.main()` |
| `_run_script(script_name, argv)` | 子进程调 scripts/ 根目录脚本 |
| `_strip_project_root_args()` / `_passthrough_tail()` / `PASSTHROUGH_TOOLS` | 透传参数裁剪，避免下游重复收到 `--project-root` |

## 2. 子命令 → 处理模块映射总表

### A. 内置 func 处理器

| 子命令 | 处理函数 | 实际处理模块 |
|--------|----------|--------------|
| `where` | `cmd_where` | project_locator |
| `use <root>` | `cmd_use` | project_locator（写 workspace 指针 + 全局 registry） |
| `preflight` | `cmd_preflight` | story_runtime_health |
| `project-status` | `cmd_project_status` | data_modules.project_status |
| `doctor` | `cmd_doctor` | data_modules.doctor |
| `write-gate` | `cmd_write_gate` | data_modules.write_gates（prewrite/precommit/postcommit） |
| `projections retry\|replay` | `cmd_projections` | data_modules.projections |
| `user-report` | `cmd_user_report` | data_modules.user_report |
| `run-ledger record-write-step\|write-resume` | `cmd_run_ledger` | data_modules.run_ledger |
| `run-log` | `cmd_run_log` | data_modules.run_logger |
| `knowledge query-entity-state\|query-relationships` | 内联 | data_modules.knowledge_query |
| `placeholder-scan` | 内联转发 | data_modules.placeholder_scanner |

### B. 透传到 data_modules（自动注入 --project-root）

| 子命令 | 目标模块 main() | 说明 |
|--------|----------------|------|
| `index` | index_manager | index.db 管理（核心实体/别名/出场/统计） |
| `state` | state_manager | state.json 读写 |
| `rag` | rag_adapter | 向量索引与检索（stats / index-chapter / search） |
| `style` | style_sampler | 风格采样 |
| `entity` | entity_linker | 实体消歧（register-alias / lookup / lookup-all） |
| `context` | context_manager | 上下文包构建（16 段合同 v3） |
| `memory` | memory.store | 长期记忆（stats / query / dump / conflicts / update / bootstrap） |
| `migrate` | migrate_state_to_sqlite | state.json → SQLite 迁移 |

### C. 透传到 scripts/ 根目录脚本（子进程，不带 / 带 project-root）

| 子命令 | 目标脚本 | 说明 |
|--------|----------|------|
| `init` | init_project.py | **唯一不注入 project_root** 的命令 |
| `status` | status_reporter.py | 宏观健康报告 |
| `update-state` | update_state.py | state.json 结构化更新 |
| `backup` | backup_manager.py | Git 备份/回滚/diff |
| `archive` | archive_manager.py | 归档 |
| `extract-context` | extract_chapter_context.py | 章节上下文提取 |
| `story-system` | story_system.py | 合同种子 + 运行时合同 |
| `story-events` | story_events.py | 事件查询/健康 |
| `chapter-commit` | chapter_commit.py | 章节提交 |
| `memory-contract` | memory_cli.py | 记忆合同 |
| `project-memory` | project_memory.py | 项目经验写入 |
| `review-pipeline` | review_pipeline.py | 审查标准化 + 落库 |
| `master-outline-sync` | update_master_outline.py | 总纲写回 |

## 3. 核心子命令详解

### 3.1 preflight（预检）

- 输出：插件路径 / 项目根 / Story System 健康（`mainline_ready`）
- 依赖：`story_runtime_health.build_story_runtime_health`

### 3.2 doctor（体检）

`doctor [--chapter N] [--deep] [--format json|text]`

- 检查构造器（`data_modules/doctor.py`）：`_preflight_checks` / `_file_checks` / `_json_checks` / `_sqlite_checks` / `_rag_checks` / `_projection_log_checks` / `_python_checks` / `_dashboard_checks`
- 输出四态：ok / warning / error / skipped；阶段感知（依赖 `project_phase.ProjectPhaseSnapshot`）

### 3.3 write-gate（三道关卡）

`write-gate <prewrite|precommit|postcommit> --chapter N`

### 3.4 projections（投影补跑/重放）

```bash
webnovel.py projections retry  --chapter N
webnovel.py projections replay --start N --end M
```

实现：读出已持久化的 commit JSON → `ChapterCommitService.apply_projection_writers()`，修复失败/遗漏投影。

### 3.5 story-system（合同生成）

```bash
webnovel.py story-system --query "退婚流+都市异能" --genre 都市异能 --chapter 5 --persist --emit-runtime-contracts
```

- 内部：`StorySystemEngine.build()`（题材路由 + CSV 检索 + 裁决注入 + 毒点排序）→ `persist` 写种子 → `emit-runtime-contracts` 生成 volume/review 合同

### 3.6 chapter-commit（章节提交）

```bash
webnovel.py chapter-commit --chapter 45 --fulfillment result.json --disambiguation d.json --extraction e.json [--review r.json]
```

内部：`ChapterCommitService.build_commit`（pydantic 校验 4 artifact）→ `persist_commit` → 事件双写 + AmendProposal → `apply_projections`。

### 3.7 run-ledger（断点续跑账本）

```bash
webnovel.py run-ledger record-write-step --chapter N --step draft --status done ...
webnovel.py run-ledger write-resume --chapter N
```

`build_write_resume_plan`：结合 commit 状态 / 投影日志 / 备份存在性（含文件签名校验）生成各步骤续跑建议。

## 4. 项目根定位机制（project_locator.py）

解析顺序（`resolve_project_root(explicit=None, *, cwd=None)`）：

1. 显式参数 `--project-root`
2. 向上候选目录（判定条件 `_is_project_root` = 存在 `.webnovel/state.json`）
3. 工作区指针（`.claude/.webnovel-current-project`，由 `use` / init 写入）
4. 全局 registry（用户级）

绑定"当前项目"：`write_current_project_pointer()`（工作区）+ `update_global_registry_current_project()`（用户级）双写。

## 5. 运行日志与脱敏

- `run-log` → `run_logger.py`：写入 `.webnovel/logs/run_last.log`、`redact_text()/redact_payload()` 正则脱敏 api_key/secret/token/password/credential
- `observability.py`：`safe_log_tool_call()` / `safe_append_perf_timing()` 永不抛异常的工具统计

## 6. 使用建议

- 写章链路 5 步：`preflight` → `story-system` → `write-gate prewrite` → `chapter-commit` → `write-gate postcommit` → `backup`
- 排查链路：`preflight` → `doctor --format text` → `projections retry` → `run-ledger write-resume`
- 只读查询：`status` / `knowledge` / `placeholder-scan` / `story-events --health`