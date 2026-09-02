# 09 · Dashboard 详解

> 只读可视化面板：FastAPI 后端 + React 前端。位于 `webnovel-writer/dashboard/`，默认监听 `127.0.0.1:8765`（仅 localhost）。

## 1. 启动链路

```text
命令行：python -m dashboard
       python -m dashboard.server --project-root <PROJECT_ROOT>
       （--host 默认 127.0.0.1，--port 默认 8765，--no-browser 禁止自动打开浏览器）
入口：__main__.py → server.main() → app.create_app() → uvicorn.run()
```

**项目根解析优先级**（`server._resolve_project_root()`）：

```text
--project-root 参数 > 环境变量 WEBNOVEL_PROJECT_ROOT > .claude/.webnovel-current-project 指针 > 当前目录（需含 .webnovel/state.json，否则退出码 1）
```

依赖：`fastapi / uvicorn[standard] / watchdog / httpx`。

## 2. 后端架构（app.py）

- 全局模块级状态：`_project_root` + `_watcher`（FileWatcher 单例）
- `_ensure_scripts_dir_on_path()`：把 `scripts/` 插入 sys.path，延迟导入 `data_modules.*`
- lifespan：存在 `.webnovel/` 或 `.story-system/` 时启动 watcher，关闭时 stop
- CORS：只放行 GET 方法，`allow_origins` 限 localhost/127.0.0.1（80/5173/8000）

### 2.1 API 路由总表（全部 GET，严格只读，30 个）

**项目与运行态**

| # | 路径 | 处理函数 | 说明 |
|---|------|----------|------|
| 1 | `/api/project/info` | `project_info` | 返回 state.json 全文（缺失 404） |
| 2 | `/api/story-runtime/health` | `story_runtime_health` | 委托 `story_runtime_health.build_story_runtime_health()` |
| 3 | `/api/env-status` | `env_status` | RAG 环境：embed/rerank key、向量库、`rag_mode`（full/embed_only/bm25_only） |
| 4 | `/api/env-status/probe` | `env_status_probe` | 一次性 4 项诊断（embed/rerank/vector_db/story_runtime）+ ok 汇总 |

**实体与关系（index.db）**

| # | 路径 | 说明 |
|---|------|------|
| 5 | `/api/entities` | 实体列表（`type`、`include_archived`，按 last_appearance 倒序） |
| 6 | `/api/entities/{entity_id}` | 实体详情 |
| 7 | `/api/relationships` | 关系（`entity`、`limit=200`） |
| 8 | `/api/relationship-events` | 关系事件（`entity`、`from_chapter`、`to_chapter`） |
| 9 | `/api/aliases` | 别名（`entity`） |
| 10 | `/api/state-changes` | 实体状态变更（`entity`、`limit=100`） |

**章节与指标**

| # | 路径 | 说明 |
|---|------|------|
| 11 | `/api/chapters` | 章节列表（characters 字段 JSON 归一化） |
| 12 | `/api/scenes` | 场景列表（`chapter`、`limit=500`） |
| 13 | `/api/reading-power` | 章节阅读力（`limit=50`） |
| 14 | `/api/review-metrics` | 审查指标（dimension_scores/severity_counts/critical_issues 归一化） |
| 15 | `/api/stats/chapter-trend` | **章节趋势大视图**：LEFT JOIN reading_power/review_metrics + `hook_strength_value`（weak=1/medium=3/strong=5）+ strand（`_build_strand_map`）+ volume（`_resolve_volume_for_chapter`）；`limit(1-500)=50`、`offset` |
| 16 | `/api/checklist-scores` | 写作检查清单得分 |

**Story System 与投影**

| # | 路径 | 说明 |
|---|------|------|
| 17 | `/api/commits` | 遍历 `.story-system/commits/*.commit.json` + 投影状态（投影日志优先，回退 commit 内嵌；`limit=20`） |
| 18 | `/api/contracts/summary` | 合同树概览（master/volume/chapter/review/commit 计数 + 当前章存在性） |
| 19 | `/api/story-events` | story_events 表（payload_json 解析；`chapter`、`limit=200`） |
| 20 | `/api/story-events/health` | 事件数 + pending amend 提案数 + 事件文件数对账 |

**追读力与可观测性**

| # | 路径 | 说明 |
|---|------|------|
| 21 | `/api/overrides` | override_contracts（`status`、`limit=100`） |
| 22 | `/api/debts` | chase_debt（`status`、`limit=100`） |
| 23 | `/api/debt-events` | debt_events（`debt_id`、`limit=200`） |
| 24 | `/api/invalid-facts` | 无效事实（`status`、`limit=100`） |
| 25 | `/api/rag-queries` | RAG 查询日志（`query_type`、`limit=100`） |
| 26 | `/api/tool-stats` | 工具调用统计（`tool_name`、`limit=200`） |

**文件与实时**

| # | 路径 | 说明 |
|---|------|------|
| 27 | `/api/files/tree` | 正文/大纲/设定集 三目录递归树 |
| 28 | `/api/files/read` | 只读读文件（`path`；safe_resolve 防穿越 + 三目录白名单 + 2MB 上限） |
| 29 | `/api/events` | **SSE 端点**：StreamingResponse 持续推送文件变更 |
| 30 | `/{full_path:path}` | SPA fallback：非 api/ 一律回 `frontend/dist/index.html`；`/assets` 静态托管 |

**辅助函数**：`_get_db()`（只读打开）、`_fetchall_safe()`（表/列缺失返回空，兼容旧库）、`_load_state_payload()`、`_parse_json_value()`、`_build_env_status()`、`_inspect_vector_db()`。

## 3. watcher.py —— SSE 实时刷新

| 组件 | 说明 |
|------|------|
| `_WebnovelFileHandler` | `.webnovel/` 只关注 `{state.json, index.db, workflow_state.json}`；`.story-system/` 关注所有 `.json`；响应 on_modified/on_created |
| `FileWatcher._on_change` | watchdog 线程 → `loop.call_soon_threadsafe` → asyncio 主循环 |
| `subscribe() / unsubscribe() / _dispatch` | 每客户端一个 `asyncio.Queue(maxsize=64)`，队列满（慢消费者）即移除 |
| `start() / stop()` | `.webnovel` recursive=False、`.story-system` recursive=True，stop 时 join(3s) |

**消费闭环**：`/api/events` 推 `data: {json}` → 前端 `subscribeSSE` 收到后 `refreshToken+1` → 所有页面 useEffect([refreshToken]) 重拉数据。

## 4. 安全设计（只读 + 纵深防御）

| 层 | 机制 |
|----|------|
| 全局 | CORS 仅 GET；无任何写 API |
| 路由层 | 全部 SELECT / read_text |
| `/api/files/read` | `safe_resolve`（`path_guard.py`）：resolve() 吃 `..` 符号链接 + `relative_to(project_root)` 强制子路径；三目录白名单 `正文/大纲/设定集`；2MB 上限（413）；非 UTF-8 回退占位 |
| 网络 | 默认 `127.0.0.1`，不暴露局域网 |

## 5. 前端（frontend/src/）

技术栈：React 19 + react-router-dom 7 + ECharts 5（echarts-for-react）+ Vite 6，像素风主题，6 页面全部 `React.lazy` 按需分包。

### 5.1 路由与全局状态（App.jsx）

| 路径 | 页面 | 侧边栏 |
|------|------|--------|
| `/` | OverviewPage | 总览 |
| `/characters` | CharactersPage | 角色图鉴 |
| `/pacing` | PacingPage | 节奏雷达 |
| `/foreshadowing` | ForeshadowingPage | 伏笔追踪 |
| `/files` | FilesPage | 文档浏览 |
| `/system` | SystemPage | 系统状态 |

全局状态：`projectInfo`（/api/project/info）、`refreshToken`（SSE 计数器）、`connected`（底部"实时同步中/断开"指示灯）；经 `Outlet context`（`useDashboardContext()`）下发。

### 5.2 页面数据与图表

| 页面 | 数据源 | 展示 |
|------|--------|------|
| Overview | project/info、runtime/health、chapters、chapter-trend | 5 统计卡；审查得分趋势折线（50 章窗口 + Pager）；按卷字数柱状图；Strand 分布饼图（quest/fire/constellation）；紧急伏笔 Top5；近 3 章概要 |
| Characters | entities、relationships(1000)、relationship-events(5000)、state-changes | 双 Tab：实体列表（类型过滤 + 详情卡 + 状态变化历史）；**关系图谱**（ECharts force 力导向图，1~最新章时间轴滑块 + 播放每 120ms+5 章） |
| Pacing | chapters、chapter-trend | 4 统计卡（钩子均分/过渡章/字数）；钩子强度面积折线；Strand 逐章堆叠柱状图；按卷字数箱线图 |
| Foreshadowing | 仅 project/info（纯前端计算） | 4 统计卡（总/活跃/已回收/紧急超期）；筛选；**伏笔时间线甘特图**（custom series + 当前章 markLine）；伏笔列表 |
| Files | files/tree、files/read | 三目录树（递归 TreeNodes）+ 只读预览（行数/字数徽标） |
| System | runtime/health、contracts/summary、commits(12)、env-status、probe | 4 统计卡（Runtime/Latest Commit 五路投影/RAG Mode/Vector 行数）；合同树概览表；Commit 历史表（五列投影徽标）；RAG 诊断卡（"运行诊断"按钮） |

### 5.3 组件与工具

| 文件 | 导出 | 职责 |
|------|------|------|
| `components/Badge.jsx` | `Badge({tone,...})` | 7 种色调徽标 |
| `components/ChartWrapper.jsx` | `ChartWrapper({option,...})` | 封装 echarts-for-react，统一 "pixel" 像素主题 |
| `components/DataTable.jsx` | `DataTable({columns,rows,...})` | 通用表格 + 内置分页（pageSize=8） |
| `components/Pager.jsx` | `Pager(...)` | 章节窗口翻页条（前 50/下一页/跳到最新） |
| `lib/charts.js` | `ensurePixelTheme` / `buildBoxplotData` | ECharts 按需注册（Line/Bar/Pie/Boxplot/Graph/Custom）+ 手写 quantile 箱线图 |
| `lib/foreshadowing.js` | `buildForeshadowingRecords` / `summarizeForeshadowing` | 伏笔归一化 + `urgencyScore = elapsed/(target-planted)*weight` + level 判定（resolved/overdue/urgent/active） |
| `lib/story.js` | `getLatestChapter` / `resolveVolumeForChapter` / `groupChaptersByVolume` | 卷归并（progress.volumes_planned 的 chapters_range） |
| `lib/format.js` | `formatNumber`（万）/`formatChapterLabel`/`formatDateTime` 等 | 展示格式化 |
| `api.js` | `fetchJSON` / `subscribeSSE` | `BASE=''` 同源相对路径 |

### 5.4 vite 配置与构建

```text
plugins: [react()]
server.proxy: { '/api': 'http://127.0.0.1:8765' }   # dev：5173 → 8765
build: outDir 'dist'、emptyOutDir、manualChunks 拆 react-vendor + echarts-vendor
```

> 发布包已含预构建的 `frontend/dist/`，无需本地 npm build；开发前端时：终端 1 `python -m dashboard`，终端 2 `npm run dev`。

## 6. 前后端连接方式

```text
生产模式（默认）：npm run build → dist/；python -m dashboard → uvicorn 127.0.0.1:8765
  GET /assets/* → StaticFiles(dist/assets)
  GET /*        → dist/index.html（SPA fallback）
  GET /api/*    → FastAPI 路由
  浏览器全部同源 8765（api.js BASE=''）
开发模式：vite 5173 + server.proxy /api → 8765（CORS 白名单含 5173）
```

## 7. 扩展新页面的标准路径

1. 后端：`app.py` 增 GET 路由（只读 SQL/委托 data_modules）
2. 前端：`api.js` 增 `fetchXxx` → `lib/` 增计算/格式化 → `pages/XxxPage.jsx` + `App.jsx` NAV_ITEMS + `main.jsx` 路由
3. 实时刷新：数据来自 index.db/state.json/.story-system 即可自动获得 SSE 联动