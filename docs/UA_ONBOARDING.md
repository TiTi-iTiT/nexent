# Nexent 开发者入手指南

> 本文档基于 `.ua/knowledge-graph.json`（6026 节点 / 8202 边 / 7 层 / 13 步导览）生成，为进入代码前提供「全景地图」：架构分层、关键概念、导览路径、文件地图与复杂度热点。

---

## 一、项目概览

| 项 | 值 |
| --- | --- |
| 项目名 | **Nexent** |
| 描述 | Nexent 是 AI Agent 平台，包含 FastAPI 后端、Next.js 前端与 Agent SDK，提供 Agent 编排、记忆、调度、工具与技能等能力。 |
| 语言 | Python、TypeScript、JavaScript、CSS、YAML、Markdown |
| 框架 | FastAPI、Next.js、React、Vite、NextIntl |
| 图谱分析时间 | 2026-09-16 |
| 对应 Git 提交 | `473b65268ba8cda3db53be735f97678e214c583b` |

三大部分构成：
- **`backend/`** — FastAPI HTTP 后端（路由、服务、数据库访问）；
- **`frontend/`** — Next.js + React + NextIntl 前端 UI；
- **`sdk/nexent/`** — Python Agent 框架核心（执行、记忆、调度、向量库、工具与技能）。

---

## 二、架构分层

图谱将代码划分为 **7 个层**，共约 **1659 个文件级节点**：

| 层 | 名称 | 说明 | 文件级节点数 |
| --- | --- | --- | --- |
| `backend` | 后端 API 层 | FastAPI 后端整体：HTTP 路由端点（`backend/apps/*`）、业务服务（`backend/services/*`）、数据库访问（`backend/database/*`）及支撑模块 | 约 351 |
| `frontend` | 前端 UI 层 | Next.js/TypeScript 前端：页面路由（`frontend/app/*`）、UI 组件（resources、hooks、services、stores）与样式 | 约 591 |
| `sdk-core` | Agent 框架与 SDK 层 | `sdk/nexent` 下的 Python Agent 框架核心：核心执行、记忆、调度、向量库、工具与技能 | 约 242 |
| `benchmark` | 基准测试与评测层 | `sdk/benchmark` 下的基准测试、评测工具、上下文管理器对比及 Langfuse 集成 | 约 72 |
| `configuration` | 配置层 | 前端 tsconfig/package、后端模型目录与 prompts，以及根级 CI 配置 | 约 72 |
| `documentation` | 文档层 | 根 README、贡献指南及 SDK 各子包说明文档 | 约 69 |
| `infrastructure` | 基础设施与构建层 | 根级构建/启动/卸载脚本、容器编排与服务定义（compose.yaml、.dockerignore 等） | 约 292 |

---

## 三、关键概念

- **沙箱隔离（Sandbox Isolation）** — `sdk/nexent/core/agents/sandbox.py`：Agent 代码执行沙箱体系，容器化（Docker/WASM）与本地执行器，含工具桥接、shell 防护、MinIO 输出同步与内核租约管理。Agent 在受控、可回滚的环境中运行。

- **额度/配额模型（Quota Model）** — `backend/services/quota_service.py`：平台额度与容量管理核心，覆盖租户/个人/知识库多级的硬限额、软配额、容量统计、提醒及写前写后校验。

- **多租户（Multi-Tenancy）** — 用户/群组/租户贯穿全链路：`tenant_service.py`、`user_tenant_db.py`、`api_key_app.py`、`invitation_service.py` 等共同实现租户内隔离、角色与限额。

- **Agent 运行时（Agent Runtime）** — `sdk/nexent/core/agents/run_agent.py`：Agent 运行真正入口（运行参数、授权上下文与历史、MCP 配置）；`core_agent.py` 提供通用基类与工具编排（代码块解析、护栏、流式执行）。后端经 `backend/apps/agent_app.py` 暴露运行/停止/查询端点。

- **上下文管理（Context Management）** — SDK 最精密的子系统：`context/manager.py`（装配/压缩/长时记忆选择）、`runtime.py`（ManagedContextRuntime）、`context_item.py`。采用「类型 + handler + policy」注册表模式，把系统提示词、工具、技能、记忆、知识库统一为上下文条目分级处理。

- **记忆系统（Memory System）** — `sdk/nexent/memory/`：`service.py`（读写与生命周期）、`retrieval/pipeline.py`（拆分、范式化、评分融合、时间衰减、MMR 去重、token 预算）；`backend/services/memory_*_service.py` 实现记忆索引、梦境整合、外部 Provider 与插件加载。

- **工具与技能库（Tools & Skills）** — `sdk/nexent/core/tools/` 大量内建工具（沙箱、搜索记忆、知识库检索、终端、SQL、多厂商检索）；技能由 `backend/apps/skill_app.py` 与 `skill_repository_service.py` 管理。「会思考的 Agent」与「能执行的工具」解耦是框架可扩展性的关键。

- **多厂商模型网关（Model Gateway）** — `sdk/nexent/core/gateway/modality/*` 提供 embedding / rerank / VLM / 语音（STT/TTS）等多类型、多厂商适配器；`core/models/*` 封装 OpenAI 兼容、Jina、Cohere、阿里、火山等厂商。新增模型只需改配置数据（`model_catalog.json`），而非枚举/硬编码。

- **A2A 与 MCP** — `a2a_agent_proxy.py`、`a2a_server_service.py`、`a2a_client_service.py` 支持跨 Agent 通信；多个 MCP 服务（`mcp_management_service.py`、`remote_mcp_service.py`、`managed_mcp.py`）用独立事件循环在独立线程中托管 MCP 会话。

- **数据管道（Data Processing）** — `backend/data_process/` 与 `sdk/nexent/data_process/` 基于 Ray/Celery 处理分片、ES 写入、文档拆分与图片提取，是知识库取数的基础。

- **评测与可观测（Evaluation & Observability）** — `backend/services/agent_evaluation_service.py`（约 2395 行）做多评估器（代码/LLM）评测；`sdk/benchmark/` 提供基准；`monitor/monitoring.py` 基于 OpenTelemetry/OpenInference，`langfuse/compose.yaml` 自托管追踪栈，形成「评测 + 可观测」闭环。

- **人机交互（Human Interaction）** — `human_interaction/` 运行时提供 Ask-User 工具、澄清、审批、租约恢复与超时，支持步骤级人工介入。

- **自动化编排（Agent Automation）** — `agent_automation/`（facade、intent_analyzer、capability_resolver）把用户自然语言意向解析为 cron 调度任务与能力绑定。

- **NL2Agent** — `nl2agent_service.py` 把自然语言需求转化为 Agent/技能配置，是「低代码生成 Agent」的能力。

---

## 四、导览路径（13 步）

1. **项目概览与工程规则** — `README.md`（项目是什么）、`AGENTS.md`（工程规则）、`CONTRIBUTING.md`（GitFlow 贡献流程）。
2. **前端服务端入口** — `frontend/server.js`（认证转发、Cookie、配置 API、附件代理、反向代理）。
3. **前端布局与国际化壳层** — `app/[locale]/layout.tsx` + `layout.client.tsx`，基于 `[locale]` 动态段的国际化路由设计。
4. **后端 API 层入口** — 后端「一文件一应用」：`backend/apps/agent_app.py` 与 `backend/data_process/app.py`。
5. **Agent 框架核心与运行时** — `sdk/nexent/core/__init__.py`、`agents/nexent_agent.py`、`agents/run_agent.py`。
6. **上下文管理与压缩** — `context/manager.py`、`runtime.py`、`context_item.py`。
7. **记忆系统** — `sdk/nexent/memory/__init__.py`、`service.py`、`retrieval/pipeline.py`。
8. **工具与技能库** — `sdk/nexent/core/tools/__init__.py`、`backend/apps/skill_app.py`、`skill/service.py`。
9. **前端核心功能页面** — `app/[locale]/chat/page.tsx` 与 `agents/page.tsx`。
10. **模型目录与配置层** — `config/model_catalog.json` + `model_catalog_loader.py`。
11. **构建、部署与容器化** — `build.sh`、`deploy.sh`、`.dockerignore`。
12. **基准测试与评测层** — `sdk/benchmark/README.md` 与 `eventqa_eval/run_eventqa.py`。
13. **可观测性编排** — `langfuse/compose.yaml` 自托管 Langfuse 追踪栈。

> 建议：第 1 步 → 第 4 步（前端 → 后端），扎入第 5–8 步的 SDK 内核，再回到 9–10 页面前端与配置，最后 11–13 看交付与观测，形成完整闭环。

---

## 五、文件地图（按层）

### 后端 API 层（`backend/`）
| 文件 | 作用 |
| --- | --- |
| `backend/apps/agent_app.py` | Agent 核心 HTTP 接口：运行/停止/信息检索等端点 |
| `backend/apps/*_app.py`（约 60 个） | 一文件一应用的 FastAPI 路由模块（quota、skill、tenant、mcp、memory、evaluation、conversation 等） |
| `backend/apps/app_factory.py` | 应用工厂，`include_router` 汇总各路由模块 |
| `backend/services/quota_service.py` | 多级额度/容量管理核心 |
| `backend/services/conversation_management_service.py` | 对话持久化、历史上下文、流式消息 |
| `backend/services/agent_evaluation_service.py` | Agent 评测核心（约 2395 行） |
| `backend/services/agent_automation/` | 自动化 facade、意图解析、能力绑定 |
| `backend/services/human_interaction/` | 人机交互应用层与运行时端口 |
| `backend/services/nl2agent_service.py` | 自然语言生成 Agent |
| `backend/database/db_models.py` | 全部 ORM 表模型中心 |
| `backend/database/client.py` | PostgreSQL + MinIO 单例客户端 |
| `backend/data_process/` `.py` `ray_actors.py` `ray_config.py` | Ray/Celery 数据处理 |
| `backend/consts/const.py` | 集中式环境变量读取与应用版本（`APP_VERSION`） |
| `backend/agents/create_agent_info.py` | Agent 实例化核心装配逻辑 |

### 前端 UI 层（`frontend/`）
| 文件 | 作用 |
| --- | --- |
| `frontend/server.js` | 自定义 Node 服务端：认证转发、Cookie、配置 API、附件代理、反向代理 |
| `frontend/app/[locale]/layout.tsx` / `layout.client.tsx` | 本地化根布局 + 客户端壳层 |
| `frontend/app/[locale]/chat/page.tsx` | 聊天主界面 |
| `frontend/app/[locale]/agents/page.tsx` | Agent 设置编排页（多面板配置） |
| `frontend/services/*` | 后端 API 封装服务（tenant、upload、quota、skill、model、memory、tag 等） |
| `frontend/stores/*` | 前端状态管理（agentConfigStore、agentStore） |
| `frontend/types/*` | 类型定义（agentConfig、chat、modelConfig、quota、mcpTools） |

### Agent 框架与 SDK 层（`sdk/nexent/`）
| 文件 | 作用 |
| --- | --- |
| `core/agents/run_agent.py` | Agent 运行入口（运行参数/授权/历史/MCP） |
| `core/agents/nexent_agent.py` | 面向平台用户的 Agent（本地/LangChain/MCP/内建工具） |
| `core/agents/core_agent.py` | 通用 Agent 基类与工具编排 |
| `core/agents/sandbox.py` | 代码执行沙箱（Docker/WASM/本地） |
| `core/agents/context/manager.py`、`runtime.py`、`context_item.py` | 上下文管理、压缩、生命周期 |
| `memory/service.py`、`retrieval/pipeline.py` | 记忆读写与检索流水线 |
| `core/tools/` | 内建工具（沙箱、终端、SQL、知识库、RAGFlow、Dify、DataMate 等） |
| `core/models/`、`core/gateway/modality/*` | 多厂商模型封装与适配器 |
| `vector_database/`（elasticsearch_core、datamate_core、base） | 向量库抽象与多后端实现 |
| `monitor/monitoring.py` | OpenTelemetry/OpenInference 追踪与 LLM 监控 |
| `skills/skill_manager.py` | 技能管理门面（单例） |
| `container/`（docker_client、k8s_client） | 容器生命周期管理 |

### 基准测试与评测层（`sdk/benchmark/`）
- `eventqa_eval/run_eventqa.py` — EventQA 评测任务运行脚本；`acon_eval/`、`infra/langfuse/compose.yaml` — 评测与追踪支撑。

### 配置层（`configuration`）
- `backend/configs/model_catalog.json` + `model_catalog_loader.py` — 模型目录（新增模型的唯二改动点）；`frontend/package.json`、`tsconfig`；`backend/consts/prompts/*` 提示词模板（中英双语）；根级 CI 配置（`.sonarcloud.properties`、`codecov.yml`）。

### 文档层（`documentation`）
- `README.md`（中/英）、`AGENTS.md`（工程规则）、`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md`、`SECURITY.md` 及 SDK 各子包说明。

### 基础设施与构建层（`infrastructure`）
- `build.sh` / `deploy.sh` / `uninstall.sh` / `start_backend.sh` — 构建、部署、卸载、启动脚本（薄入口，转发到 `deploy/`）；`deploy/`、`docker/`、`k8s/`；容器编排与服务定义；`.dockerignore`。

---

## 六、复杂度热点（新开发请谨慎）

文件级节点共 **1659 个**：**499 `complex`** / 621 `moderate` / 539 `simple`。按层 complex 分布：

| 层 | complex 数 |
| --- | --- |
| frontend | 180 |
| backend | 146 |
| sdk-core | 67 |
| infrastructure | 50 |
| configuration | 23 |
| benchmark | 23 |
| documentation | 10 |

**最值得谨慎触碰的高复杂度区域：**

**SDK 内核（sdk-core，67 complex）**
- 上下文生命周期：`context/runtime.py`、`manager.py`、`projector.py`、`history_compression.py`、`llm_summary.py`、`capacity_budget.py`
- Agent 执行：`run_agent.py`、`nexent_agent.py`、`core_agent.py`、`sandbox.py`、`managed_mcp.py`、`verification.py`
- 记忆与向量：`memory/service.py`、`memory/retrieval/pipeline.py`、`vector_database/elasticsearch_core.py`
- 并发与人机交互：`core/concurrency/manager.py`、`human_interaction/runtime.py`、`live_runtime.py`

**后端核心（backend，146 complex）**
- 评测与自动化：`services/agent_evaluation_service.py`、`services/agent_automation/facade.py`、`intent_parser.py`、`intent_analyzer.py`
- 会话与额度：`services/conversation_management_service.py`、`services/quota_service.py`
- 多租户与身份：`services/tenant_service.py`、`user_management_service.py`、`database/user_tenant_db.py`、`database/token_db.py`
- 数据模型与记忆栈：`database/db_models.py`、`database/conversation_db.py`、`services/memory_*_service.py`、`memory_provider_plugins/*`
- A2A 与 MCP：`services/a2a_server_service.py`、`a2a_client_service.py`、`a2a_agent_adapter.py`、`remote_mcp_service.py`、`mcp_container_service.py`
- 数据处理：`data_process/tasks.py`、`data_process_service.py`、`services/data_process_service.py`

**前端（frontend，180 complex）** — 集中在 `services/*`、`stores/*`、`types/*` 与 `server.js`，改后端契约时要同步关注。

**提示**：改动上述文件前，请先加载仓库对应技能规则（见 `AGENTS.md` 的 Load rules）。特别注意：`deploy/sql/` 下已合并进目标分支的 SQL 文件不可编辑，迁移须放 `deploy/sql/migrations/`；环境变量读取必须集中在 `backend/consts/const.py`。

---

## 图谱范围小结

- **节点**：约 6026 个（含 7 个 layer 节点；内容节点约 6019，其中文件级节点 1659）
- **边**：约 8202 条
- **层**：7 层
- **导览步骤**：13 步
- **文件级复杂度**：complex 499 / moderate 621 / simple 539