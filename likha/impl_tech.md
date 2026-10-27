# Jellyfish AI 短剧工作台 — 技术设计文档（impl_tech.md）

> 版本：v1.0 ｜ 日期：2026-10-27 ｜ 适用范围：Jellyfish（AI Short Drama Studio）
> 目标读者：架构师、后端/前端开发、SRE、测试
> 部署目标：阿里云（主区域菲律宾 Manila `ap-southeast-6`，就近服务泰国 Bangkok `ap-southeast-7`）
> SLA 目标：**99.95%**（月度不可用预算 ≤ 21.6 分钟）

---

## 目录

1. [系统概述](#1-系统概述)
2. [系统整体分层架构设计](#2-系统整体分层架构设计)
3. [代码目录结构设计](#3-代码目录结构设计)
4. [日志、监控、灰度发布、限流与回滚能力](#4-日志监控灰度发布限流与回滚能力)
5. [数据存储结构设计](#5-数据存储结构设计)
6. [核心业务逻辑时序图](#6-核心业务逻辑时序图)
7. [完整部署方案（阿里云 + Docker Compose）](#7-完整部署方案阿里云--docker-compose)
8. [SLA 99.95% 达成方案](#8-sla-9995-达成方案)
9. [容量设计与压测验收](#9-容量设计与压测验收)
10. [系统不足与改进措施](#10-系统不足与改进措施)
11. [附录](#11-附录)

---

## 1. 系统概述

Jellyfish 是一个端到端的 AI 短剧生产工作台，覆盖「剧本输入 → 章节拆分 → 分镜提取与确认 → 一致性资产管理（角色/场景/道具/服装）→ 关键帧/图片/视频生成 → 任务跟踪 → 剪辑导出」全链路。

**核心特征：**

| 维度 | 说明 |
| --- | --- |
| 前端 | React 18 + TypeScript + Vite 5 + Ant Design 5 + TailwindCSS + Zustand，OpenAPI generated client 统一调用后端 |
| 后端 | Python 3.12 + FastAPI（async）+ SQLAlchemy 2.0 + LangGraph/LangChain Agent 编排 |
| 异步任务 | Celery + Redis broker，任务真相层落 DB（`generation_tasks`），不依赖 result backend |
| 数据存储 | MySQL 9.0（生产）/ SQLite（开发），S3 兼容对象存储（开发用 RustFS，生产用阿里云 OSS） |
| AI 供应商 | OpenAI 兼容协议 + 火山引擎（文本 LLM / 图片生成 / 视频生成），供应商与模型配置存 DB |
| 部署 | Docker Compose（8 服务），CI 产出 ghcr.io 镜像；生产目标为阿里云多可用区部署 |

**业务量假设（容量规划基线）：**

- 菲律宾 + 泰国双市场，初期日活创作团队 200~500，峰值并发编辑用户 ~100；
- 单部短剧 ≈ 80~120 章节 × 每章 5~15 分镜；单分镜产生 1~4 个生成任务；
- 视频生成为长任务（1~10 分钟/个），是系统吞吐与稳定性的主要压力源；
- API QPS 峰值预估 200~400（含前端任务轮询），对象存储日增量 50~200 GB（帧图/视频）。

### 1.1 设计目标与量化指标

| 维度 | 目标值 | 校验方式 |
| --- | --- | --- |
| 可用性 SLA | ≥ **99.95%**（月度不可用预算 ≤ 21.6 分钟） | SLO 燃尽图 + `/health` 外部拨测（PH/TH 双探测点） |
| API 延迟 | 读路径 P99 ≤ 500 ms；写路径 P99 ≤ 1.5 s（不含 LLM 同步调用） | ARMS Prometheus 分位数 + 压测基线（第 9 章） |
| LLM 同步端点 | `/script_processing` 任务化后端到端 P95 ≤ 90 s（Agent 编排耗时） | task-log elapsed_ms 统计 |
| 单实例容量 | API 实例 ≥ 400 QPS；单 Celery worker ≥ 4 并发视频长任务 | k6 压测实测替换（见 9.2） |
| 任务正确性 | 任务不重复执行、不丢状态；取消收敛率 100% | task_id 幂等键 + reaper 对账（见 10.4） |
| 生成成功率 | 业务 SLO ≥ 95%（与平台 SLA 分离计量） | 按 task_kind/provider 维度指标 |
| 故障恢复 | 发布回滚 RTO ≤ 5 min；RPO ≤ 5 min（binlog）；区域灾难 RTO ≤ 4 h | 季度演练（见 8.6） |
| 区域延迟 | PH 用户 RTT ≤ 15 ms；TH 用户经 CDN/GA 后 API RTT ≤ 100 ms | 拨测 |
| 配置收敛 | 默认模型/供应商切换生效 ≤ 60 s | 灰度开关演练 |

---

## 2. 系统整体分层架构设计

### 2.1 分层架构图

```mermaid
flowchart TB
    subgraph L1["接入层 (Edge / Access)"]
        CDN["阿里云 CDN / DCDN<br/>(PH·TH 边缘节点加速)"]
        WAF["Web 应用防火墙 WAF<br/>(CC 防护 / Bot 管理)"]
        ALB["ALB 应用型负载均衡<br/>(多可用区 / TLS 终结 / 灰度分流)"]
    end

    subgraph L2["前端展现层 (Presentation)"]
        FE["React SPA (Nginx 容器)<br/>项目大厅 / 分镜编辑页 / 分镜工作室<br/>任务中心 / 资产管理 / 模型管理 / 剪辑器"]
        ENVJS["env.js 运行时配置注入<br/>window.__ENV.BACKEND_URL"]
    end

    subgraph L3["API 网关层 (API Layer)"]
        API["FastAPI /api/v1<br/>收参校验(Pydantic) · 响应组织(ApiResponse)<br/>CORS · 全局异常处理 · 限流(目标态)"]
        ROUTES["路由分组<br/>/studio /film /llm /script_processing /health"]
    end

    subgraph L4["业务服务层 (Service Layer)"]
        STUDIO["studio 服务<br/>shots/entities/generation<br/>(frame·video·asset_image)"]
        LLM_SVC["llm 服务<br/>provider_registry / resolver<br/>runtime / manage"]
        FILM["film 服务<br/>generated_video / task_status"]
        SCRIPT["script_processing<br/>LangGraph Agents 编排"]
        TASKMGR["task_manager<br/>任务状态机 / 取消 / 查询"]
    end

    subgraph L5["任务执行层 (Async Task Layer)"]
        CELERY["Celery Worker 集群<br/>(Redis Broker)"]
        EXECUTOR["AbstractWorkerTaskExecutor<br/>running→execute→apply_result<br/>→succeeded/failed/cancelled"]
        REGISTRY["task_registry<br/>(task_kind, provider_key) 双键适配器"]
    end

    subgraph L6["能力集成层 (Integration / Contracts)"]
        CONTRACTS["core/contracts<br/>ImageGenerationInput/Result<br/>VideoGenerationInput · ProviderConfig"]
        OPENAI["OpenAI 兼容适配器<br/>(文本/图片/视频)"]
        VOLC["火山引擎适配器<br/>(图片/视频/能力表)"]
    end

    subgraph L7["数据与存储层 (Data Layer)"]
        MYSQL[("MySQL 9.0<br/>projects/chapters/shots<br/>assets/tasks/providers")]
        REDIS[("Redis 7<br/>Celery Broker / 缓存 / 限流计数")]
        OSS[("S3 兼容对象存储<br/>OSS: 帧图/视频/素材文件")]
    end

    subgraph L8["可观测与治理层 (Observability & Governance)"]
        LOG["日志: SLS(结构化 JSON)"]
        METRIC["指标: Prometheus/ARMS"]
        TRACE["链路: OpenTelemetry"]
        ALERT["告警: CloudMonitor/ARMS"]
    end

    CDN --> WAF --> ALB
    ALB --> FE
    FE -.-> ENVJS
    FE --> API
    API --> ROUTES --> STUDIO & LLM_SVC & FILM & SCRIPT
    STUDIO & FILM --> TASKMGR
    TASKMGR --> REDIS
    REDIS --> CELERY --> EXECUTOR --> REGISTRY
    REGISTRY --> CONTRACTS
    CONTRACTS --> OPENAI & VOLC
    API --> MYSQL
    STUDIO --> OSS
    EXECUTOR --> MYSQL & OSS
    SCRIPT --> LLM_SVC --> OPENAI
    L3 & L4 & L5 & L6 -.-> L8
```

### 2.2 各层职责说明

| 层 | 职责 | 关键约束 |
| --- | --- | --- |
| **L1 接入层** | CDN 静态加速（PH/TH 边缘节点）、WAF 安全防护、ALB 多可用区流量分发与 TLS 终结；灰度发布时在 ALB/网关做权重分流 | 静态资源命中率 > 95%；ALB 跨 ≥2 AZ |
| **L2 前端展现层** | React SPA 承载全部业务界面；页面职责严格区分——**分镜编辑页=准备**（提取/确认/修正，推进 `shot.status→ready`），**分镜工作室=生成**（视频准备度、关键帧、图片、视频生成），**任务中心=通用任务状态面板**（只展示状态/进度/取消/回跳，不承载业务重信息） | 后端地址经 `env.js` 运行时注入，镜像与环境解耦 |
| **L3 API 层** | 收参、鉴权（目标态）、限流（目标态）、响应组织。统一响应包 `ApiResponse{code,message,data,meta}`；全局异常 handler 把 HTTPException/ValidationError/Exception 收敛为统一格式 | **不做业务逻辑**；Pydantic schema 按域拆分（`schemas/studio`、`schemas/skills`） |
| **L4 业务服务层** | 业务逻辑、状态流转、数据编排。studio（分镜/资产/生成上下文构建）、llm（供应商注册/默认模型解析，缺失返回 503）、film（视频与任务查询）、script_processing（LangGraph Agent：剧本拆分、元素抽取、一致性检查、实体合并等） | **service 层不感知 HTTP**；生成能力经 4 层模型（Base Draft → Context → Derived Preview → Submission）收敛 |
| **L5 任务执行层** | Celery Worker 消费 Redis 队列；`run_task_celery(task_id)` 统一入口按 `task_kind` 解析执行器；执行模板管理状态机（pending→running/streaming→succeeded/failed/cancelled）与取消检查点；async runner 经 `AbstractAsyncDelegatingExecutor` 桥接（默认 1800s 超时） | 任务真相在 DB（`generation_tasks`），Redis 仅作 broker；prefork 子进程需 `reset_db_runtime()` 重建 async engine |
| **L6 能力集成层** | `core/contracts` 定义跨层 DTO（`ImageGenerationInput/Result`、`VideoGenerationInput`、`ProviderConfig`，`extra=forbid`）；OpenAI 兼容与火山引擎适配器实现图片/视频生成；能力表按 (provider, model) 注册支持的 size/ratio 并做提交前校验 | **integrations 只依赖 contracts，不依赖 tasks**；HTTP 交互经 `http_logging` 记录 |
| **L7 数据与存储层** | MySQL 存业务与任务真相；Redis 作 broker/缓存/限流计数；对象存储（OSS）存帧图、视频、素材，DB 中以 `files` + `file_usages` 多态引用 | 大文件一律走对象存储，DB 只存元数据与引用 |
| **L8 可观测与治理层** | 结构化日志（SLS）、指标（Prometheus/ARMS）、链路（OTel）、告警（CloudMonitor）；贯穿 L3~L6 | 详见第 4 章 |

**状态语义三权分立（架构级约定）：**

- `shot.status`：仅表示**信息提取确认状态**（`pending`/`ready`），不表达运行时状态；
- 运行时生成状态：统一来自任务系统（`generation_tasks`）与 runtime summary；
- `video-readiness`：独立表示**是否具备视频生成条件**，与 `shot.status` 分离（ready ≠ 立即可生成）。

---

## 3. 代码目录结构设计

### 3.1 现状总览（仓库实际结构）

```text
Jellyfish/
├── backend/                          # Python FastAPI 后端
│   ├── app/
│   │   ├── main.py                   # 应用入口：中间件/异常处理/路由挂载
│   │   ├── config.py                 # pydantic-settings 配置（.env）
│   │   ├── bootstrap.py              # 启动注册：provider + task adapter
│   │   ├── api/                      # ── API 层 ──
│   │   │   ├── dependencies.py       #   get_db / get_llm 依赖注入
│   │   │   ├── utils.py
│   │   │   └── v1/
│   │   │       ├── routes/
│   │   │       │   ├── health.py     #   健康检查
│   │   │       │   ├── llm.py        #   供应商/模型/默认模型 CRUD
│   │   │       │   ├── script_processing.py  # 20+ Agent 端点
│   │   │       │   ├── studio/       #   projects/chapters/shots/entities/
│   │   │       │   │                 #   prompts/files/timeline/image_tasks/
│   │   │       │   │                 #   shot_character_links（12 子路由）
│   │   │       │   └── film/         #   generated_video/tasks_images/task_status
│   │   ├── schemas/                  # ── DTO 层（Pydantic）──
│   │   │   ├── common.py             #   ApiResponse 统一响应包
│   │   │   ├── task.py / llm.py
│   │   │   ├── studio/               #   分镜域 DTO
│   │   │   └── skills/               #   剧本处理技能 DTO
│   │   ├── services/                 # ── 业务服务层 ──
│   │   │   ├── studio/               #   shots/entities/image_task_runner/
│   │   │   │   └── generation/       #   frame·video·asset_image
│   │   │   │                         #   (build_context/build_submission/derive_preview)
│   │   │   ├── llm/                  #   provider_registry/resolver/runtime/manage
│   │   │   ├── film/                 #   视频与任务查询
│   │   │   ├── worker/               #   task_executor/task_registry/task_logging
│   │   │   ├── common/
│   │   │   └── script_processing_tasks.py
│   │   ├── core/                     # ── 基础设施层 ──
│   │   │   ├── db.py                 #   async engine + session proxy
│   │   │   ├── db_sync.py            #   Celery 用同步 engine（aiomysql→pymysql）
│   │   │   ├── storage.py            #   S3/boto3（anyio 线程池防阻塞）
│   │   │   ├── celery_app.py         #   Celery 实例（ignore_result）
│   │   │   ├── task_manager/         #   manager/stores/strategies/types
│   │   │   ├── tasks/                #   registry/bootstrap/image·video_generation_tasks
│   │   │   ├── contracts/            #   跨层契约 DTO（provider/image/video）
│   │   │   └── integrations/         #   openai/ + volcengine/ 适配器
│   │   │       ├── openai/           #     images/video/capabilities/payload
│   │   │       ├── volcengine/
│   │   │       └── http_logging.py   #     供应商 HTTP 交互日志
│   │   ├── chains/agents/            # ── LangGraph Agent 编排 ──
│   │   │   ├── base.py               #   Agent 基类
│   │   │   ├── script_divider.py     #   章节→分镜拆分
│   │   │   ├── element_extractor.py  #   角色/场景/道具/服装/对白抽取
│   │   │   ├── consistency_checker.py#   一致性检查
│   │   │   ├── entity_merger.py      #   实体合并
│   │   │   └── character_portrait.py 等定向分析 Agent
│   │   ├── models/                   # ── ORM 层（SQLAlchemy 2.0）──
│   │   └── tasks/execute_task.py     # Celery 统一入口 run_task_celery
│   ├── sql/                          # 手写增量迁移 001~008-*.sql
│   ├── scripts/                      # 运维脚本（backfill_file_usages.py 等）
│   ├── tests/                        # pytest（~40 文件）
│   │   ├── core/integrations/        #   适配器/能力表/http_logging 单测
│   │   └── support/llm_api_app.py    #   模拟 LLM API
│   ├── init_db.py / init_storage.py  # 初始化脚本
│   └── pyproject.toml                # uv 管理，pytest/pylint 配置
├── front/                            # React 前端
│   ├── src/
│   │   ├── App.tsx                   # 路由定义
│   │   ├── layouts/MainLayout.tsx
│   │   ├── pages/
│   │   │   ├── Settings/ NotFound/
│   │   │   └── aiStudio/             # 主业务域
│   │   │       ├── project/          #   ProjectLobby / ProjectWorkbench
│   │   │       ├── chapter/          #   ChapterStudio(工作室) / ChapterPrep
│   │   │       ├── shots/            #   ChapterShotsPage / ChapterShotEditPage(编辑页)
│   │   │       ├── assets/           #   AssetManager（演员/场景/道具/服装）
│   │   │       ├── editor/           #   VideoEditor 剪辑
│   │   │       ├── files/ models/ prompts/ agents/
│   │   │       └── components/TaskCenter.tsx  # 任务中心 + TaskRuntimeProvider
│   │   ├── services/
│   │   │   ├── generated/            # OpenAPI generated client（core/models/services）
│   │   │   ├── http.ts               # axios 封装（baseURL 可运行时注入）
│   │   │   └── openapi.ts            # OpenAPI.BASE 初始化
│   │   ├── store/useAppStore.ts      # Zustand 全局状态
│   │   ├── locales/                  # i18next（多语言）
│   │   └── mocks/                    # MSW mock
│   └── package.json                  # openapi:gen 脚本同步后端契约
├── deploy/
│   ├── docker/
│   │   ├── backend.Dockerfile        # python:3.12-slim + uv + uvicorn
│   │   ├── front.Dockerfile          # node20+pnpm9 构建 → nginx:1.27-alpine
│   │   ├── nginx.conf                # SPA try_files / env.js no-store
│   │   └── docker-entrypoint.d/10-generate-env-js.sh
│   └── compose/
│       ├── docker-compose.yml        # 8 服务编排
│       └── .env.example
├── site/content/docs/                # Hugo 文档站
│   ├── architecture/                 # 当前架构（project-structure/shot-status-flow/
│   │                                 #   task-execution/generation-workspace 等）
│   ├── plans/                        # 推进中计划（development-plan/
│   │                                 #   task-async-cancellation-plan 等）
│   ├── guide/                        # 开发指南 how-to
│   └── blog/                         # release note（固化后不得修改）
├── docs/                             # README 多语言 + 截图
├── .github/workflows/                # CI/CD（见 3.4）
├── AGENTS.md                         # 工程约定（分层职责/状态语义/文档规则）
├── pytest.ini / conftest.py / pyrightconfig.json
└── likha/impl_tech.md                # 本文档
```

### 3.2 后端逻辑分层规则（强约束）

```mermaid
flowchart LR
    A[api 层<br/>收参/鉴权/响应组织] --> B[services 层<br/>业务逻辑/状态流转/数据编排]
    B --> C[core 层<br/>db/storage/celery/task_manager]
    B --> D[core/contracts<br/>跨层 DTO 契约]
    D --> E[core/integrations<br/>供应商适配器]
    B --> F[chains/agents<br/>LangGraph 编排]
    F --> G[services/llm<br/>模型解析与运行时]
    C --> H[(MySQL/Redis/S3)]
    E --> I[外部 AI 供应商]
```

- `api` 不写业务逻辑，`services` 不感知 HTTP；
- 生成能力通用契约（输入/输出 DTO、供应商配置）统一放 `core/contracts`；`tasks` 只做任务封装/分派/执行编排；`integrations` 只依赖 `contracts`；
- API 变更后必须 `pnpm run openapi:update` 同步前端 generated client，禁止新增手写 service 封装。

### 3.3 测试体系

| 层级 | 位置 | 现状 | 目标补充 |
| --- | --- | --- | --- |
| 单元测试 | `backend/tests/`（~40 文件） | API 响应包格式、task_manager/task_registry、LLM resolver、shot 状态/候选/链接、file_usages、integrations 适配器与能力表 | service 层覆盖率提升至 ≥70%；契约测试（contracts schema 兼容性） |
| 集成测试 | `backend/tests/`（marker: `integration`） | `tests/support/llm_api_app.py` 模拟 LLM API | 引入 testcontainers（MySQL/Redis）跑真实依赖 |
| 前端测试 | `front/`（MSW mock 已就绪） | 以 mock 支撑开发 | 补充 Vitest 组件测试 + Playwright E2E（覆盖「编辑页→ready→工作室→生成」主链路） |
| **压测（现无，规划）** | `loadtest/`（新建） | — | k6 场景：① API 基线（400 RPS 混合读写）② 任务风暴（并发 500 视频任务入队）③ 轮询压力（2000 客户端 task_status 轮询）；接入 CI 夜间跑，基线：P99 < 500ms（读）/ < 1.5s（写），错误率 < 0.1% |
| 契约测试 | CI | — | 后端 OpenAPI diff 检查，Breaking change 阻断合并 |

### 3.4 CI/CD 流水线（现状）

| Workflow | 触发 | 内容 |
| --- | --- | --- |
| `backend-pylint.yml` | PR（backend/**） | `uv run pylint app` |
| `commit-messages.yml` | PR | 校验 `[类型] 摘要` 提交格式 |
| `ghcr-images.yml` | main / v* | QEMU+Buildx 构建并推送 backend/front 镜像至 ghcr.io |
| `tag-release.yml` | v* tag | 自动创建 GitHub Release（tag 注释作摘要） |
| `deploy-site.yml` | main（site/**） | Hugo 构建部署文档站 |

**目标态补充**：单测+覆盖率门禁 → OpenAPI diff → 镜像构建（语义化 tag）→ 部署 staging → 冒烟/E2E → 灰度发布（见 4.3）→ 生产全量。

### 3.5 文档体系（site/content/docs）

- `architecture/`：**当前真实生效**的架构（project-structure、shot-status-flow、shot-page-boundary、task-execution、generation-workspace、llm-default-model-resolution、shot-frame-prompt-generation）；
- `plans/`：**推进中**的计划（development-plan、task-async-cancellation-plan、generation-workspace-refactor）；
- `guide/`：开发流程 how-to；`blog/`：release note（固化后不得修改）；
- 强约束：架构落地但文档未更新 = 未完成；计划变更但文档未更新 = 未完成。

---

## 4. 日志、监控、灰度发布、限流与回滚能力

> 说明：本章按「现状 → 目标态设计 → 实现时序图」组织。现状部分基于代码事实，目标态为阿里云生产化方案。

### 4.1 日志能力

**现状：**

- 各模块 `logging.getLogger(__name__)`，**无集中式 logging 配置**（无 dictConfig）；
- `services/worker/task_logging.py` 输出结构化任务事件日志（started/succeeded/failed/cancelled + elapsed_ms）；
- `core/integrations/http_logging.py` 记录供应商 HTTP 交互；
- 容器 stdout 由 Docker 收集，无持久化与检索。

**目标态设计：**

| 项 | 方案 |
| --- | --- |
| 格式 | 全量 JSON 结构化：`{ts, level, service, trace_id, task_id, user_id, event, message, extra}`；uvicorn/gunicorn 日志接管进同一 formatter |
| 采集 | 容器 stdout → **阿里云 SLS Logtail**（DaemonSet/ECS agent），按 logstore 分流：`app-log` / `access-log` / `task-log` / `integration-log` |
| 关联 | API 入口生成/透传 `X-Trace-Id`，经 contextvar 注入所有日志；Celery 任务在 `run_task_celery` 入口把 `task_id` 绑定到日志上下文 |
| 保留 | 热存 30 天（PH 主区域），冷存 OSS 180 天；含个人信息字段脱敏 |
| 审计 | 供应商调用（含 token 消耗）、任务取消、模型配置变更单独 logstore，保留 1 年 |

**日志采集与检索时序图：**

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户/前端
    participant API as FastAPI (api层)
    participant SVC as Service 层
    participant CW as Celery Worker
    participant P as AI 供应商
    participant SLS as 阿里云 SLS
    participant OSS as OSS 冷归档

    U->>API: 请求（携带/生成 X-Trace-Id）
    API->>API: access-log: {trace_id, route, status, latency_ms}
    API->>SVC: 业务调用（trace_id 经 contextvar 传递）
    SVC->>SLS: app-log: {trace_id, event, entity_ids}
    SVC->>CW: 入队任务（trace_id 写入任务 headers）
    CW->>SLS: task-log: {task_id, trace_id, event=started}
    CW->>P: 生成请求
    P-->>CW: 结果/错误
    CW->>SLS: integration-log: {provider, model, tokens, http_status}
    CW->>SLS: task-log: {event=succeeded/failed, elapsed_ms}
    SLS->>OSS: 30 天后自动投递冷存（180 天）
    Note over SLS: 告警规则：error 率>1%/5min、<br/>任务失败率>5%、供应商超时激增
```

### 4.2 监控能力

**现状：** 仅 `GET /health` 健康检查端点；无 metrics、无 tracing、无告警。

**目标态设计（三层监控）：**

| 层 | 指标 | 工具 |
| --- | --- | --- |
| 基础设施 | CPU/内存/磁盘/网络、MySQL 连接数与慢查询、Redis 内存与命中率、OSS 用量 | 云监控 CloudMonitor + RDS/Tair 自带监控 |
| 应用 | HTTP QPS/延迟分位(P50/P95/P99)/错误率、`/health` 存活、Celery 队列深度与消费延迟、worker 存活数 | `prometheus-fastapi-instrumentator` + `celery-exporter` → ARMS Prometheus |
| 业务 | 任务成功率（按 task_kind/provider 维度）、任务平均耗时、视频生成失败 Top 原因、活跃项目数、shot ready 转化率、供应商配额余量 | 自定义 metrics（在 task_logging 埋点处同步打点）+ Grafana 大盘 |

**核心告警规则（对接 99.95% SLA）：**

| 告警 | 阈值 | 级别 | 响应 |
| --- | --- | --- | --- |
| API 5xx 比例 | > 1% 持续 3min | P1 | 电话 + 钉钉，5min 内响应 |
| API P99 延迟 | > 2s 持续 5min | P2 | 钉钉群 |
| `/health` 探测失败 | 连续 3 次（30s 间隔） | P1 | ALB 自动摘除节点 + 电话 |
| Celery 队列积压 | > 500 任务或最老任务 > 10min | P2 | 扩容 worker |
| 视频任务失败率 | > 10% 持续 10min | P1 | 检查供应商侧，必要时切换 provider |
| MySQL 连接使用率 | > 80% | P2 | 连接池调优/升配 |
| Redis 内存 | > 75% maxmemory | P2 | 扩容 |

**监控数据流时序图：**

```mermaid
sequenceDiagram
    autonumber
    participant API as FastAPI
    participant W as Celery Worker
    participant EXP as /metrics 端点
    participant PROM as ARMS Prometheus
    participant GRAF as Grafana 大盘
    participant AM as 告警管理器
    participant OC as 值班 (钉钉/电话)

    loop 每 15s 抓取
        PROM->>EXP: GET /metrics
        EXP-->>PROM: http_request_duration_seconds,<br/>celery_queue_depth, task_success_total...
    end
    API->>EXP: 请求级指标自动累加（中间件）
    W->>EXP: 任务级指标（started/succeeded/failed/elapsed）
    PROM->>GRAF: PromQL 查询渲染（SLA 大盘/业务大盘）
    PROM->>AM: 规则评估（5xx>1% / P99>2s / 积压>500）
    AM->>OC: 分级通知（P1 电话, P2 钉钉）
    OC->>GRAF: 定位（按 trace_id 下钻到 SLS 日志）
    Note over OC: P1 事件 → 触发回滚预案（见 4.5）
```

### 4.3 灰度发布能力

**现状：** 无灰度机制，compose 直接重建容器（有停机窗口）。

**目标态设计（金丝雀 + 权重灰度）：**

- **前端**：静态资源版本化（`index.html` no-cache + 带 hash 的 assets 长缓存），CDN 刷新即完成发布；灰度按 `env.js` 下发不同 BACKEND_URL 或按 Cookie 分流到新灰度页面版本。
- **后端**：ALB 服务器组权重灰度（5% → 25% → 50% → 100%），或使用 MSE 云原生网关按 Header（`X-Canary: true`，内部测试白名单）路由。
- **Celery Worker**：新旧版本双队列共存——新版本 worker 消费 `queue:v2`，发布时先把新任务按灰度比例路由到 v2 队列（API 侧按 project_id hash 决定），旧 worker 排空 `queue:v1` 后下线。任务表带 `schema_version`，保证新旧 worker 都能处理存量任务。
- **灰度准入/准出**：观察窗口 ≥30min；准入指标 = 5xx 率、P99、任务成功率与基线偏差 < 20%；任一越界自动回滚。
- **数据库变更约束**：只允许**扩展式迁移**（加列/加表/加索引），禁止在同一发布中删列改列；破坏性变更延后一个版本（先双写/兼容读，再清理），保证回滚旧代码仍可运行。

**灰度发布时序图：**

```mermaid
sequenceDiagram
    autonumber
    participant CI as CI/CD 流水线
    participant REG as 镜像仓库 (ACR)
    participant ALB as ALB / MSE 网关
    participant OLD as backend v1.4 (基线)
    participant NEW as backend v1.5 (金丝雀)
    participant PROM as Prometheus
    participant W1 as Celery v1 (queue:v1)
    participant W2 as Celery v2 (queue:v2)

    CI->>REG: 构建推送 v1.5 镜像（语义化 tag）
    CI->>NEW: 部署金丝雀实例（跨 AZ，先不接流量）
    NEW->>NEW: 自检 /health + 冒烟用例
    CI->>ALB: 权重调整 v1.5 = 5%
    ALB->>OLD: 95% 流量
    ALB->>NEW: 5% 流量（含内部白名单 X-Canary）
    CI->>W2: 启动 v2 worker，API 按 project_id hash<br/>将 5% 项目的新任务投递 queue:v2
    loop 观察窗口 30min
        PROM->>PROM: 对比 v1.4/v1.5 的 5xx、P99、任务成功率
        alt 指标劣化 > 20%
            PROM-->>CI: 自动触发回滚
            CI->>ALB: v1.5 权重 → 0%
            CI->>W2: 停止接新任务，排空后下线
        else 指标达标
            CI->>ALB: 逐级放量 25% → 50% → 100%
            CI->>W1: v1 worker 排空 queue:v1 后下线
            CI->>CI: 发布完成，记录 Release Note
        end
    end
```

### 4.4 限流能力

**现状：** **无任何限流**（未引入 slowapi 等）；CORS 全开 methods/headers；无认证，端点裸露。这是生产化最高优先级补齐项。

**目标态设计（四层限流漏斗）：**

| 层 | 手段 | 规则示例 |
| --- | --- | --- |
| L1 边缘 | 阿里云 WAF CC 防护 + CDN 速率限制 | 单 IP > 100 req/s 触发人机校验 |
| L2 网关 | ALB/MSE 网关路由级限流 | `/api/v1/script_processing/*`（LLM 同步调用，成本高）：全局 50 QPS |
| L3 应用 | FastAPI 中间件 + Redis 滑动窗口（slowapi 或自研） | 匿名：60 req/min/IP；认证用户：600 req/min；`task_status` 轮询：30 req/min/用户（前端配合指数退避） |
| L4 资源 | 任务入队配额 + Celery worker 并发上限 | 单用户进行中视频任务 ≤ 5；单项目 ≤ 20；全局队列软上限 5000，超限返回 429 + Retry-After；供应商侧按 API 配额令牌桶限速 |

**统一错误语义**：超限返回 `429 {code: 42900, message: "rate limit exceeded", meta: {retry_after}}`，前端 http.ts 拦截 429 自动退避重试（最多 2 次）。

**限流判定时序图：**

```mermaid
sequenceDiagram
    autonumber
    participant C as 客户端
    participant WAF as WAF/CDN (L1)
    participant GW as ALB/MSE (L2)
    participant MW as FastAPI 限流中间件 (L3)
    participant R as Redis (滑动窗口计数)
    participant H as 路由 Handler
    participant TM as task_manager (L4)
    participant Q as Celery 队列

    C->>WAF: 请求
    alt IP 频率超限 (L1)
        WAF-->>C: 405/人机校验
    end
    WAF->>GW: 转发
    alt 路由全局限流 (L2)
        GW-->>C: 429
    end
    GW->>MW: 转发
    MW->>R: INCR rate:{user_id|ip}:{route}:{minute} (EXPIRE 60)
    R-->>MW: 当前计数
    alt 计数超限 (L3)
        MW-->>C: 429 {retry_after}
    else 通过
        MW->>H: 进入业务
        H->>TM: 创建生成任务
        TM->>TM: 检查用户/项目进行中任务配额 (L4)
        alt 配额超限
            TM-->>C: 429 任务配额超限
        else 通过
            TM->>Q: 入队（队列深度 > 5000 时拒绝）
            Q-->>C: 202 {task_id}
        end
    end
```

### 4.5 回滚能力

**回滚分层与 RTO 目标：**

| 层 | 回滚手段 | RTO |
| --- | --- | --- |
| 前端 | CDN 切回上一版本静态资源目录（版本化目录 + `index.html` 指针切换），刷新 CDN | < 3 min |
| 后端 API | ALB 权重切回 v1.4 服务器组（灰度期间零回滚成本）；全量发布后回滚 = 重新部署旧镜像 tag | < 5 min |
| Celery | 停 v2 worker → 重启 v1 镜像 worker；queue:v2 中未消费任务由 v1 worker（兼容 schema_version）接管或重新入队 | < 10 min |
| 数据库 | **不回滚 schema**（扩展式迁移保证旧代码兼容）；数据级错误用备份恢复：RDS 自动备份（每日全量 + binlog），支持任意时间点恢复 PITR | PITR < 30 min |
| 配置 | Nacos/环境变量版本化，配置变更同样走灰度与一键回退 | < 2 min |

**回滚决策与执行时序图：**

```mermaid
sequenceDiagram
    autonumber
    participant PROM as 监控告警
    participant OC as 值班工程师
    participant CD as 发布系统
    participant ALB as ALB
    participant NEW as v1.5 (故障)
    participant OLD as v1.4 (稳定)
    participant W as Celery Worker
    participant DB as RDS MySQL

    PROM->>OC: P1 告警：v1.5 5xx=8%，任务失败率 25%
    OC->>PROM: 确认劣化集中于 v1.5 实例（按版本维度下钻）
    OC->>CD: 触发一键回滚
    CD->>ALB: v1.5 权重 → 0%，流量全回 v1.4
    ALB->>OLD: 100% 流量
    CD->>W: 停止 v2 worker 消费；<br/>queue:v2 剩余任务重投 queue:v1
    W->>DB: running 状态且 executor 已死的任务<br/>标记 failed(reason=rollback)，允许用户重试
    CD->>CD: 校验 v1.4 /health + 冒烟
    CD-->>OC: 回滚完成（RTO < 5min）
    OC->>OC: 复盘：DB schema 保持向前兼容，无需数据回滚
    Note over OC,DB: 若为数据级事故 → RDS PITR<br/>恢复到故障前时间点（RTO < 30min）
```

---

## 5. 数据存储结构设计

### 5.1 存储选型

| 存储 | 用途 | 现状 | 阿里云生产态 |
| --- | --- | --- | --- |
| 关系型 DB | 业务实体、任务真相、模型配置 | MySQL 9.0（compose）/ SQLite（开发默认），SQLAlchemy 2.0，手写 SQL 迁移 `backend/sql/001~008` | RDS MySQL 8.0 高可用版（主备跨 AZ）+ 只读实例 |
| 缓存/队列 | Celery broker、限流计数、会话 | Redis 7 单实例 | 云数据库 Tair（Redis 兼容）主从版，跨 AZ |
| 对象存储 | 帧图、视频、素材文件 | RustFS（S3 兼容，compose）；代码经 `core/storage.py` boto3 访问 | OSS（Manila 主 bucket + 跨区域复制到 Bangkok 可选），CDN 回源 |
| 文件元数据 | `files` + `file_usages` 多态引用 | DB 表 | 同左 |

### 5.2 ER 结构图（核心域）

```mermaid
erDiagram
    PROJECTS ||--o{ CHAPTERS : "包含"
    CHAPTERS ||--o{ SHOTS : "拆分出"
    SHOTS ||--|| SHOT_DETAILS : "1:1 详情"
    SHOTS ||--o{ SHOT_FRAME_IMAGES : "关键帧(first/last/key)"
    SHOTS ||--o{ SHOT_DIALOG_LINES : "对白"
    SHOTS ||--o{ SHOT_EXTRACTED_CANDIDATES : "资产候选"
    SHOTS ||--o{ SHOT_EXTRACTED_DIALOGUE_CANDIDATES : "对白候选"
    PROJECTS ||--o{ CHARACTERS : "角色"
    PROJECTS ||--o{ SCENES : "场景"
    PROJECTS ||--o{ PROPS : "道具"
    PROJECTS ||--o{ COSTUMES : "服装"
    PROJECTS ||--o{ ACTORS : "演员"
    CHARACTERS ||--o{ CHARACTER_IMAGES : "形象图"
    SCENES ||--o{ SCENE_IMAGES : "场景图"
    PROPS ||--o{ PROP_IMAGES : "道具图"
    COSTUMES ||--o{ COSTUME_IMAGES : "服装图"
    ACTORS ||--o{ ACTOR_IMAGES : "演员图"
    SHOTS }o--o{ CHARACTERS : "shot_character_links"
    PROJECTS }o--o{ SCENES : "project_scene_links(可挂chapter/shot)"
    PROJECTS }o--o{ PROPS : "project_prop_links"
    PROJECTS }o--o{ COSTUMES : "project_costume_links"
    PROJECTS }o--o{ ACTORS : "project_actor_links"
    CHARACTERS }o--o{ PROPS : "character_prop_links"
    PROJECTS ||--o{ TIMELINE_CLIPS : "剪辑时间线"
    PROJECTS ||--o{ FILES : "文件"
    FILES ||--o{ FILE_USAGES : "多态引用(project/chapter/shot/asset)"
    PROMPT_TEMPLATES ||--o{ SHOT_DETAILS : "提示词模板"
    PROVIDERS ||--o{ MODELS : "1:N (CASCADE)"
    MODEL_SETTINGS }o--|| MODELS : "默认 text/image/video 模型(SET NULL)"
    GENERATION_TASKS ||--o{ GENERATION_TASK_LINKS : "任务↔业务实体"
    SHOTS ||--o{ GENERATION_TASKS : "经 task_links 关联"

    PROJECTS {
        bigint id PK
        string title
        string style "全局风格"
        string visual_style
        int seed
        bool unify_style
        json progress
        string default_video_ratio
        json stats
    }
    CHAPTERS {
        bigint id PK
        bigint project_id FK
        int index
        string title
        text raw_text "原始剧本"
        text condensed_text "浓缩文本"
        int storyboard_count
        string status
    }
    SHOTS {
        bigint id PK
        bigint chapter_id FK
        int index
        string status "pending|ready(仅提取确认语义)"
        bool skip_extraction
        text script_excerpt
        bigint generated_video_file_id FK
    }
    SHOT_DETAILS {
        bigint id PK
        bigint shot_id FK "1:1"
        string camera_shot "景别"
        string camera_angle
        string camera_movement
        float duration
        json mood_tags
        string vfx_type
        json action_beats "动作拍点"
        text first_frame_prompt
        text last_frame_prompt
        text key_frame_prompt
        bigint prompt_template_id FK
    }
    GENERATION_TASKS {
        bigint id PK
        string task_kind "image|video|text..."
        string status "pending|running|streaming|succeeded|failed|cancelled"
        string delivery_mode "streaming|async_polling"
        string executor_type
        string executor_task_id "Celery task id"
        bool cancel_requested
        int progress
        json payload "提交参数快照"
        json result
        string failure_reason
        datetime created_at
        datetime updated_at
    }
    PROVIDERS {
        bigint id PK
        string provider_key "openai|volcengine"
        string base_url
        string image_base_url "覆盖"
        string video_base_url "覆盖"
        string api_key
        string api_secret
        string status
    }
    MODELS {
        bigint id PK
        bigint provider_id FK
        string category "text|image|video"
        string model_name
        json params
    }
    MODEL_SETTINGS {
        bigint id PK "单例表"
        bigint default_text_model_id FK
        bigint default_image_model_id FK
        bigint default_video_model_id FK
        int api_timeout
        string log_level
    }
    FILES {
        bigint id PK
        string object_key "S3 key"
        string content_type
        bigint size
        string source
    }
    FILE_USAGES {
        bigint id PK
        bigint file_id FK
        string usage_type "shot_frame|asset_image|video..."
        string ref_entity "多态引用"
        bigint ref_id
    }
```

### 5.3 关键设计说明

1. **任务真相层与执行层分离**：`generation_tasks`（真相）+ `generation_task_links`（任务↔业务实体多态关联）构成真相层；Celery/Redis 只是执行层。`task_ignore_result=True`，不使用 result backend——worker 崩溃后可通过 DB 中 `running` 且 executor 失活的任务识别并允许重试。
2. **候选确认模型**：AI 抽取结果先进 `shot_extracted_candidates` / `shot_extracted_dialogue_candidates`（候选态），用户在分镜编辑页 accept/ignore/link 后才落正式资产表——保证 AI 输出必须经人工确认，避免污染一致性资产。
3. **资产链接三级挂载**：`project_*_links` 支持 project/chapter/shot 三级作用域，实现「项目级默认 + 章节/镜头级覆盖」的一致性管理。
4. **多态文件引用**：`files` + `file_usages` 解耦存储与业务，任何实体（帧图、资产图、视频、素材）通过 usage 关联，便于生命周期管理与孤儿文件清理（`scripts/backfill_file_usages.py` 为回填工具）。
5. **迁移策略**：手写编号 SQL（001~008）按序执行，`init_db.py` 负责 `create_all` 建表。生产态需引入 Alembic 版本化迁移（见 5.6 与第 10 章 D1）。

### 5.4 索引设计要点（按真实查询模式）

> 现状：建表依赖 `create_all` + 手写 SQL，**索引未按查询模式系统设计**（MySQL 仅对外键自动建索引）。以下为目标态索引清单，落地时以慢查询日志验证。

| 表 | 索引 | 服务的查询 | 优先级 |
| --- | --- | --- | --- |
| `shots` | `(chapter_id, index)` 唯一 | 分镜编辑页/工作室按章节列镜头（最高频查询） | P0 |
| `shot_details` | `shot_id` 唯一 | 1:1 详情联查 | P0 |
| `generation_tasks` | `(status, created_at)` | reaper 扫描悬挂 `running` 任务、按状态过滤 | P0 |
| `generation_tasks` | `(task_kind, status, created_at)` | 业务统计（成功率/耗时按 kind 维度）、任务中心筛选 | P1 |
| `generation_tasks` | `(created_at, id)` | 任务列表 keyset 分页（防深分页 OFFSET 扫描） | P1 |
| `generation_task_links` | `(link_type, link_id)` | 实体反查任务（镜头维度任务列表，前端轮询入口） | P0 |
| `generation_task_links` | `task_id` | 任务→实体正查 | P0 |
| `shot_extracted_candidates` / `shot_extracted_dialogue_candidates` | `(shot_id, status)` | 编辑页待确认候选列表 | P0 |
| `file_usages` | `(file_id)`、`(usage_type, ref_id)` | 多态引用正/反查、孤儿文件 GC 扫描 | P1 |
| `files` | `object_key` 唯一 | 上传去重、S3 key 反查 | P1 |
| `shot_frame_images` | `(shot_id, frame_type)` | 关键帧齐备性检查（video-readiness 判定路径） | P0 |
| `shot_character_links` | `(shot_id)`、`(character_id)` | 镜头↔角色双向查询 | P1 |
| `models` | `(provider_id, category)` | 模型管理列表、默认模型解析 | P1 |
| `chapters` | `(project_id, index)` | 章节列表 | P1 |

**慢查询治理**：RDS 慢日志阈值 1 s → SLS 每日聚合 Top10 → 月度索引评审；任务中心轮询类查询必须命中 `(link_type, link_id)` 或走只读实例 + Redis 缓存（见 5.9）。

### 5.5 主键 / 时间戳 / 软删除与大字段约定

| 约定 | 规则 |
| --- | --- |
| 主键 | 统一 `bigint` 自增；不使用业务字段作主键 |
| 时间戳 | `created_at` / `updated_at` 统一 `DATETIME(3)` **UTC 存储**，展示层按用户时区（PHT/ICT）转换；任务耗时统计一律用 `elapsed_ms`（task_logging 已有） |
| 软删除 | 业务资产（角色/场景/道具/服装）**不做物理删除**，用候选 `ignored` 标记与链接解除表达"删除"；`files` 由 GC 任务按 `file_usages` 引用计数清理，禁止业务代码直接删 OSS 对象 |
| JSON 大字段 | `generation_tasks.payload/result`、`projects.stats`、`shot_details.action_beats` 为 JSON 列。**风险**：`VideoGenerationInput` 含 first/last/key 帧 `base64`，若原样快照进 payload，单行可达数 MB，导致表膨胀与主从复制延迟——**payload 中 base64 必须替换为 `file_id/object_key` 引用**，提交时再物化（改进项，见 10.3 D6） |
| 枚举 | `status`/`task_kind`/`delivery_mode` 用短字符串 + 应用层枚举校验（现状），禁止用 MySQL ENUM（迁移不友好） |
| 字符集 | 全库 `utf8mb4`（compose mysql-init-sql 已强制 `--default-character-set=utf8mb4`），支持泰文/菲律宾语内容与 emoji |

### 5.6 迁移与版本化策略

**现状与风险：**

```text
现状链路：init_db.py (Base.metadata.create_all)  →  mysql-init-sql 容器按文件名顺序执行 backend/sql/001~008-*.sql
```

| 风险 | 说明 |
| --- | --- |
| 无版本记录表 | SQL 是否已执行只靠"容器只跑一次"的编排假设；重放非幂等 SQL（如 ADD COLUMN）会失败 |
| 双轨漂移 | `create_all`（models 定义）与手写 SQL 两套 schema 来源可能不一致，新环境 vs 存量环境结构漂移 |
| 无 down 迁移 | 无法回退，只能靠备份恢复 |
| 无 CI 校验 | 迁移脚本不在 CI 中重放验证 |

**目标态（Alembic 收编）：**

1. 以当前 `models/` + 001~008 合并结果生成 Alembic **基线版本**（`alembic revision --autogenerate` 后人工校对）；
2. `alembic_version` 表成为唯一版本真相，`init_db.py` 退化为「alembic upgrade head」；
3. CI 增加迁移重放测试：空库 → head、基线库 → head 两条路径都必须通过；
4. 发布顺序强约束：**先迁移（扩展式）→ 灰度新代码 → 全量**；破坏性变更（删列/改类型）延后一个版本，保证 4.5 回滚时序中「DB 不回滚 schema」成立；
5. worker 与 API 混版本期间，双方必须兼容 `schema_version` 前后各一个版本。

### 5.7 数据容量估算与治理（写入放大 / 归档 / 分页）

**行数与存储增长估算**（按 1 章业务量假设：日均新增 20 个项目 × 80 章 × 10 镜头；每镜头平均 3 个生成任务、30% 重试）：

| 表 | 日增行数 | 单行估算 | 月增量 | 治理策略 |
| --- | --- | --- | --- | --- |
| `shots` + `shot_details` | 1.6 万 × 2 | 0.5~2 KB（提示词文本） | ~1.5 GB | 无需分表；深分页用 keyset |
| `shot_extracted_candidates`（含对白候选） | ~8 万 | 0.3~1 KB | ~1.5 GB | 已确认/忽略候选 90 天后归档表或压缩 |
| `generation_tasks` | ~7 万 | 2~10 KB（payload 快照，base64 引用化后 ≤ 2 KB） | 4~15 GB | **完成态任务 30 天后 payload/result 转存 OSS，DB 留摘要**；热表只保 90 天 |
| `generation_task_links` | ~10 万 | < 100 B | ~0.2 GB | 随任务归档 |
| `files` + `file_usages` | ~30 万 | < 200 B | ~1.5 GB | GC 清孤儿引用 |
| OSS 对象 | 50~200 GB | — | 1.5~6 TB | 生命周期：90 天转低频（IA）、180 天转归档；失败任务产物 7 天清理 |

**结论**：关系库年增量 < 50 GB，**RDS 高可用版 4c16g + 500 GB 存储（自动扩容）3 年内无需分库分表**；真正的容量压力在 OSS（媒体文件）与 `generation_tasks` JSON 膨胀，治理重点是 5.5 的 payload 引用化 + 本节归档策略。

**分页与写入放大控制：**

| 机制 | 规则 |
| --- | --- |
| 分页上限 | 所有列表接口 `page_size ≤ 100`（Pydantic 校验）；任务中心/文件管理默认 20 |
| keyset 分页 | `generation_tasks`、`files` 等大表列表用 `(created_at, id)` 游标，禁止 `OFFSET > 10000` |
| 轮询写放大 | 任务进度更新节流：worker 侧 progress 变更 ≥ 5% 或间隔 ≥ 3 s 才落库，避免轮询风暴放大为 DB 写风暴 |
| 批量写 | 候选确认、批量工具栏操作走事务批量 UPDATE，禁止循环单条提交 |
| 大结果集 | Agent 抽取结果分批插入（batch 500），防单事务过大拖慢主从复制 |

### 5.8 Redis 用途与 Key 规划

现状：Redis 仅作 Celery broker（`task_ignore_result=True`，无 result backend）。目标态 Key 清单：

| Key 模式 | 类型 | TTL | 用途 | 状态 |
| --- | --- | --- | --- | --- |
| `celery` / `queue:v1` / `queue:v2` / `queue:video` / `queue:image` / `queue:text` | LIST | — | broker 队列（灰度双队列见 4.3；优先级拆队见 10.4 T2） | 现状 + 目标 |
| `unacked*` | ZSET/HASH | — | Celery 未确认任务（acks_late） | 现状 |
| `ratelimit:{user_id|ip}:{route}:{window}` | 计数 | 60 s | L3 应用限流滑动窗口（4.4） | 目标 |
| `quota:user:{uid}:inflight` | 计数 | 任务生命周期 | L4 用户进行中任务配额 | 目标 |
| `task:status:{task_id}` | HASH | 2 s | `task_status` 轮询缓存（读路径卸载，见 5.9） | 目标 |
| `lock:reaper:{job}` | 锁 | 60 s + 心跳 | reaper/归档/GC 定时任务多实例互斥（租约锁） | 目标 |
| `idem:task:{client_request_id}` | 标记 | 24 h | 任务提交幂等键（防前端重复提交/重试重复扣配额） | 目标 |
| `provider:health:{provider_key}` | HASH | 60 s | 供应商熔断状态位（失败率窗口统计，配合双 provider 切换） | 目标 |

**关键约束**：Tair 必须 `maxmemory-policy=noeviction`（broker 与幂等键不可淘汰）；限流类 Key 允许 Redis 故障时**降级放行**（fail-open，可用性优先，见 8.4），队列类 Key 故障则任务提交快速失败并返回 503 + Retry-After。

### 5.9 读写分离与连接池预算

| 项 | 设计 |
| --- | --- |
| 读写分离 | 写路径（创建任务/确认候选/状态流转）→ RDS 主库；读路径（任务轮询/列表/工作台聚合查询）→ 只读实例；`DATABASE_URL` 之外增加 `DATABASE_READ_URL`，`get_db` 依赖按方法注入 |
| 轮询卸载 | `task_status` 先读 `task:status:{id}` 缓存（2 s TTL），未命中回源只读库——2000 客户端轮询场景下 DB QPS 从 ~700 降到 < 100 |
| 连接池参数 | SQLAlchemy：`pool_size=20, max_overflow=10, pool_recycle=3600, pool_pre_ping=True`（防 RDS 主备切换后持有死连接）；Celery 同步 engine（`db_sync.py`）独立池 `pool_size=10` |
| 连接数预算 | 不变量：`API 副本 × 30 + worker 副本 × 20 + 只读实例同额度 ≤ RDS max_connections × 0.8`。示例：API 4 副本 + worker 6 副本 = 240 连接，RDS 4c16g（默认 max_connections ≈ 8,000）余量充足；**扩容至 API+worker 合计 > 100 副本时必须引入 RDS 数据库代理**收敛连接 |
| 事务边界 | 单请求单事务；Agent 批量落库按章分事务，禁止跨章长事务（防主从延迟与锁等待） |

---

## 6. 核心业务逻辑时序图

### 6.1 剧本 → 分镜提取 → 确认 → ready（主流程前半段）

```mermaid
sequenceDiagram
    autonumber
    participant FE as 前端(分镜编辑页)
    participant API as FastAPI /api/v1
    participant SP as script_processing
    participant AG as LangGraph Agents
    participant LLM as LLM 供应商(OpenAI兼容)
    participant SVC as studio service
    participant DB as MySQL

    FE->>API: POST /script_processing 章节拆分请求
    API->>SP: 参数校验(Pydantic)
    SP->>AG: script_divider 编排
    AG->>LLM: 章节文本 → 分镜列表
    LLM-->>AG: 分镜结构化结果
    AG->>SVC: 落库
    SVC->>DB: INSERT shots(status=pending) + shot_details
    SVC-->>FE: 分镜列表

    FE->>API: POST 元素抽取(角色/场景/道具/服装/对白)
    API->>AG: element_extractor / *_info_analysis
    AG->>LLM: 抽取与定向分析(角色肖像/场景细节)
    LLM-->>AG: 抽取结果
    AG->>DB: INSERT shot_extracted_candidates<br/>+ shot_extracted_dialogue_candidates
    AG-->>FE: 候选列表(待确认)

    loop 用户逐项确认
        FE->>API: accept / ignore / link 已有资产
        API->>SVC: 候选确认
        SVC->>DB: 候选 → 正式资产表 / 标记 ignored<br/>写入 shot_character_links 等关联
    end

    FE->>API: 修正镜头基础信息(景别/时长/提示词)
    API->>SVC: 更新 shot_details
    SVC->>DB: UPDATE shot_details
    FE->>API: GET shot readiness 检查
    API->>SVC: 统一就绪判定(提取确认全部完成?)
    SVC->>DB: UPDATE shots.status = ready
    SVC-->>FE: status=ready → 引导进入分镜工作室
```

### 6.2 视频生成任务全链路（主流程后半段，异步任务系统）

```mermaid
sequenceDiagram
    autonumber
    participant FE as 前端(分镜工作室)
    participant API as FastAPI /film
    participant GEN as generation service(video)
    participant TM as task_manager
    participant DB as MySQL
    participant R as Redis(broker)
    participant CW as Celery Worker
    participant EX as WorkerTaskExecutor
    participant AD as 供应商适配器(volcengine/openai)
    participant P as AI 视频供应商
    participant OSS as 对象存储

    FE->>API: video-readiness 检查
    API->>GEN: 就绪判定(帧图/参数/提示词齐备?)
    GEN-->>FE: ready / 缺失项清单(引导回编辑页)

    FE->>API: POST 提交视频生成(prompt+first/last/key帧+ratio+seconds)
    API->>GEN: build_context → build_submission
    GEN->>GEN: contracts 校验(VideoGenerationInput, extra=forbid)<br/>能力表校验(provider,model 支持的 ratio/size)
    GEN->>TM: 创建任务
    TM->>DB: INSERT generation_tasks(status=pending, payload快照)
    TM->>R: enqueue run_task_celery(task_id)
    TM->>DB: UPDATE executor_type/executor_task_id
    API-->>FE: 202 {task_id}

    CW->>R: 消费任务
    CW->>DB: SELECT generation_tasks by task_id
    CW->>EX: 按 task_kind 从 task_registry 解析执行器
    EX->>DB: status=running, progress=5（检查 cancel_requested）
    EX->>AD: execute()（async runner 桥接, 1800s 超时）
    AD->>P: 提交生成请求(异步轮询模式)
    P-->>AD: provider_task_id
    loop async_polling
        AD->>P: 查询进度
        P-->>AD: processing / succeeded(video_url)
        AD->>DB: 更新 progress（检查 cancel）
    end
    P-->>AD: 视频产物 URL
    AD->>OSS: 下载并转存产物 → 生成 files 记录
    AD-->>EX: VideoGenerationResult
    EX->>DB: apply_result: shots.generated_video_file_id<br/>+ generation_task_links 关联
    EX->>DB: status=succeeded, progress=100（task_logging 记录 elapsed_ms）

    loop 前端轮询
        FE->>API: GET /film/task_status
        API->>DB: 查询任务状态
        API-->>FE: {status, progress, result} → 任务中心渲染
    end
```

### 6.3 任务取消（双通道）

```mermaid
sequenceDiagram
    autonumber
    participant FE as 前端(任务中心)
    participant API as FastAPI
    participant TM as task_manager
    participant DB as MySQL
    participant CW as Celery Worker
    participant EX as Executor
    participant P as AI 供应商

    FE->>API: POST /film/tasks/{id}/cancel
    API->>TM: request_cancel(task_id)
    TM->>DB: UPDATE cancel_requested=true
    TM->>CW: AsyncResult.revoke(terminate=SIGTERM)（通道2）
    alt Worker 在检查点内（通道1 协作式）
        EX->>DB: 阶段间读取 cancel_requested
        EX->>DB: status=cancelled（记录 cancel 字段）
        EX-->>FE: 任务中心显示已取消
    else SIGTERM 直接终止（通道2 强制）
        CW->>CW: 进程终止，任务留在 running
        Note over TM,DB: 兜底：API 侧 mark_cancelled<br/>将 DB 状态收敛为 cancelled
    end
    opt 供应商侧任务仍在跑
        P-->>P: 供应商任务自然结束（产物丢弃/不转存）
    end
```

---

## 7. 完整部署方案（阿里云 + Docker Compose）

### 7.1 区域与总体拓扑

**区域策略（服务菲律宾 + 泰国客户）：**

| 项 | 方案 | 理由 |
| --- | --- | --- |
| 主区域 | **菲律宾（Manila）`ap-southeast-6`** | 主要客户在菲律宾，本地落地延迟最低 |
| 泰国接入 | **CDN/DCDN 泰国节点加速 + 全球加速 GA 回源 Manila**；如泰国用户占比 > 40%，在 **曼谷 `ap-southeast-7`** 部署独立全套环境 | PH→TH 公网 RTT ≈ 40~70ms，CDN 加速后静态资源 < 20ms，API 经 GA 专线回源可稳定在 < 100ms |
| 可用区 | Manila 区域内 **≥ 2 个 AZ**（ALB/RDS/Tair/ECS 全跨 AZ） | 单 AZ 故障不中断服务，是 99.95% 的前提 |
| 合规 | 数据不出境（PH 用户数据存 Manila）；泰国若独立部署则数据存 Bangkok | 菲律宾 DPA（Data Privacy Act 2012）、泰国 PDPA 要求 |

**生产拓扑图：**

```mermaid
flowchart TB
    subgraph USERS["用户 (菲律宾 / 泰国)"]
        PH[PH 创作者]
        TH[TH 创作者]
    end

    subgraph EDGE["边缘接入"]
        DCDN["阿里云 DCDN<br/>PH/TH 边缘节点"]
        WAF["WAF + CC 防护"]
        GA["全球加速 GA<br/>(TH API 流量专线回源)"]
    end

    subgraph AZ1["Manila ap-southeast-6 · AZ-A"]
        ALB["ALB (跨 AZ)"]
        subgraph ACK["ACK 集群 / ECS 集群"]
            FE1["front nginx ×2"]
            API1["backend API ×2"]
            WK1["celery-worker ×2"]
        end
        RDS_M[("RDS MySQL 主")]
        TAIR_M[("Tair Redis 主")]
    end

    subgraph AZ2["Manila · AZ-B"]
        API2["backend API ×1 (standby)"]
        WK2["celery-worker ×1"]
        RDS_S[("RDS 备(自动切换)")]
        TAIR_S[("Tair 备")]
    end

    subgraph SHARED["区域共享服务"]
        OSS[("OSS 对象存储<br/>帧图/视频/素材")]
        SLS["SLS 日志服务"]
        ARMS["ARMS Prometheus + Grafana"]
        ACR["ACR 镜像仓库"]
        NAS["NAS(OSS 挂载可选)"]
    end

    subgraph EXT["外部 AI 供应商"]
        LLM_P["OpenAI 兼容 / 火山引擎<br/>(文本·图片·视频)"]
    end

    PH & TH --> DCDN
    DCDN -->|静态| FE1
    DCDN -->|API| WAF --> ALB
    TH -->|API 低延迟| GA --> ALB
    ALB --> API1 & API2
    API1 & API2 --> RDS_M & TAIR_M & OSS
    WK1 & WK2 --> TAIR_M
    WK1 & WK2 --> LLM_P
    WK1 & WK2 --> OSS & RDS_M
    RDS_M -.主备秒级切换.- RDS_S
    TAIR_M -.主备.- TAIR_S
    ACK -.stdout.-> SLS
    ACK -./metrics.-> ARMS
    ACR -.镜像拉取.-> ACK
```

**生产资源清单（初期规格，可弹性扩容）：**

| 资源 | 规格 | 数量 | 用途 |
| --- | --- | --- | --- |
| ACK 托管版（或 ECS 7 代 4c8g） | 3 worker 节点起 | 跨 2 AZ | front/backend/celery 容器 |
| ALB | 标准版 | 1（跨 AZ） | 七层负载 + 灰度分流 |
| RDS MySQL | 高可用版 4c16g + 只读 2c8g | 1 主 1 备 + 1 只读 | 业务库（读写分离：任务轮询走只读） |
| Tair (Redis) | 主从版 4G | 跨 AZ | broker / 缓存 / 限流 |
| OSS | 标准存储 + 生命周期（90 天转低频） | 1 bucket | 媒体文件 |
| DCDN + GA | 按流量 | — | PH/TH 加速 |
| SLS / ARMS / CloudMonitor | 按量 | — | 可观测 |
| NAT 网关 + EIP | 固定出口 | — | worker 访问外部 AI 供应商（固定 IP 便于供应商白名单） |

### 7.2 Docker Compose 配置

#### 7.2.1 现状基线（仓库 `deploy/compose/docker-compose.yml`，8 服务）

现有编排已包含：`mysql:9.0`（healthcheck + 数据卷）、`redis:7`、`rustfs`（S3 兼容）、`backend-init-db`（一次性 `init_db.py`）、`mysql-init-sql`（顺序执行 `backend/sql/*.sql`）、`backend`（8000，依赖健康条件）、`celery-worker`、`front`（7788→80，`BACKEND_URL` 运行时注入 env.js）。启动顺序由 `depends_on: condition` 严格保证。

#### 7.2.2 生产化 Compose（在现状基础上扩展：可观测 + 限流 + 多副本）

适用于单机试点 / staging / 小规模生产（更大规模建议 ACK，见 7.3）：

```yaml
# deploy/compose/docker-compose.prod.yml（目标态示例）
services:
  mysql:
    image: mysql:9.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./conf/mysql/my.cnf:/etc/mysql/conf.d/my.cnf:ro   # innodb_buffer_pool=60%RAM, 慢查询日志
    # 生产不暴露 3306 到公网，仅内网
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD --silent"]
      interval: 5s
      timeout: 5s
      retries: 30
    restart: always
    logging: &logrotate
      driver: json-file
      options: { max-size: "50m", max-file: "5" }

  redis:
    image: redis:7
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}", "--maxmemory", "2gb",
              "--maxmemory-policy", "noeviction"]   # broker 数据不可淘汰
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 5s
      retries: 30
    restart: always
    logging: *logrotate

  rustfs:            # staging 用；生产替换为 OSS（仅需改 S3_ENDPOINT_URL 等环境变量）
    image: rustfs/rustfs:latest
    command: ["/data"]
    environment:
      RUSTFS_ACCESS_KEY: ${RUSTFS_ACCESS_KEY}
      RUSTFS_SECRET_KEY: ${RUSTFS_SECRET_KEY}
    volumes:
      - rustfs_data:/data
    restart: always

  backend-init-db:
    image: ${REGISTRY}/jellyfish-backend:${IMAGE_TAG}
    environment: &backend-env
      DATABASE_URL: mysql+aiomysql://${MYSQL_USER}:${MYSQL_PASSWORD}@mysql:3306/${MYSQL_DATABASE}
      REDIS_HOST: redis
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      S3_ENDPOINT_URL: ${S3_ENDPOINT_URL}
      S3_REGION_NAME: ${S3_REGION_NAME}
      S3_ACCESS_KEY_ID: ${S3_ACCESS_KEY_ID}
      S3_SECRET_ACCESS_KEY: ${S3_SECRET_ACCESS_KEY}
      S3_BUCKET_NAME: ${S3_BUCKET_NAME}
      S3_PUBLIC_BASE_URL: ${S3_PUBLIC_BASE_URL}
      CORS_ORIGINS: ${CORS_ORIGINS}
      DEBUG: "false"
    command: ["uv", "run", "python", "init_db.py"]
    depends_on:
      mysql: { condition: service_healthy }
    restart: "no"

  mysql-init-sql:
    image: mysql:9.0
    depends_on:
      mysql: { condition: service_healthy }
      backend-init-db: { condition: service_completed_successfully }
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ../../backend/sql:/sql:ro
    command: ["bash", "-lc", "set -e; for f in $$(find /sql -maxdepth 1 -type f -name '*.sql' | sort); do mysql --default-character-set=utf8mb4 -h mysql -u$$MYSQL_USER -p$$MYSQL_PASSWORD $$MYSQL_DATABASE < $$f; done"]
    restart: "no"

  backend:
    image: ${REGISTRY}/jellyfish-backend:${IMAGE_TAG}
    environment: *backend-env
    deploy:
      replicas: 2                     # 双副本，配合 nginx/ALB 负载均衡
    expose: ["8000"]
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
      mysql-init-sql: { condition: service_completed_successfully }
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://127.0.0.1:8000/health').raise_for_status()"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    restart: always
    logging: *logrotate

  celery-worker:
    image: ${REGISTRY}/jellyfish-backend:${IMAGE_TAG}
    environment: *backend-env
    command: ["uv", "run", "celery", "-A", "app.core.celery_app:celery_app", "worker",
              "-l", "info", "-c", "4", "--max-tasks-per-child", "100"]
    deploy:
      replicas: 2                     # worker 水平扩容 = 提升生成吞吐
    depends_on:
      backend: { condition: service_healthy }
    restart: always
    logging: *logrotate

  front:
    image: ${REGISTRY}/jellyfish-front:${IMAGE_TAG}
    environment:
      BACKEND_URL: ${BACKEND_URL}     # 经 env.js 运行时注入，镜像与环境解耦
    expose: ["80"]
    depends_on: [backend]
    restart: always

  nginx:                              # 统一入口：路由 + L3 限流 + TLS(或交给 ALB)
    image: nginx:1.27-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf/nginx/prod.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on: [front, backend]
    restart: always

  prometheus:
    image: prom/prometheus:v2.54.0
    volumes:
      - ./conf/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom_data:/prometheus
    restart: always

  grafana:
    image: grafana/grafana:11.2.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./conf/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
    ports: ["3000:3000"]              # 仅内网/白名单访问
    restart: always

  node-exporter:
    image: prom/node-exporter:v1.8.2
    restart: always

  redis-exporter:
    image: oliver006/redis_exporter:v1.62.0
    environment:
      REDIS_ADDR: redis://redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    restart: always

volumes:
  mysql_data:
  redis_data:
  rustfs_data:
  prom_data:
  grafana_data:
```

配套 nginx 限流片段（`conf/nginx/prod.conf`）：

```nginx
limit_req_zone $binary_remote_addr zone=api_perip:10m rate=30r/m;   # 轮询类
limit_req_zone $binary_remote_addr zone=llm_perip:10m rate=10r/m;   # LLM 同步端点
limit_conn_zone $binary_remote_addr zone=conn:10m;

server {
  listen 80;
  location /api/v1/film/task_status { limit_req zone=api_perip burst=20 nodelay;
    proxy_pass http://backend; }
  location /api/v1/script_processing/ { limit_req zone=llm_perip burst=5 nodelay;
    proxy_pass http://backend; }
  location /api/ { proxy_pass http://backend;
    proxy_read_timeout 300s; }
  location / { proxy_pass http://front; }
}
```

### 7.3 应用部署形态选择

| 方案 | 适用 | 说明 |
| --- | --- | --- |
| **推荐：ACK + ALB Ingress + stable/canary 双 Deployment** | 生产（PH 主站） | 满足 99.95%（多副本跨 AZ + PDB + `maxUnavailable=0`）、KEDA 按队列深度扩缩 Celery、ALB 金丝雀注解原生支持灰度门禁；清单见 7.4 |
| 备选：ECS × N + Docker Compose（7.2.2） | 成本敏感、单区域试点、staging | 需自建 nginx 负载均衡与健康摘除；滚动发布靠逐台 `up -d --no-deps`，灰度粒度粗；可用性 ≈ 99.5~99.9%（受单机上限约束） |
| 不推荐：SAE / 函数计算 FC | — | Celery 长任务（视频 1~10 min、1800 s 超时）与 Serverless 请求超时/冷启动模型不匹配；worker 常驻消费与按请求计费冲突 |
| 不推荐：单实例 ECS + Compose（现状形态） | 仅本地开发 | 任何滚动发布都是停机；单点故障直接击穿 SLA，无法达成 99.95% |

**Jellyfish 特有的形态约束：**

1. **API 与 worker 必须独立 Deployment**：视频长任务在 worker 侧执行，API 只入队——两者扩缩容驱动指标不同（API 看 QPS/CPU，worker 看队列深度），合并部署会导致资源浪费或互相挤占；
2. **迁移必须与发布解耦**：`init_db.py`/SQL 迁移作为 ArgoCD **PreSync hook Job** 先行执行，schema 就绪后才滚动新代码（配合 5.6 扩展式迁移规则，旧代码可继续运行，保证 4.5 回滚成立）;
3. **front 静态站可独立发布**：nginx 容器只依赖 `BACKEND_URL` 运行时注入（env.js），前后端镜像版本可独立灰度与回滚；
4. **worker 出口需固定 EIP**（NAT 网关）：供应商 API 白名单与 `provider:health` 熔断统计都依赖稳定出口。

### 7.4 ACK 部署清单（YAML）

> 目标态清单（GitOps 仓库 `deploy/aliyun/`，ArgoCD 管理）。镜像地址格式：`registry-vpc.ap-southeast-6.aliyuncs.com/jellyfish/<app>:<git-sha>`（ACR VPC 内网拉取，禁用 latest）。

#### 7.4.1 命名空间、配置与密钥

```yaml
# deploy/aliyun/00-namespace-config.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jellyfish
  labels: { "app.kubernetes.io/name": "jellyfish" }
---
apiVersion: v1
kind: ConfigMap
metadata: { name: jellyfish-env, namespace: jellyfish }
data:
  # ---- 运行时 ----
  DEBUG: "false"
  API_V1_PREFIX: "/api/v1"
  TZ: "Asia/Manila"                  # 曼谷站点改 Asia/Bangkok；DB 时间戳统一 UTC（见 5.5）
  CORS_ORIGINS: "https://app.example-ph.com,https://app.example-th.com"
  # ---- Redis / Celery ----
  REDIS_HOST: "tair-mnl.redis.rds.aliyuncs.com"
  REDIS_PORT: "6379"
  REDIS_DB: "0"
  CELERY_BROKER_URL: ""              # 留空则由 REDIS_* 拼装（app/config.py 缺省行为）
  # ---- 对象存储（生产 = OSS，S3 兼容协议不变，仅换 endpoint）----
  S3_ENDPOINT_URL: "https://oss-ap-southeast-6-internal.aliyuncs.com"   # 内网 endpoint 免流量费
  S3_REGION_NAME: "ap-southeast-6"
  S3_BUCKET_NAME: "jellyfish-mnl-media"
  S3_BASE_PATH: "prod"
  S3_PUBLIC_BASE_URL: "https://media-cdn.example-ph.com"                # CDN 域名，非 OSS 直链
  # ---- 可观测（目标态，见 10.2 O2/O3）----
  LOG_FORMAT: "json"
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://arms-collector.jellyfish:4317"
---
apiVersion: v1
kind: Secret
metadata: { name: jellyfish-secret, namespace: jellyfish }
type: Opaque
stringData:
  # 下列值由 KMS 凭据管家经 ExternalSecret / RRSA 注入，禁止写进 Git（见 10.1 S4）
  DATABASE_URL: "mysql+aiomysql://jellyfish:REPLACE_ME@rm-mnl-jellyfish.mysql.rds.aliyuncs.com:3306/jellyfish"
  DATABASE_READ_URL: "mysql+aiomysql://jellyfish_ro:REPLACE_ME@rr-mnl-jellyfish.mysql.rds.aliyuncs.com:3306/jellyfish"
  REDIS_PASSWORD: "REPLACE_ME"
  S3_ACCESS_KEY_ID: "REPLACE_ME"        # 目标态：RRSA 免 AK 访问 OSS，boto3 走 STS
  S3_SECRET_ACCESS_KEY: "REPLACE_ME"
  OPENAI_API_KEY: "REPLACE_ME"          # 供应商 key 真相在 DB providers 表（KMS 加密），env 仅兜底
```

> **供应商密钥不进 ConfigMap**：Jellyfish 的 provider api_key/secret 存 `providers` 表（现状明文，改进为 KMS 加密列，见 10.1 S4），换 key 是 DB 操作而非发布操作——这是与 new-api 类网关不同的关键点，密钥轮转不需要滚动 Pod。

#### 7.4.2 数据库迁移 Job（ArgoCD PreSync，先于一切滚动）

```yaml
# deploy/aliyun/05-job-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: jellyfish-migrate-APP_VERSION
  namespace: jellyfish
  annotations:
    argocd.argoproj.io/hook: PreSync          # 同步前执行，失败即中止发布
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  backoffLimit: 1                              # 迁移失败不盲重试，人工介入
  activeDeadlineSeconds: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: registry-vpc.ap-southeast-6.aliyuncs.com/jellyfish/backend:CANDIDATE_SHA
          command: ["uv", "run", "alembic", "upgrade", "head"]   # 目标态；现状过渡期为 init_db.py + sql/*.sql 幂等重放
          envFrom:
            - configMapRef: { name: jellyfish-env }
            - secretRef: { name: jellyfish-secret }
          resources: { requests: { cpu: 250m, memory: 512Mi }, limits: { cpu: "1", memory: 1Gi } }
```

> 迁移只允许**扩展式变更**（5.6）；Job 成功后 stable 旧版本仍可运行，任何时刻可执行 4.5 回滚而无需回退 schema。

#### 7.4.3 backend stable / canary 双 Deployment

```yaml
# deploy/aliyun/10-deployment-backend-stable.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-stable
  namespace: jellyfish
  labels: { app: backend, track: stable }
spec:
  replicas: 3                        # 跨 2 AZ；单 AZ 故障仍有 ≥1.5 副本承载（9.5 的 1.5 倍不变量）
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }   # 0 不可用是 99.95% 的前提
  selector:
    matchLabels: { app: backend, track: stable }
  template:
    metadata:
      labels: { app: backend, track: stable }
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"        # 需补建 instrumentator（10.2 O2）
    spec:
      serviceAccountName: jellyfish           # RRSA 绑定：OSS / SLS / KMS 免 AK
      terminationGracePeriodSeconds: 60
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: backend } }
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector: { matchLabels: { app: backend } }
              topologyKey: kubernetes.io/hostname
      containers:
        - name: backend
          image: registry-vpc.ap-southeast-6.aliyuncs.com/jellyfish/backend:STABLE_SHA
          imagePullPolicy: IfNotPresent
          command: ["uv", "run", "uvicorn", "app.main:app",
                    "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
          ports:
            - { containerPort: 8000, name: http }
          envFrom:
            - configMapRef: { name: jellyfish-env }
            - secretRef: { name: jellyfish-secret }
          readinessProbe:
            httpGet: { path: /health, port: 8000 }      # 目标态换 /health/ready（探测 DB/Redis/OSS，10.2 O3）
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2                          # 快速摘流
          livenessProbe:
            httpGet: { path: /health, port: 8000 }       # 目标态换 /health/live（轻量，不查依赖）
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:
            httpGet: { path: /health, port: 8000 }       # 覆盖启动注册（bootstrap provider+adapter）耗时
            failureThreshold: 30
            periodSeconds: 2
          resources:
            requests: { cpu: "1", memory: 2Gi, ephemeral-storage: 10Gi }
            limits:   { cpu: "2", memory: 4Gi }
          lifecycle:
            preStop:
              exec: { command: ["/bin/sh", "-c", "sleep 15"] }   # 等 ALB 摘除后端再收 SIGTERM
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: backend-stable, namespace: jellyfish }
spec:
  minAvailable: 2                    # 3 副本允许 1 个维护性驱逐
  selector: { matchLabels: { app: backend, track: stable } }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: backend-stable, namespace: jellyfish }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: backend-stable }
  minReplicas: 3
  maxReplicas: 8
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600
      policies: [ { type: Pods, value: 1, periodSeconds: 120 } ]
```

```text
canary 与 stable 的唯一差异：labels/track=canary、replicas=1、PDB minAvailable=1、
镜像为 CANDIDATE_SHA、ALB 服务器组独立、不打 HPA（权重由灰度 Ingress 控制）。
发布顺序：migrate Job → canary 1 副本就绪 → ALB 权重 5% → 门禁 → stable 滚动 → canary 归零保留 24h 热回滚位。
```

#### 7.4.4 Celery worker：按队列拆分 + KEDA 队列深度扩缩

```yaml
# deploy/aliyun/11-deployment-worker-video.yaml（视频长任务独立池，防阻塞轻任务，10.4 T2）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-video
  namespace: jellyfish
  labels: { app: worker, queue: video }
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }   # 在途任务由 acks_late + reaper 兜底，仍要求 0 不可用
  selector:
    matchLabels: { app: worker, queue: video }
  template:
    metadata:
      labels: { app: worker, queue: video }
    spec:
      serviceAccountName: jellyfish
      terminationGracePeriodSeconds: 1900               # > 任务硬超时 1800s：SIGTERM 后允许在途任务收尾
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: worker } }
      containers:
        - name: worker
          image: registry-vpc.ap-southeast-6.aliyuncs.com/jellyfish/backend:STABLE_SHA
          command: ["uv", "run", "celery", "-A", "app.core.celery_app:celery_app",
                    "worker", "-l", "info", "-Q", "queue:video",
                    "-c", "4", "--max-tasks-per-child", "100"]
          envFrom:
            - configMapRef: { name: jellyfish-env }
            - secretRef: { name: jellyfish-secret }
          resources:
            requests: { cpu: "1", memory: 2Gi }
            limits:   { cpu: "2", memory: 4Gi }         # 帧图 base64 物化是内存峰值来源（5.5 D6）
          livenessProbe:                               # celery inspect ping 周期探活
            exec:
              command: ["uv", "run", "celery", "-A", "app.core.celery_app:celery_app", "inspect", "ping", "-t", "10"]
            initialDelaySeconds: 60
            periodSeconds: 60
            timeoutSeconds: 15
            failureThreshold: 3
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: worker-video, namespace: jellyfish }
spec:
  scaleTargetRef: { name: worker-video }
  minReplicaCount: 3
  maxReplicaCount: 16                                   # 9.2 峰值并发视频任务 400 / (4 槽位×副本) 的弹性上限
  cooldownPeriod: 300
  triggers:
    - type: redis
      metadata:
        addressFromEnv: "REDIS_HOST:REDIS_PORT"
        passwordFromEnv: "REDIS_PASSWORD"
        listName: "queue:video"
        listLength: "100"                               # 每副本目标积压 ≤ 100（9.5 触发阈值）
---
# worker-light（图片/文本队列）同构：-Q queue:image,queue:text，-c 16，
# replicas 2、KEDA min 2 / max 8、listLength 200；轻任务池与视频池资源隔离，
# 视频风暴时图片任务 P95 不受影响（9.6 S2 验收项）。
```

#### 7.4.5 front Deployment 与 Service

```yaml
# deploy/aliyun/12-deployment-front.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front
  namespace: jellyfish
  labels: { app: front }
spec:
  replicas: 2
  selector: { matchLabels: { app: front } }
  template:
    metadata: { labels: { app: front } }
    spec:
      containers:
        - name: front
          image: registry-vpc.ap-southeast-6.aliyuncs.com/jellyfish/front:STABLE_SHA
          ports: [ { containerPort: 80, name: http } ]
          env:
            - name: BACKEND_URL            # entrypoint 生成 env.js → window.__ENV（镜像与环境解耦）
              value: "https://api.example-ph.com/api"
          readinessProbe: { httpGet: { path: /, port: 80 }, periodSeconds: 5 }
          resources: { requests: { cpu: 100m, memory: 128Mi }, limits: { cpu: 500m, memory: 256Mi } }
---
apiVersion: v1
kind: Service
metadata: { name: backend, namespace: jellyfish }
spec:
  selector: { app: backend }           # 同时选中 stable+canary；灰度权重由 ALB Ingress 服务器组控制
  ports: [ { port: 80, targetPort: 8000 } ]
---
apiVersion: v1
kind: Service
metadata: { name: front, namespace: jellyfish }
spec:
  selector: { app: front }
  ports: [ { port: 80, targetPort: 80 } ]
```

#### 7.4.6 AlbConfig 与 ALB Ingress（含金丝雀灰度）

```yaml
# deploy/aliyun/20-alb-ingress.yaml
apiVersion: alibabacloud.com/v1
kind: AlbConfig
metadata: { name: jellyfish-alb }
spec:
  config:
    name: jellyfish-alb-mnl
    addressType: Internet
    addressAllocatedMode: Dynamic
    zoneMappings:                       # 至少 2 AZ，是 8.2 可用性推导的前提
      - { vSwitchId: vsw-az-a-REPLACE_ME }
      - { vSwitchId: vsw-az-b-REPLACE_ME }
  listeners:
    - { port: 443, protocol: HTTPS, certificates: [ { CertificateId: REPLACE_ME } ] }
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata: { name: jellyfish-alb }
spec:
  controller: ingress.k8s.alibabacloud/alb
  parameters:
    apiGroup: alibabacloud.com
    kind: AlbConfig
    name: jellyfish-alb
---
# 主 Ingress：全部流量 → stable（不带 canary 注解）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jellyfish-main
  namespace: jellyfish
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/healthcheck-path: "/health"
    alb.ingress.kubernetes.io/backend-scheduler: "wrr"
spec:
  ingressClassName: jellyfish-alb
  rules:
    - host: api.example-ph.com
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: backend, port: { number: 80 } } } }
    - host: app.example-ph.com
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: front, port: { number: 80 } } } }
---
# 灰度 Ingress：canary 注解必须在独立 Ingress 上；权重由 GitOps 逐步改 5 → 25 → 50 → 100
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jellyfish-canary
  namespace: jellyfish
  annotations:
    alb.ingress.kubernetes.io/canary: "true"
    alb.ingress.kubernetes.io/canary-weight: "5"
    alb.ingress.kubernetes.io/canary-by-header: "X-Canary"        # 内部白名单先行验证（4.3）
    alb.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  ingressClassName: jellyfish-alb
  rules:
    - host: api.example-ph.com
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: backend-canary, port: { number: 80 } } } }
```

> **Celery worker 的灰度不走 ALB**（worker 不接 HTTP 流量）：按 4.3 时序，API 侧按 `project_id hash` 把灰度比例内的新任务投递 `queue:v2`，新版本 worker 消费 v2、旧版本排空 v1 后下线；`generation_tasks.executor_type/executor_task_id` 保证两代 worker 状态写回同一真相层。

### 7.5 发布流水线与发布集成

```mermaid
flowchart LR
  A["PR 合并 main"] --> B["CI: pylint + pytest<br/>tsc --noEmit + openapi:gen diff<br/>commit-msg 校验（现有 workflows）"]
  B --> C["构建镜像（ghcr-images.yml 现有）<br/>tag = git sha"]
  C --> D["同步 ACR Manila（VPC 内网拉取）<br/>+ cosign 签名"]
  D --> E["迁移预演：对生产快照影子库<br/>alembic upgrade head 跑两次<br/>验证幂等与前向兼容"]
  E --> F{"Schema 兼容检查<br/>禁止同窗口破坏性变更（5.6）"}
  F -->|不通过| G["阻断发布<br/>拆分为兼容发布 + 清理发布"]
  F -->|通过| H["GitOps 提交：canary 镜像 sha<br/>ArgoCD 自动同步"]
  H --> I["PreSync: migrate Job（7.4.2）"]
  I --> J["canary 1 副本 startup/readiness 通过"]
  J --> K["ALB 权重 5% + X-Canary 白名单"]
  K --> L{"自动门禁 30min<br/>5xx / P99 / 任务成功率 / 取消收敛"}
  L -->|失败| M["权重归零 + P1 告警（4.5 回滚时序）"]
  M --> N["人工研判：回滚或前向修复"]
  L -->|通过| O["权重 25 → 50 → 100"]
  O --> P["stable 滚动升级 maxUnavailable=0"]
  P --> Q["worker 灰度：queue:v2 比例同步放量<br/>v1 排空后下线旧 worker"]
  Q --> R["canary 归零权重，保留 24h 热回滚位"]
  R --> S["发布记录：镜像 sha、迁移版本、<br/>Nacos 配置变更清单 → site/content/blog release note"]
```

**发布窗口与冻结策略**：发布窗口固定 **马尼拉/曼谷时间 02:00–05:00**（当地低峰）；避开菲律宾发薪日（每月 15/30 日）与 PH/TH 公共假期前夜（短剧消费高峰）；错误预算燃尽 > 80% 自动冻结非必要发布（8.3）。

**配置与备份策略**（随发布集成管理）：

| 项 | 方案 |
| --- | --- |
| 动态配置 | Nacos：默认模型（`model_settings`）、供应商开关（`providers.status`）、限流阈值、灰度开关热更 ≤ 60 s，变更走审计 |
| 备份 | RDS 每日全量 + binlog（保留 30 天，PITR）；OSS 版本化 + 跨区域复制 Bangkok（可选）；ACK 集群 etcd 自动备份；GitOps 仓库即配置备份 |
| 镜像保留 | ACR 保留最近 30 个 tag（≥ 90 天），保证任意历史版本可回滚重建 |

### 7.6 环境矩阵与发布验证（详细方案）

#### 7.6.1 环境矩阵

| 环境 | 位置/形态 | 数据与依赖 | 用途 | 晋级门禁 |
| --- | --- | --- | --- | --- |
| dev | 本地 Compose（7.2.1，SQLite/MySQL + RustFS + mock provider） | 种子数据 | 功能开发 | `uv run pylint app`、`uv run pytest`、`pnpm exec tsc --noEmit` 全绿 |
| ci | GitHub Actions（现有 5 条 workflow + 目标态补充） | 无状态 | 静态检查/单测/契约 | 单测通过 + OpenAPI diff 无 breaking（E4）+ 镜像构建成功 |
| staging | Manila ACK 独立 namespace `jellyfish-staging`（1 副本 backend/worker） | RDS MySQL 测试实例 + Tair 测试实例 + OSS 测试 bucket；LLM 用 `tests/support/llm_api_app.py` mock + 少量真实供应商配额 | 集成验证、E2E、迁移预演 | 迁移预演幂等通过 + 冒烟用例（7.6.3）全过 + E2E 主链路通过 |
| perf | 复用 staging 拓扑 + k6 压测机 + mock 供应商（可注入延迟/失败） | 数据量预灌至生产 6 个月规模 | 第 9 章 S1~S5 压测 | 相对上次基线劣化 > 10% 阻断发布 |
| prod-mnl | Manila ACK（7.4 全清单） | RDS HA + Tair 主从 + OSS + 双供应商 | 菲律宾流量 | 灰度门禁（7.6.4）→ 全量 |
| prod-bkk（未来） | Bangkok ACK（TH 用户占比 > 40% 时启用） | 独立全套，数据不出境（PDPA） | 泰国流量 | 与 mnl **错峰发布**（先 mnl 稳定 48h 后 bkk） |
| dr | Bangkok 冷备（RDS 跨区备份 + OSS 跨区复制 + GitOps 一键重建） | 备份数据 | 区域级灾备 | 每季度切换演练（8.6） |

#### 7.6.2 发布验证阶段（V0~V5，每阶段有明确出口条件）

| 阶段 | 时机 | 验证内容 | 出口条件 | 失败动作 |
| --- | --- | --- | --- | --- |
| V0 制品校验 | CI 产出后 | 镜像 tag = git sha、cosign 签名验证、`kustomize/helm diff` 人工评审清单变更 | 签名有效 + diff 双人审批 | 阻断进入 staging |
| V1 迁移预演 | staging | 生产快照影子库上 `alembic upgrade head` **连跑两次**；校验旧版本代码可在新 schema 上运行（前向兼容） | 两次结果一致 + 旧镜像启动正常 | 拆分为兼容发布 + 清理发布 |
| V2 部署校验 | staging → prod canary | Pod 全部 Ready、startupProbe 通过、无 CrashLoop、`/health` 200、日志无 ERROR 突增、`bootstrap` provider/adapter 注册完整 | 5 min 观察窗全绿 | 中止发布，镜像回退 |
| V3 冒烟验证 | canary 就绪后 | 7.6.3 自动化冒烟用例（canary 与 stable 各跑一遍，对比行为一致性） | 用例 100% 通过 | 权重归零回滚 |
| V4 灰度门禁 | 权重 5% 起 30 min | 7.6.4 指标表自动对比 canary vs stable | 全部指标在阈值内 | 自动权重归零 + P1 告警（4.5） |
| V5 全量回归 | 100% 后 24 h | E2E 主链路（编辑页→ready→工作室→生成→任务中心）、任务对账（无悬挂 running）、SLO 大盘无劣化、release note 落 `site/content/blog` | 24 h 观察期结束 | 24 h 内保留 canary 热回滚位 |

#### 7.6.3 自动化冒烟用例（V3，staging 与 prod canary 共用）

| # | 用例 | 端点/操作 | 断言 |
| --- | --- | --- | --- |
| 1 | 健康检查 | `GET /health` | 200，响应 < 500 ms |
| 2 | OpenAPI 契约 | `GET /openapi.json` | schema 可解析，与 CI 产物 hash 一致 |
| 3 | 项目创建 | `POST /api/v1/studio/projects` | 200，`ApiResponse.code` 成功语义 |
| 4 | 章节拆分（mock LLM） | `POST /api/v1/script_processing`（拆分） | shots 落库，`status=pending` |
| 5 | 候选提取与确认 | 抽取 → accept 一个资产候选 | 候选进入正式资产表，`shot_character_links` 建立 |
| 6 | shot ready 流转 | 完成全部确认 → readiness 检查 | `shots.status=ready` |
| 7 | 图片任务提交（mock provider） | `POST /api/v1/film` 图片生成 | 202 + `task_id`，DB `pending→running→succeeded` |
| 8 | 任务轮询 | `GET /api/v1/film/task_status` | 状态/进度单调推进，P99 < 200 ms |
| 9 | 任务取消 | 提交长任务 → cancel | `cancel_requested` 置位，终态 `cancelled`（双通道收敛，6.3） |
| 10 | 文件上传下载 | 上传素材 → OSS → 读取 | object_key 可访问，`file_usages` 引用正确 |
| 11 | 视频就绪检查 | `video-readiness`（缺帧图场景） | 返回缺失项清单而非 500 |
| 12 | 限流行为（prod 开启后） | 超频调用 `task_status` | 429 + `retry_after`，不产生 5xx |

> 用例 4/7 在 staging 用 mock 供应商；prod canary 用**真实供应商小额配额**跑一次真实图片生成（用例 7 变体），验证出口 EIP、供应商凭据与能力表在生产环境可用——这是 staging 无法覆盖的关键差异点。

#### 7.6.4 灰度门禁指标（V4，自动对比 canary vs stable）

| 指标 | 数据源 | 中止阈值（任一触发即自动回滚） |
| --- | --- | --- |
| HTTP 5xx 比例 | ALB 访问日志 / Prometheus | canary > stable + 0.5pp 或绝对值 > 1% |
| API P99 延迟 | Prometheus histogram | canary > stable × 1.3 且绝对值 > 2 s |
| 任务提交成功率 | task-log（started 事件/请求数） | canary < stable − 2pp |
| 任务执行成功率（同 task_kind 维度） | `generation_tasks` 终态统计 | canary < stable × 0.8 |
| 取消收敛率 | cancelled 终态 vs cancel_requested | 存在 > 5 min 未收敛任务即告警 |
| Panic/未捕获异常 | SLS `level=ERROR` 关键字 | canary 新增异常签名（相对 stable）≥ 1 种 |
| Pod 重启次数 | K8s event | canary 重启 ≥ 2 次 |

### 7.7 监控集成（阿里云侧）

| 数据面 | 阿里云产品 | 接入方式 | 保留 |
| --- | --- | --- | --- |
| 指标 | ARMS Prometheus 版 | ACK 装 arms-prometheus 组件，ServiceMonitor 抓 backend `/metrics`（需补建 instrumentator，10.2 O2）+ celery-exporter | 90 天热 + 降采样 |
| 日志 | SLS | Logtail DaemonSet 采容器 stdout（JSON），四 logstore 分流（4.1） | 热 30 天 / 冷 180 天 |
| 链路 | ARMS / OTel | OTel SDK 贯通 API → Celery headers → 供应商调用（trace_id 透传，4.1） | 30 天 |
| 看板 | Grafana 托管版 | 三块大盘：SLA/SLO 燃尽（8.3）、生成链路健康（task_kind × provider）、容量与成本（9.5 指标） | — |
| 拨测 | 云监控站点监控 | 探测点：马尼拉、曼谷、新加坡；探测 `/health` + 关键只读 API；断言 200 与响应体版本字段 | 15 个月 |
| 前端 RUM | ARMS RUM | front 注入 SDK：页面性能、JS 错误、任务中心轮询失败率（PH/TH 分地域） | 30 天 |
| 告警 | ARMS 告警 + 云监控 | 规则见 4.2 表；P1 电话 + 钉钉，P2 钉钉群；值班表轮转 | — |
| 审计合规 | ActionTrail + RDS SQL 洞察 | 云 API 变更全留痕；慢 SQL/异常 SQL 审计（5.4 治理闭环） | 180 天 |

### 7.8 SLA 与容量总览

SLA 99.95% 的度量口径、可用性推导、错误预算管理、故障域韧性设计与灾备演练详见**第 8 章**；业务容量模型、各层容量核算、扩缩容触发条件与压测验收标准详见**第 9 章**。部署形态（7.3~7.4）是二者的载体：多副本跨 AZ、RDS 高可用、Tair 主从是达成 99.95% 的最低配置前提。

---

## 8. SLA 99.95% 达成方案

### 8.1 SLA 定义与度量口径

**必须先定义清楚，否则 99.95% 无法验收：**

| 项 | 定义 |
| --- | --- |
| 服务窗口 | 7×24，按自然月统计（30 天 = 43,200 分钟） |
| 不可用判定 | 在 PH/TH 探测点，对 `/health` 或核心 API（`GET /api/v1/film/task_status` 等只读端点）**连续 2 个 15 s 周期**返回 5xx / 连接失败 / 超 10 s 无响应 |
| 计划内维护 | 提前 72 h 公告、每月 ≤ 30 min，安排在当地低峰（02:00–05:00 PHT/ICT）；必须通过灰度实现**零停机**（零停机维护不计入不可用） |
| 部分降级 | 成功率 < 99.5% 或 P95 超阈值持续 5 min：计入 **SLO 违约**（错误预算单列跟踪），不计入"完全不可用" |
| 排除项 | ① AI 供应商自身故障（平台提供重试/换供应商机制，生成失败计入业务 SLO 而非平台 SLA）；② 阿里云公告的区域级 IaaS 故障；③ 客户侧网络/浏览器问题；④ 超出限流配额的 429；⑤ 生成任务本身的业务性失败（提示词违规等） |
| **月度不可用预算** | 43,200 × 0.05% = **21.6 分钟**（周预算 5.04 分钟，日预算 0.72 分钟） |

**计量合成**：`平台可用率 = f(ALB 入口 5xx 比例, /health 拨测可用率)`，由 ARMS + 云拨测（PH/TH 双探测点）出月报；**生成成功率（目标 ≥ 95%）单列为业务 SLO**，与平台 SLA 分离——避免供应商侧故障击穿平台赔付承诺。

### 8.2 可用性推导（为什么必须按第 7 章拓扑部署）

| 环节 | 配置 | 期望可用性 | 备注 |
| --- | --- | --- | --- |
| DNS + GA/DCDN | 健康探测 + 多区域边缘 | 99.99% | TH 流量经 GA 专线回源，公网劣化不放大 |
| WAF + ALB | 阿里云多 AZ 实例 | 99.99% | 长任务提交接口超时需要 ≥ 300 s 配置，避免"假不可用" |
| ACK 控制面 | Pro 托管版多 AZ | 99.95% | 控制面故障不影响已运行 Pod 数据面 |
| API 数据面 | ≥ 3 副本跨 2 AZ、readiness 探针、PDB、preStop 排空 | 99.99% | 单实例 99.5% × 3 副本并联 ≈ 99.9999% |
| Celery 执行面 | ≥ 3 worker 跨 2 AZ + KEDA 队列深度扩缩 | 99.9% | **worker 全挂不阻塞 API 可用性**（任务只入队不执行，计入业务 SLO 降级而非平台不可用） |
| 主库 | RDS 高可用版跨 AZ 主备自动切换 | 99.99% | 切换 ≤ 30 s，写失败由客户端重试吸收；`pool_pre_ping` 加速连接收敛 |
| 缓存/broker | Tair 主从跨 AZ | 99.95% | Redis 故障时**限流 fail-open、任务提交快速失败 503**（见 5.8），不放大为 API 全站 5xx |
| 对象存储 | OSS（区域内多 AZ 冗余） | 99.995% | 产物转存失败任务标记可重试，不丢真相（DB 先行） |
| AI 供应商 | OpenAI 兼容 + 火山引擎双活 | — | 不计入平台 SLA；`provider:health` 熔断 + 自动换供应商重试 1 次 |

**串联（近似独立）**：`0.9999(DNS/ALB) × 0.9999(API) × 0.9999(RDS) × 0.9995(Tair) × 0.99995(OSS) ≈ 0.9992`——注意：**Tair 与读路径是短板**。要稳定站上 99.95% 必须做到：

1. 读路径（任务轮询/列表）走只读实例 + Redis 2 s 缓存，**Redis 故障时缓存穿透到只读库而非报错**（降级放行）；
2. 生成链路（Celery → 供应商）与 API 链路完全异步解耦（当前架构已满足），供应商慢/挂不拖垮 API；
3. 消除单点：现状 compose 单机形态 ≈ 99.5%（不达标），生产必须 7.3/7.4 的多副本跨 AZ ACK 形态；
4. 留 ~0.03% 预算给"人因与变更"（业界最大故障源），靠灰度 + 自动回滚（4.3/4.5）压缩。

### 8.3 错误预算管理

```mermaid
flowchart TD
    A["每月错误预算 21.6 分钟"] --> B["实时消耗统计<br/>5min 窗口累计不可用 + SLO 违约"]
    B --> C{"燃尽比例"}
    C -->|"< 50%"| D["正常发布节奏"]
    C -->|"50% ~ 80%"| E["发布双人审批<br/>灰度观察窗口 30min → 60min"]
    C -->|"> 80%"| F["冻结非必要发布<br/>只允许修复与止血变更"]
    C -->|"燃尽"| G["停止所有变更进入稳定期<br/>触发正式事件复盘（含供应商侧根因）"]
    D --> H["月度复盘：按故障源分类<br/>（变更/依赖/容量/供应商）→ 反哺门禁与预案"]
    E --> H
    F --> H
    G --> H
```

### 8.4 故障域与韧性设计

| 故障域 | 检测 | 自动处置 | 人工处置 | RTO |
| --- | --- | --- | --- | --- |
| 单容器/Pod 崩溃（OOM/panic） | readiness 失败 + restart 计数告警 | K8s 重启、ALB 30 s 内摘除 | 看 core dump / 内存剖析 | 秒级 |
| 单节点故障 | node NotReady | Pod 反亲和 + ACK 自动重调度到另一 AZ | 扩容节点池 | ≤ 3 min |
| 单 AZ 故障 | CloudMonitor + 拨测 | ALB/RDS/Tair 跨 AZ 自动切换；HPA 在存活 AZ 扩容 | 确认容量水位 | ≤ 5 min |
| RDS 主库故障 | 连接错误 + RDS HA 事件 | 主备自动切换（≤ 30 s）；应用 `pool_pre_ping` 重建连接 | 切换后校验任务状态一致性（reaper 对账） | ≤ 1 min |
| Tair/Redis 故障 | `redis_up == 0` | 限流降级 fail-open；轮询缓存穿透只读库；**新任务提交 503 + Retry-After**（broker 不可用） | 立即恢复 Tair；期间积压任务恢复后重放 | 分钟级 |
| Celery worker 全部失活 | 队列深度 > 500 且消费速率 = 0 | KEDA 扩容；reaper 把 executor 失活的 `running` 任务标记 failed(可重试) | 检查镜像/broker 凭据 | ≤ 10 min |
| AI 供应商故障/配额耗尽 | `provider:health` 失败率窗口 + 任务失败率告警 | 熔断该 provider → 自动路由到备用 provider 重试 1 次 | 调整 `model_settings` 默认模型（≤ 60 s 生效） | ≤ 10 min |
| OSS 故障/限流 | 转存失败率告警 | 任务重试（指数退避）；产物 URL 暂存 result 待补转 | 提工单；必要时临时切备用 bucket | 分钟级 |
| 发布引入故障 | 灰度指标对比（4.3） | ALB 权重自动归零回滚（4.5） | 复盘 + 前向修复 | ≤ 5 min |
| 配置误改（默认模型/供应商） | 变更后 5 min 成功率劣化 | 无自动识别（缺配置基线）→ 依赖审计日志 | Nacos 历史版本一键回退 | ≤ 2 min |
| 流量突增/CC 攻击 | WAF/ALB QPS 异常 | CDN/WAF 吸收 + 四层限流（4.4）+ HPA 扩容 | 封禁恶意 IP 段 | 自动 |
| NAT 出口故障（worker → 供应商） | 出口探测失败 | 多 EIP 冗余切换 | 检查 NAT 网关配额 | ≤ 5 min |
| 磁盘/存储打满 | 使用率 > 85% 告警 | RDS 存储自动扩容；OSS 无上限；日志按 4.1 生命周期投递 | 清理临时产物 | 分钟级 |
| 区域级灾难（Manila 全区） | GTM/拨测连续失败 | DNS 切 Bangkok 冷备（文档化预案，非自动） | 按 DR runbook 重建（GitOps + RDS 跨区备份 + OSS 跨区复制） | ≤ 4 h（RPO ≤ 15 min） |

### 8.5 分级 SLO 分解（内部承诺）

| 服务 | SLO | 违约动作 |
| --- | --- | --- |
| API 网关（读路径） | 可用率 99.95%，P99 ≤ 500 ms | 燃尽预警 → 冻结发布 |
| API 网关（写路径/任务提交） | 可用率 99.9%，P99 ≤ 1.5 s | 同上 |
| 任务执行（图片） | P95 端到端 ≤ 3 min，成功率 ≥ 97% | 供应商侧根因分析 |
| 任务执行（视频） | P95 端到端 ≤ 15 min，成功率 ≥ 95% | 同上 + 容量评估（第 9 章） |
| Agent 剧本处理 | P95 ≤ 90 s，成功率 ≥ 97% | 检查 LLM 配额与超时配置 |
| 数据层 | 主从延迟 P99 ≤ 5 s；慢查询 < 0.1% | 索引评审（5.4） |

### 8.6 灾备演练（季度必做）

1. **AZ 切换演练**：强制驱逐单 AZ 全部 Pod，验证另一 AZ 承载全量（容量预留 1.5 倍，见 9.5）且 TH 用户 RTT 上升 < 90 ms；
2. **RDS 主备切换 + PITR 演练**：触发 HA 切换验证应用重连 ≤ 60 s；从 binlog 恢复到新实例，跑任务状态对账（验证 RPO ≤ 5 min）；
3. **Redis 摘除演练**：断开 Tair，验证限流 fail-open、轮询穿透只读库、任务提交优雅 503，**无 5xx 雪崩**；
4. **回滚演练**：从灰度门禁失败到 ALB 权重归零，实测止血耗时（目标 < 60 s）；
5. **供应商全故障演练**：mock 双 provider 100% 失败，验证熔断、任务不重复扣配额、失败原因可追溯、reaper 收敛悬挂任务；
6. **worker 悬挂对账演练**：kill -9 全部 worker，验证 reaper 在 2 个扫描周期内把 `running` 失活任务标记 failed(可重试) 且用户可重新提交。

---

## 9. 容量设计与压测验收

### 9.1 业务容量模型（所有资源核算的输入）

| 参数 | 基线值 | 峰值系数 | 峰值推导 |
| --- | --- | --- | --- |
| 活跃创作团队（DAU） | 500 | — | PH 350 + TH 150 |
| 并发在线编辑用户 | 100 | ×3（当地 19:00–23:00 高峰） | 300 |
| 新增项目/日 | 20 | ×2 | 40 |
| 新增镜头/日 | 1.6 万 | ×2 | 3.2 万 |
| 生成任务/日（图 + 视频 + 文本） | 7 万 | ×2 | 14 万 ≈ 峰值 6 任务/s 入队 |
| 其中视频长任务（1~10 min/个） | 2 万 | ×2 | **峰值并发视频任务 ≈ 200~400**（关键容量约束） |
| API 请求（含轮询） | 日均 500 万 | — | 峰值 400 QPS（轮询占 ~70%） |
| OSS 日增量 | 50~200 GB | — | 峰值日 300 GB；视频产物占 > 80% |

> 以上为规划基线，上线后每月用真实指标（ARMS + task-log）回填校正，容量结论随之滚动更新。

### 9.2 应用层容量核算

| 组件 | 单实例基线（待压测实测替换） | 峰值需求 | 副本结论 |
| --- | --- | --- | --- |
| backend API（4c8g） | 400 QPS（读为主，P99 < 500 ms）；asyncio 并发连接 ~1000 | 400 QPS | **2 副本即可扛峰值，按 N+1 部署 3 副本跨 2 AZ** |
| Celery worker（视频队列，4c8g，`-c 4`） | 4 并发长任务/worker（IO 密集，async 桥接，1800 s 超时） | 并发视频任务 400 | **视频队列 ≈ 100 并发 worker 槽位 → 8~10 副本**；或改 gevent/提升 `-c` 至 16（IO 密集型可行）压到 3 副本，需压测验证内存 |
| Celery worker（图片/文本队列） | 16 并发/worker | 图片峰值 ~4 任务/s、文本 Agent 并发 ≤ 50 | 2~3 副本独立队列（防视频长任务阻塞轻任务，见 10.4 T2） |
| front nginx | 静态资源走 CDN，源站 < 50 QPS | — | 2 副本 |
| KEDA 扩缩 | 队列深度 > 100/worker 触发扩容；< 10 触发缩容 | — | 视频队列弹性上限 16 副本 |

### 9.3 数据层容量核算

| 资源 | 规格结论 | 核算依据 |
| --- | --- | --- |
| RDS MySQL | 高可用版 **4c16g + 500 GB ESSD PL1**（存储自动扩容开）+ 只读 2c8g | 写 TPS 峰值 < 50（任务状态节流后），读经只读实例 + 缓存 < 200 QPS；年增 < 50 GB（5.7）；buffer pool 16g × 60% 可全缓存 3 年热数据 |
| 连接数 | 主库预算 ≤ 400（5.9 不变量），只读实例 ≤ 200 | 远低于 RDS 默认 max_connections，无需代理；> 100 副本时引入 RDS 代理 |
| Tair Redis | 主从版 **4 GB**（noeviction） | 队列峰值 5000 任务 × ~2 KB ≈ 10 MB + 限流/缓存/锁 < 500 MB + Celery unacked，余量 8 倍 |
| OSS | 标准存储，**年增 20~70 TB** | 生命周期 90 天转 IA / 180 天转归档后，热存储稳态 ≈ 6~12 TB；归档层承接历史项目 |
| CDN/DCDN | 月流量 ≈ 15~40 TB | 视频产物下发为主（9.4） |
| SLS | 月写入 ≈ 50~100 GB | app/access/task/integration 四 logstore，30 天热存 + 180 天冷存 |

### 9.4 带宽与出口容量

| 链路 | 估算 | 预留 |
| --- | --- | --- |
| 产物回传（供应商 → worker → OSS） | 2 万视频/日 × 20 MB ≈ 400 GB/日 下行 + 等量上行 | NAT 出口带宽 ≥ 200 Mbps（峰值 3 倍系数 ≈ 110 Mbps） |
| CDN 用户下发 | 400 GB/日 × 命中回源比 + 直接下发 ≈ 峰值 250~400 Mbps | DCDN 按带宽峰值计费，95 计费评估 |
| worker → 供应商轮询 | 400 并发任务 × 每 5 s 一次 ≈ 80 QPS 小请求 | 固定 EIP ≥ 2 个（供应商白名单 + 出口冗余） |
| 帧图 base64 提交 | 图片任务峰值 4/s × 3 帧 × 2 MB ≈ 24 MB/s | **提交前压缩/引用化**（5.5），API 侧 body 上限 50 MB |

### 9.5 扩缩容触发条件与弹性策略

| 指标 | 扩容阈值 | 缩容阈值 | 动作 |
| --- | --- | --- | --- |
| API CPU（HPA） | > 60% 持续 3 min | < 30% 持续 15 min | ±1 副本，上限 8 |
| 视频队列深度（KEDA） | > 100/worker 槽位 | < 10 | worker ±2，上限 16 |
| API P99 | > 1 s 持续 5 min | — | 先查慢 SQL/供应商，再扩容 |
| RDS CPU / IOPS | > 70% 持续 10 min | — | 升配（在线变配 < 30 min，安排低峰） |
| Redis 内存 | > 75% | — | 升配 |
| OSS 带宽限流事件 | 出现即告警 | — | 提配额工单 / 拆分 bucket 前缀 |
| **区域容量预留不变量** | 单 AZ 必须能承载全量峰值的 **1.5 倍** | — | 每次容量评审校验（AZ 故障时另一 AZ 可扛全量） |

### 9.6 压测验收标准（k6，`loadtest/` 目录落地）

| 场景 | 负载模型 | 通过标准 |
| --- | --- | --- |
| S1 API 基线 | 400 QPS 混合（70% 读：task_status/列表；30% 写：候选确认/任务提交），阶梯加压至 800 QPS | 400 QPS 下 P99 < 500 ms（读）/ 1.5 s（写），错误率 < 0.1%；找出拐点 QPS |
| S2 任务风暴 | 并发提交 500 视频任务（mock 供应商，异步轮询模式） | 入队 202 全成功、无重复执行（幂等键校验）、队列消费无饿死（图片/文本队列 P95 不受影响） |
| S3 轮询压力 | 2000 虚拟用户 × 每 3 s 轮询 task_status | 缓存命中 > 90%，DB QPS < 100，P99 < 200 ms |
| S4 供应商劣化 | mock 供应商延迟 30 s / 50% 失败 | 熔断触发、自动换 provider、任务成功率 ≥ 90%、API 层无级联超时 |
| S5 浸泡测试 | S1 的 60% 负载 × 8 h | 无内存泄漏（RSS 增长 < 10%）、无连接泄漏、无从延迟累积 |

**验收规则**：每次大版本发布前跑 S1~S4；每夜间 CI 跑 S1 留存基线趋势；压测报告（含实测单实例基线）回填 9.2 表格替换估算值——**未经压测的数字不得写入对外 SLA 承诺**。

---

## 10. 系统不足与改进措施

> 按优先级排序（P0 = 生产化阻断项）。

### 10.1 安全（P0）

| # | 不足 | 现状证据 | 改进措施 |
| --- | --- | --- | --- |
| S1 | **API 完全无认证鉴权** | `api/` 无 auth/JWT/API key，任何人可调用全部端点（含删除、模型配置、供应商 key 管理） | 引入认证（JWT + 用户表）与 RBAC；`/llm/providers` 等管理端点加管理员角色；供应商 api_key 存储加密（KMS），响应中脱敏 |
| S2 | **无限流**，LLM/视频生成端点成本裸露 | 未引入 slowapi 等；`/script_processing` 每次调用都打 LLM | 落地 4.4 四层限流；生成类端点加用户配额与成本预算熔断 |
| S3 | CORS methods/headers 全开 | `main.py` CORS 中间件 | 收敛为白名单 origin + 必要 methods；生产经网关统一控制 |
| S4 | 供应商密钥走明文环境变量 | compose `.env` | 迁移阿里云 KMS/Secrets Manager，ACK Secret 加密存储，定期轮转 |
| S5 | 对象存储 URL 若直接暴露存在越权访问风险 | `S3_PUBLIC_BASE_URL` 直链 | 媒体访问改签名 URL（短时效）+ CDN 鉴权（URL Auth） |

### 10.2 可观测性（P0）

| # | 不足 | 改进措施 |
| --- | --- | --- |
| O1 | 无集中 logging 配置、无 trace_id 贯穿 | dictConfig 统一 JSON formatter；contextvar 注入 trace_id（4.1 方案） |
| O2 | 无 metrics/tracing | `prometheus-fastapi-instrumentator` + `celery-exporter`；OTel SDK 贯通 API→Celery→供应商调用 |
| O3 | `/health` 无依赖深度检查 | 增加 `/health/ready`（探测 DB/Redis/S3）供 ALB/K8s 就绪探针，`/health/live` 保持轻量 |

### 10.3 数据层（P1）

| # | 不足 | 改进措施 |
| --- | --- | --- |
| D1 | **无版本化迁移工具**：手写 SQL 001~008 + `create_all` 并存，环境漂移风险高，无法回滚 | 引入 Alembic，把现有 sql 收编为基线版本；CI 校验迁移可重放；发布流程强制「先迁移后发布」且只允许扩展式变更（配合 4.5 回滚） |
| D2 | 任务轮询直打主库 | 轮询/列表查询走只读实例 + Redis 短缓存（1~2s TTL），保护主库 |
| D3 | `running` 状态任务在 worker 崩溃后可能悬挂 | 增加 reaper：定期扫描 `running` 且 executor 心跳超时的任务 → 标记 failed(可重试) 或自动重入队（幂等键 = task_id） |
| D4 | 无孤儿文件治理 | `file_usages` 已具备引用关系，补充定期 GC 任务清理无引用 files 与 OSS 对象 |
| D5 | SQLite 默认配置易被误用于生产 | config 启动时校验：非 debug 模式禁止 sqlite，强制 MySQL |
| D6 | **任务 payload 快照可能包含帧图 base64**（`VideoGenerationInput` 三参考帧），单行数 MB，导致 `generation_tasks` 膨胀与主从复制延迟 | payload 中 base64 一律替换为 `file_id/object_key` 引用，提交供应商时再物化（见 5.5）；完成态任务 30 天后 payload 转存 OSS（见 5.7） |
| D7 | 索引未按查询模式设计、列表接口无分页上限、无 keyset 分页 | 落地 5.4 索引清单（P0 项先行）；Pydantic 强制 `page_size ≤ 100`；大表列表改 `(created_at, id)` 游标分页 |

### 10.4 任务与生成链路（P1）

| # | 不足 | 改进措施 |
| --- | --- | --- |
| T1 | 任务失败无自动重试策略 | Celery `autoretry_for` + 指数退避（供应商 429/5xx/超时可重试，业务校验错误不重试）；重试次数入 `generation_tasks` |
| T2 | 单队列无优先级，视频长任务会阻塞轻量图片任务 | 拆分队列：`queue:text` / `queue:image` / `queue:video`，独立 worker 池与并发配置 |
| T3 | 取消的强制通道（SIGTERM）可能留下不一致中间态 | 收敛为以协作式取消为主：适配器轮询循环内检查 cancel；SIGTERM 后由 reaper（D3）兜底对账 |
| T4 | `/script_processing` 20+ 端点为同步 Agent 调用，LLM 慢时占满 API worker | 按既有 `plans/task-async-cancellation-plan.md` 推进任务化：同步端点改为「创建任务 + 轮询/SSE」，API 层不再长阻塞 |
| T5 | 供应商单点依赖 | 双 provider 冗余 + 故障自动切换；能力表已支持 (provider, model) 维度，补充「同能力多 provider 路由策略」 |

### 10.5 工程质量与交付（P2）

| # | 不足 | 改进措施 |
| --- | --- | --- |
| E1 | **无压测体系** | 建 `loadtest/`（k6）：API 基线 / 任务风暴 / 轮询压力三场景（3.3 指标），接入 CI 夜间执行并留存基线报告 |
| E2 | 前端无自动化测试，MSW 仅用于开发 | Vitest 组件测试 + Playwright E2E 覆盖主链路（编辑页→ready→工作室→生成→任务中心） |
| E3 | 后端测试对 DB 依赖走 skip 兜底，CI 覆盖率不明 | testcontainers 提供真实 MySQL/Redis；覆盖率门禁 ≥ 60% 起步 |
| E4 | 无 API 契约防退化机制 | CI 加 OpenAPI diff（oasdiff），breaking change 阻断合并 |
| E5 | 部署仍依赖 ghcr.io（PH/TH 拉取慢且受网络影响） | 镜像同步 ACR Manila（7.5）；部署走 GitOps（ArgoCD），杜绝手工 compose up 生产 |
| E6 | compose 单机形态无高可用 | 按 7.3/7.4 迁移 ACK 多副本跨 AZ；compose 仅保留 dev/staging 用途 |
| E7 | 无用户/多租户模型，项目数据无隔离 | 随 S1 认证一并引入 user/tenant 维度与行级隔离，为多团队协作与计费打基础 |
| E8 | 文档站 plans 中多项计划未落地（生成链收敛重构等） | 按 AGENTS.md 约定保持 plans 与代码同步；本文件 10.4/10.1 与其对齐排期 |

### 10.6 改进路线图建议

```mermaid
gantt
    title 生产化改进路线（面向 PH/TH 上线）
    dateFormat  YYYY-MM-DD
    section P0 上线阻断
    认证鉴权+RBAC (S1)          :a1, 2026-11-03, 14d
    四层限流 (S2/S3)            :a2, after a1, 7d
    日志/指标/健康检查 (O1-O3)   :a3, 2026-11-03, 10d
    密钥治理 KMS (S4)           :a4, after a3, 5d
    section P1 稳定性
    Alembic 迁移收编 (D1)       :b1, after a2, 7d
    索引+分页治理 (D7)          :b5, after a2, 5d
    payload base64 引用化 (D6)  :b6, after b1, 5d
    任务重试+队列拆分+reaper (T1-T3,D3) :b2, after b1, 10d
    script_processing 异步化 (T4) :b3, after b2, 14d
    只读实例+缓存 (D2)          :b4, after b1, 5d
    section P2 交付质量
    ACK+ACR+GitOps 迁移 (E5/E6) :c1, after a4, 14d
    k6 压测体系+第9章验收 (E1)   :c2, after c1, 7d
    E2E+契约测试 (E2/E4)        :c3, after c2, 14d
    多租户 (E7)                 :c4, after c3, 21d
```

---

## 11. 附录

### 11.1 关键验证命令

```bash
# 后端
cd backend && uv run pylint app && uv run pytest            # 静态检查 + 测试
uv run python init_db.py                                     # 建表初始化
uv run celery -A app.core.celery_app:celery_app worker -l info   # 启动 worker

# 前端
cd front && pnpm install && pnpm run openapi:gen             # 同步后端契约
pnpm exec tsc --noEmit && pnpm dev                           # 类型检查 + 开发服务器(7788)

# 部署
cd deploy/compose && docker compose up -d                    # 本地/staging 一键起 8 服务
curl http://localhost:8000/health                            # 健康检查
curl http://localhost:8000/docs                              # OpenAPI 文档
```

### 11.2 术语表

| 术语 | 含义 |
| --- | --- |
| `shot.status` | 镜头**信息提取确认状态**（pending/ready），不表达运行时生成状态 |
| video-readiness | 独立的**视频生成准备度**检查（帧图/参数/提示词齐备性） |
| 任务真相层 | `generation_tasks` + `generation_task_links`，任务状态的唯一事实来源（DB） |
| 执行层 | Celery + Redis broker，仅负责任务执行编排 |
| Base Draft / Context / Derived Preview / Submission | 生成工作区 4 层模型：基础草稿 → 上下文构建 → 派生预览 → 提交参数 |
| 候选（candidate） | AI 抽取的待确认资产/对白，经 accept 后才进入正式资产表 |
| provider_key | 供应商标识（openai / volcengine），与 model 组成适配器注册双键 |

### 11.3 SLA 承诺口径（对外建议文本）

- **平台可用性 99.95%/月**：以入口网关 5xx 比例与 `/health` 探测合成计量；不含计划内维护（提前 72h 公告、安排在当地低峰 02:00–05:00 PHT/ICT）。
- **生成成功率 SLO ≥ 95%**（业务指标，非 SLA 赔付口径）：受第三方 AI 供应商影响的失败，系统提供自动换供应商重试与手动重试入口。
- **RPO/RTO**：数据 RPO ≤ 5min（binlog）；发布回滚 RTO ≤ 5min，重大故障 RTO ≤ 30min，区域灾难 RTO ≤ 4h（冷备），与 8.4/8.6 口径一致。
- **计量与排除项**：不可用判定、部分降级、排除项（供应商故障/区域级 IaaS/客户侧网络/429 限流）按 8.1 定义执行，月度出可用率报告。

---

*本文档由当前代码库实态调研生成（backend/app、deploy/、site/content/docs、.github/workflows 均经核对）；「现状」章节描述均为代码事实，「目标态」章节为生产化设计方案，落地进度以 `site/content/docs/plans/` 为准。*
