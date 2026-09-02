# 08 · 记忆与 RAG 检索详解

## 1. 子系统总览

```text
写前注入（读路径）                        写后沉淀（写路径）
┌──────────────────────┐              ┌──────────────────────────┐
│ MemoryOrchestrator   │              │ MemoryWriter             │
│ .build_memory_pack() │              │ .update_from_chapter_result│
└────────┬─────────────┘              │ .apply_commit_projection │
         │ 注入                        └────────────┬─────────────┘
         ▼                                          │ 章节结果 / commit
┌──────────────┐        RAG             ┌───────────▼──────────┐
│ context-agent │ ◄── 截图 ── 写前上下文  │ ScratchpadManager    │
└──────────────┘        + BM25          │ (自动压缩超过 500 项) │
                                        └───────────┬──────────┘
┌──────────────┐  查 询                        ┌─────▼──────┐
│ knowledge_query│◄────────── index.db ◄──────  vectors.db/BM25
└──────────────┘                              └────────────┘
```

## 2. 长期记忆系统（memory/ 子包）

### 2.1 数据模型（schema.py）

- 层级 `VALID_LAYERS`：`semantic`（语义/常识）/ `episodic`（情景/事件）
- 状态 `VALID_STATUSES`：`active` / `outdated` / `contradicted` / `tentative`
- `CATEGORY_TO_BUCKET`：7 类记忆 → 7 个桶

| 类别 | 桶 | 去重 key |
|------|-----|----------|
| character_state | `character_states` | `(subject, field)` |
| story_fact | `story_facts` | `(subject, field)` |
| world_rule | `world_rules` | `(subject, field)` |
| timeline | `timeline_events` | `(subject, field)` |
| open_loop | `open_loops` | `(subject,)` |
| reader_promise | `reader_promises` | `(subject, field)` |
| relationship | `relationships` | `(subject, target)` |

- `MemoryItem`（dataclass）：`id/layer/category/subject/field/value/payload/status/source_chapter/evidence/updated_at`；方法 `normalized()/to_dict()/from_dict()`
- `ScratchpadData`：7 桶列表 + meta；`count_items()`

### 2.2 持久化与查询（store.py）

`ScratchpadManager`：

| 方法 | 说明 |
|------|------|
| `load()/save(data)` | 读写 `memory_scratchpad.json`；save 超过 `memory_compactor_threshold`(默认 500) 自动触发压缩 |
| `upsert_item(item)` | **核心写入**：同 key 旧值降级 `outdated` 保留审计轨迹；返回 `{added, updated, outdated}` |
| `mark_status(item_id, status)` | 按 ID 改状态 |
| `query(category, subject, status="active")` | 条件查询 |
| `conflicts()` | 检测同类同 key 多条 active（矛盾检测） |

### 2.3 写后沉淀（writer.py）

`MemoryWriter`：
- `_item_id()`：sha256 前 16 位稳定 ID
- `update_from_chapter_result(chapter, result)`：**Stage 2 零成本结构化映射**——`state_changes→character_state`、`entities_new→first_seen`、`relationships_new→relationship`、`chapter_meta.hook→story_fact`；随后 **Stage 4** 深度提取（timeline/ world_rule / open_loop / reader_promise）
- `_coerce_loop_content(payload, event)`：open_loop 多候选字段兜底
- `apply_commit_projection(commit_payload)`：commit 投影入口——entity_deltas / state_deltas / accepted_events（`world_rule_revealed/broken→world_rules`、`open_loop_created→open_loops`、`promise_*→reader_promises`）→ 复用 update_from_chapter_result

### 2.4 写前注入（orchestrator.py）

`MemoryOrchestrator.build_memory_pack(chapter, task_type="write")` 返回：

| 段 | 内容 |
|----|------|
| `working_memory` | 本章 outline + 近 3 章摘要 + state 导出（protagonist/plot_threads/disambiguation_pending） |
| `episodic_memory` | index.db 近期 state_changes / relationships / appearances（章节倒序） |
| `semantic_memory` / `long_term_facts` | 相关性过滤 + 预算裁剪后的语义记忆 |
| `active_constraints` | `world_rule` + `open_loop`（硬约束） |
| `warnings` | `memory_conflict`（来自 store.conflicts()） |

- `_filter_relevant(items, chapter, outline)`：subject/field/value 前 20 字符命中 outline，或 source_chapter 在窗口内（默认 20 章）；按 `PRIORITY`（world_rule=0 最高）+ 新近度排序
- 预算：`budget.allocate_limits`（write 30 项 45/30/25；review 40 项 35/35/30；query 25 项 30/45/25 的 working/episodic/semantic 配比）

### 2.5 压缩与冷启动

- `compactor.py` `compact_scratchpad(data, max_items=500)` 四步：① outdated 保留最新 ② 清理已回收伏笔（status ∈ resolved/closed/done/paid_off/payoff）③ 距最新章节 50 章以上的旧 timeline 合并为 `timeline_summary` ④ 超限按 status+source_chapter+updated_at 截断
- `bootstrap.py` `bootstrap_from_index(config)`：从 index.db 回填——实体 current_json → character_state；state_changes 历史 → outdated/active；relationships → relationship；`summaries/*.md` 伏笔区块 → open_loop

## 3. RAG 检索链路

### 3.1 存储结构（rag_adapter.py）

`vectors.db` 三表 + schema meta：

| 表 | 列/用途 |
|----|---------|
| `vectors` | chunk_id / embedding BLOB(float32) / parent_chunk_id / chunk_type / metadata |
| `bm25_index` | term → chunk_id 倒排 |
| `doc_stats` | 文档统计 |
| `rag_schema_meta` | schema_version="2"；迁移前自动备份、失败回滚（`_backup_vector_db` / `_restore_vector_db_from_backup`） |

chunk_id 约定：summary 为 `chNNNN_summary`（父块）、scene 为 `chNNNN_sN`（子块）。

### 3.2 检索策略与多级降级

```text
strategy="auto" → QueryRouter.route_intent
  ├─ needs_graph=true  → graph_hybrid_search（图谱扩展：种子实体 → 关系扩展 → 候选块打分 → 向量精算 → 先验加分 → rerank）
  └─ 否则              → hybrid_search（向量 + BM25 → RRF 融合 → rerank 精排，失败回退 RRF）
降级条件：
  - 无 embed_api_key / 嵌入失败（嵌入失败的 chunk 仅保留 BM25 索引）
  - Embedding API 401 → DEGRADED_MODE 显式告警 + vector_search 返回空
  - 统一入口 search() 未知策略 → 降级 hybrid
```

### 3.3 关键类与函数（api_client.py）

| 类 | 关键方法 | 说明 |
|----|----------|------|
| `EmbeddingAPIClient` | `embed(texts)` / `embed_batch(texts)` / `warmup()` | 重试+指数退避（429/500/502/503/504）；`last_error_status`（401 → 降级依据）；支持 `openai`/`modal` 两种 API 类型 |
| `RerankAPIClient` | `rerank(query, documents, top_n)` | Jina/Cohere/DashScope 原生（`qwen3-vl-rerank`/`gte-rerank-v2`）/Modal；兼容 `output.results` 与 `results` 返回 |
| `ModalAPIClient` | 统一门面（向后兼容） | 组合 embed+rerank；`warmup()`；`stats` |
| `get_client(config)` | 全局单例工厂 | —— |

### 3.4 业务检索（rag_adapter.py）

| 方法 | 说明 |
|------|------|
| `store_chunks(chunks)` | embed_batch → INSERT OR REPLACE + 更新 BM25；失败 chunk 跳过向量仅 BM25 |
| `vector_search()` | 查询向量 + 全表余弦；`chunk_type` / `chapter<=` 防剧透过滤 |
| `bm25_search()` | k1=1.5, b=0.75；中文按字、英文按词分词 |
| `hybrid_search()` | RRF 融合（rrf_k）→ 候选 2×rerank_top_n → rerank；小规模全表扫，大规模 BM25 预筛 |
| `graph_hybrid_search()` | `_extract_query_seed_entities`（别名/ID 匹配）/ `_expand_related_entities`（graph_rag_expand_hops 跳）/ `_collect_graph_candidate_chunk_ids` / `_apply_graph_priors` |
| `search_with_backtrack()` | 子块命中回溯父块合并 |
| `search()` | 统一入口 |

观测：`_log_query` → `IndexManager.log_rag_query`（hit_sources/latency）。

### 3.5 查询路由（query_router.py）

`QueryRouter`：5 类意图正则（relationship/entity/scene/setting/plot）；`route_intent(query)` → `{intent, entities, time_scope, needs_graph, raw_query}`；`plan_subqueries(intent_payload)` 生成子查询计划；`_extract_entities()` 2-6 字中文短语 + 停用词过滤（≤4 个）；`_extract_time_scope()` 解析"第N章"。

### 3.6 章节时点知识查询（knowledge_query.py）

- `entity_state_at_chapter(entity_id, chapter)`：按 state_changes 升序重放 new_value，反推实体在指定章节时点状态
- `entity_relationships_at_chapter(entity_id, chapter)`：该章节前的关系事件，按实体对去重取最新

## 4. 写前上下文组装

### 4.1 上下文包（context_manager.py）

`ContextManager.build_context(chapter, template, max_chars)`：`_build_pack` → `ContextRanker.rank_pack` → `_assemble_json_payload`（按模板权重过滤，写入 `context_contract_version=v3`）。

`SECTION_ORDER` 16 段：`core → story_contract → runtime_status → latest_commit → prewrite_validation → scene → global → reader_signal → genre_profile → writing_guidance → plot_structure → story_skeleton → memory → long_term_memory → preferences → alerts`

| 段 | 来源 |
|----|------|
| `core` | outline（≤1500 字）+ protagonist_snapshot + 近窗口摘要/meta（可被记忆包 working_memory 覆盖） |
| `scene` | 位置 + 近期出场（`filter_invalid_items` 剔除 confirmed 无效，pending 打警告；`apply_confidence_filter`） |
| `story_contract` | master/chapter/volume/review 四合同 + anti_patterns |
| `reader_signal` | reading_power / pattern_usage / hook_type_usage / review_trend / low_score_ranges |
| `genre_profile` | 合同 primary_genre 优先，legacy profile 兜底，复合题材 `_build_composite_genre_hints` |
| `writing_guidance` | 指导条目 + checklist + `_compute_writing_checklist_score`（0.5×加权完成率+0.3×必需项+0.2×总完成率±10%）并 `_persist_writing_checklist_score` |
| `story_skeleton` | 每 `context_story_skeleton_interval` 章采样远期摘要 |

阶段判定：`_resolve_context_stage`（early ≤30 / mid / late ≥120）+ `_resolve_template_weights`。

### 4.2 权重与重排（context_weights.py / context_ranker.py）

- `TEMPLATE_WEIGHTS`：plot/battle/emotion/transition 模板的 core/scene/global 权重；`TEMPLATE_WEIGHTS_DYNAMIC_DEFAULT`：early/mid/late 动态覆盖
- `ContextRanker.rank_pack`：分数 = recency(1/(1+章差)) + frequency(log 缩放) − length + hook_bonus（悬念/钩子/反转/冲突/？）+ alert_bonus（critical/关键词 +0.3）；debug 时附加 `_context_score` 字段

### 4.3 章节上下文 CLI（extract_chapter_context.py）

`build_chapter_context_payload(project_root, chapter_num)`：outline + 前两章摘要 + state 摘要 + `ContextManager` 合同上下文 + plot_structure + rag_assist（`_RAG_TRIGGER_KEYWORDS` 触发才检索；`_search_with_rag` 无 key 直接 BM25）。

### 4.4 写作指导（writing_guidance_builder.py）

- `build_methodology_strategy_card`：chapter_stage（按 chapter%5 分 build_up/confront/release）+ emotion_anchor + long_arc_controls + observability 三指标
- `build_writing_checklist`：8 类条目（fix_low_score_range/hook_diversification/coolpoint_combo/readability_loop/genre_anchor_consistency + methodology + fallback），带 id/label/weight/required/source/verify_hint
- `is_checklist_item_completed(item, reader_signal)`：hook_diversification 需钩子类型数 ≥2 等

## 5. 记忆闭环示意

```text
写前： context-agent ← memory-contract load-context（含 MemoryOrchestrator pack）
写中： CSS/CSV 检索（reference_search BM25）按需触发
写后： data-agent 三 artifact → chapter-commit → MemoryProjectionWriter
       → MemoryWriter.apply_commit_projection（零成本映射 + 深度提取）
       → ScratchpadManager.upsert（旧值 outdated，超阈值自动压缩）
       → 下一次写前的 MemoryOrchestrator 注入（闭环完成）
```