# TradingAgents-CN 项目详细分析报告

> 分析日期：2026-04-08  
> 项目版本：v1.0.0-preview  
> 源项目：[TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents)

---

## 目录

1. [项目概述](#1-项目概述)
2. [项目架构](#2-项目架构)
3. [代码目录层级](#3-代码目录层级)
4. [业务数据流](#4-业务数据流)
5. [核心代码逻辑](#5-核心代码逻辑)

---

## 1. 项目概述

### 1.1 项目名称与主要功能

**TradingAgents-CN**（中文增强版）是一个面向中文用户的**多智能体 AI 股票分析学习与研究平台**。基于原版 [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents) 深度改造，提供完整的 A 股、港股、美股分析能力。

**核心功能：**

- **多智能体协作分析**：模拟真实投行团队，由市场分析师、新闻分析师、基本面分析师、社交媒体分析师等多个 AI 智能体协作完成股票分析
- **多空辩论机制**：看涨研究员与看跌研究员进行结构化辩论，研究经理综合裁定
- **风险评估系统**：激进、保守、中性三位风险分析师讨论后由风险经理最终决策
- **多市场支持**：完整支持 A 股（沪深两市）、港股、美股三个市场
- **多 LLM 供应商**：支持 OpenAI、Anthropic、Google Gemini、DeepSeek、阿里百炼/DashScope、智谱 AI、SiliconFlow、OpenRouter、Ollama 等
- **混合模型模式**：快速思考模型（分析师、研究员）与深度思考模型（研究经理、风险经理）可来自不同 LLM 供应商
- **Web 管理界面**：Vue 3 前端 + FastAPI 后端，提供配置管理、批量分析、报告导出、模拟交易等企业级功能
- **命令行界面（CLI）**：基于 Typer + Rich 的交互式终端分析工具

### 1.2 技术栈

| 层级 | 技术选型 |
|------|----------|
| **后端框架** | FastAPI + Uvicorn + Pydantic v2 |
| **前端框架** | Vue 3 + Element Plus + Vite + TypeScript |
| **AI/LLM 框架** | LangChain + LangGraph |
| **数据库** | MongoDB（业务数据，motor 异步驱动 + pymongo 同步驱动） |
| **缓存** | Redis（会话/缓存）+ 文件缓存 + MongoDB 缓存（多级缓存） |
| **向量数据库** | ChromaDB（金融记忆系统） |
| **任务调度** | APScheduler（AsyncIOScheduler） |
| **实时通信** | SSE（Server-Sent Events）+ WebSocket 双通道 |
| **金融数据源** | Tushare、AKShare、BaoStock（A 股）；yfinance、Finnhub（美股/港股） |
| **CLI 框架** | Typer + Rich |
| **容器化** | Docker + docker-compose（支持 amd64 + arm64 多架构） |
| **编程语言** | Python 3.10+（后端）；TypeScript（前端） |

### 1.3 运行环境与依赖

**必要运行环境：**

| 组件 | 用途 | 备注 |
|------|------|------|
| Python 3.10+ | 后端运行时 | 推荐 3.11+ |
| MongoDB | 业务数据存储 | FastAPI 后端必需 |
| Redis | 缓存和会话管理 | FastAPI 后端必需 |
| Node.js + yarn | 前端构建 | 仅前端开发需要 |

**CLI 模式（最小依赖）：** 仅需 Python 3.10+ 和 LLM API Key，无需 MongoDB/Redis。CLI 直接调用 `tradingagents` 核心库。

**关键 Python 依赖：**

- `langchain-openai`、`langchain-anthropic`、`langchain-google-genai`、`langgraph` — AI 智能体编排
- `akshare`、`tushare`、`baostock`、`yfinance`、`finnhub-python` — 金融数据
- `fastapi`、`uvicorn`、`motor`、`redis` — Web 后端
- `chromadb` — 向量记忆存储
- `apscheduler` — 定时任务
- `dashscope` — 阿里百炼 SDK

**环境变量配置（`.env` 文件）：** 包含 LLM API Key（`OPENAI_API_KEY`、`DASHSCOPE_API_KEY`、`DEEPSEEK_API_KEY` 等）、数据库连接信息、数据源 Token（`TUSHARE_TOKEN`）、代理配置等。

---

## 2. 项目架构

### 2.1 整体架构

本项目采用**分层架构**，分为四个主要层次：

```
┌─────────────────────────────────────────────────────────┐
│                    表现层 (Presentation)                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│   │  Vue 3 前端  │  │  CLI 终端   │  │  REST API   │    │
│   │ (Element+)  │  │ (Typer+Rich)│  │  (FastAPI)   │    │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
├──────────┼─────────────────┼────────────────┼───────────┤
│          │          服务层 (Service)        │            │
│          │   ┌──────────────────────────┐   │            │
│          │   │  SimpleAnalysisService   │   │            │
│          │   │  ConfigService           │   │            │
│          │   │  ScreeningService        │   │            │
│          │   │  QueueService            │   │            │
│          │   └────────────┬─────────────┘   │            │
├───────────────────────────┼─────────────────────────────┤
│                    核心层 (Core Engine)                   │
│   ┌───────────────────────┴─────────────────────────┐   │
│   │            TradingAgentsGraph (LangGraph)         │   │
│   │  ┌─────────┐ ┌──────────┐ ┌──────────┐          │   │
│   │  │ 分析师团队│ │ 研究团队  │ │ 风险团队  │          │   │
│   │  │(4个分析师)│ │(多空辩论) │ │(3种风格)  │          │   │
│   │  └────┬────┘ └────┬─────┘ └────┬─────┘          │   │
│   │       │   Toolkit (统一工具层)    │                │   │
│   │  ┌────┴───────────┴────────────┴────┐            │   │
│   │  │    LLM Adapters (模型适配器层)     │            │   │
│   │  └──────────────────────────────────┘            │   │
│   └─────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│                    数据层 (Data)                          │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│   │  MongoDB  │  │  Redis   │  │ ChromaDB │              │
│   │ (业务数据) │  │ (缓存)   │  │ (记忆)   │              │
│   └──────────┘  └──────────┘  └──────────┘              │
│   ┌──────────────────────────────────────┐              │
│   │  外部数据源 (Tushare/AKShare/yfinance等)│            │
│   └──────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

### 2.2 各模块职责与交互方式

#### 核心库 `tradingagents/`

| 模块 | 职责 | 交互方式 |
|------|------|----------|
| `graph/` | LangGraph 工作流编排，状态管理，条件路由 | 被 `app/services` 和 `cli/` 直接调用 |
| `agents/` | 各角色智能体实现（分析师、研究员、交易员、风险分析师） | 由 `graph/setup.py` 注册为 LangGraph 节点 |
| `dataflows/` | 数据获取管道，数据源抽象，缓存管理 | 被 `agents/` 通过 `interface.py` 调用 |
| `llm_adapters/` | LLM 供应商适配器 | 被 `graph/trading_graph.py` 实例化 |
| `config/` | 配置管理（`ConfigManager` 单例，`settings.json`） | 全局共享，`interface.py` 和 `agents/` 读取 |
| `tools/` | 分析工具集 | 被 `agents/` 调用 |
| `utils/` | 日志管理、股票代码识别等公用工具 | 全项目引用 |

#### Web 后端 `app/`

| 模块 | 职责 |
|------|------|
| `main.py` | FastAPI 应用入口，生命周期管理，中间件注册，定时任务配置 |
| `core/` | 基础设施：Pydantic Settings、MongoDB/Redis 连接池、配置桥接（`config_bridge.py`） |
| `routers/` | 35+ 个 API 路由模块（分析、筛选、队列、收藏、配置、报告等） |
| `services/` | 业务逻辑层：分析执行、配置管理、数据源同步、筛选、队列等 |
| `worker/` | 后台同步工作器（Tushare/AKShare/BaoStock 数据同步服务） |
| `models/` | Pydantic 请求/响应模型 |
| `middleware/` | CORS、请求 ID、操作日志、限流 |

#### 前端 `frontend/`

| 模块 | 职责 |
|------|------|
| `views/` | 页面组件：分析、筛选、收藏、报告、设置、模拟交易等 |
| `api/` | Axios API 封装 |
| `stores/` | Pinia 状态管理 |
| `router/` | Vue Router 路由 |
| `components/` | 复用组件（Dashboard、Sync、Layout 等） |

#### CLI `cli/`

| 模块 | 职责 |
|------|------|
| `main.py` | Typer CLI 入口，交互式 LLM/分析师选择，Rich 实时进度显示 |
| `models.py` | 分析师类型枚举和模型选择辅助 |
| `utils.py` | CLI 工具函数（供应商选择、模型选择等） |

### 2.3 关键设计模式与架构决策

#### (1) 双 LLM 混合模式

系统区分**快速思考模型**（`quick_thinking_llm`）和**深度思考模型**（`deep_thinking_llm`）：

- **快速模型**：驱动分析师（市场、新闻、基本面、社交媒体）和研究员（看涨、看跌），追求速度和成本效率
- **深度模型**：驱动研究经理（`Research Manager`）和风险经理（`Risk Judge`），追求判断质量

两个模型可来自不同供应商（如快速用 Qwen，深度用 DeepSeek），通过 `quick_provider` 和 `deep_provider` 配置区分。

#### (2) 配置桥接模式

核心库 `tradingagents/` 通过 `os.getenv()` 读取 API Key，但 FastAPI 后端将 API Key 存储在 MongoDB 中。`app/core/config_bridge.py` 在启动时将 MongoDB 中的配置写入环境变量，实现无侵入式桥接：

```
MongoDB system_configs → config_bridge.py → os.environ → tradingagents 核心库
```

#### (3) 死循环断路器

部分 LLM 会无限循环调用工具。系统在 `AgentState` 中维护每类分析师的工具调用计数器（`market_tool_call_count` 等），`ConditionalLogic` 中检查计数器，超过阈值（基本面 1 次，其他 3 次）强制结束该分析师节点。

#### (4) 异步/线程隔离模式

`TradingAgentsGraph.propagate()` 是同步阻塞调用，而 FastAPI 运行在异步事件循环中。解决方案分三层：

1. **路由层**：`BackgroundTasks.add_task()` 异步返回 HTTP 200
2. **服务层**：`ThreadPoolExecutor` + `loop.run_in_executor()` 在线程池中执行同步代码
3. **线程内异步桥接**：线程内需要异步操作时，创建 `asyncio.new_event_loop()` 执行后关闭

#### (5) 统一工具层（Toolkit）

`Toolkit` 类使用类级别 `_config` 字典（所有实例共享），提供 4 个"统一"工具方法，自动根据股票代码识别市场类型（A 股/港股/美股），路由到对应的数据提供者：

```python
get_stock_market_data_unified()      # 行情与技术指标
get_stock_fundamentals_unified()     # 基本面分析
get_stock_news_unified()             # 新闻数据
get_stock_sentiment_unified()        # 社交媒体情绪
```

#### (6) 消息清理节点

每个分析师完成后，`create_msg_delete()` 节点删除所有消息并添加 `HumanMessage("Continue")` 占位符。目的有二：
- 防止前一个分析师的上下文泄露到下一个分析师
- 满足 Anthropic 模型要求消息列表非空的约束

#### (7) 数据源级联降级

A 股数据源支持多级降级：Tushare → AKShare → BaoStock。优先级来源于 MongoDB 配置，可在运行时通过 Web UI 调整。港股/美股的数据源优先级在每次数据请求时从 MongoDB 实时读取（非缓存），支持运行时变更。

---

## 3. 代码目录层级

```
TradingAgents-CN/
├── tradingagents/                  # 核心多智能体库（Apache 2.0）
│   ├── graph/                      # LangGraph 工作流引擎
│   │   ├── trading_graph.py        # ★ 主入口：TradingAgentsGraph 类
│   │   ├── setup.py                # StateGraph 构建：节点注册与边连接
│   │   ├── agent_states.py         # AgentState/InvestDebateState/RiskDebateState 类型定义
│   │   ├── conditional_logic.py    # 条件路由逻辑（工具循环控制、辩论轮次）
│   │   ├── propagation.py          # 初始状态创建与 stream 参数配置
│   │   ├── reflection.py           # 分析后反思：将结果写入 ChromaDB 记忆
│   │   └── signal_processing.py    # LLM 输出解析为结构化决策
│   ├── agents/                     # 智能体实现
│   │   ├── analysts/               # 分析师团队
│   │   │   ├── market_analyst.py           # 市场分析师（技术指标）
│   │   │   ├── fundamentals_analyst.py     # 基本面分析师（财务数据）
│   │   │   ├── news_analyst.py             # 新闻分析师
│   │   │   ├── social_media_analyst.py     # 社交媒体分析师
│   │   │   └── china_market_analyst.py     # 中国市场特化分析师
│   │   ├── researchers/            # 研究团队
│   │   │   ├── bull_researcher.py  # 看涨研究员
│   │   │   └── bear_researcher.py  # 看跌研究员
│   │   ├── managers/               # 管理团队
│   │   │   ├── research_manager.py # 研究经理（使用深度模型）
│   │   │   └── risk_manager.py     # 风险经理（使用深度模型）
│   │   ├── risk_mgmt/              # 风险评估团队
│   │   │   ├── risky_debator.py    # 激进风险分析师
│   │   │   ├── safe_debator.py     # 保守风险分析师
│   │   │   └── neutral_debator.py  # 中性风险分析师
│   │   ├── trader/                 # 交易员
│   │   │   └── trader.py           # 交易计划生成
│   │   └── utils/                  # 智能体工具
│   │       ├── agent_utils.py      # ★ Toolkit 类（统一工具注册）+ create_msg_delete()
│   │       ├── agent_states.py     # 状态类型定义
│   │       └── memory.py           # FinancialSituationMemory（ChromaDB 记忆）
│   ├── dataflows/                  # 数据流管道
│   │   ├── interface.py            # ★ 数据层公共 API（agents 调用此文件）
│   │   ├── optimized_china_data.py # OptimizedChinaDataProvider（A 股优化数据提供者）
│   │   ├── data_source_manager.py  # A 股数据源优先级管理与降级
│   │   ├── providers/              # 具体数据提供者
│   │   │   ├── base_provider.py    # BaseStockDataProvider（ABC 基类）
│   │   │   ├── china/              # A 股数据源
│   │   │   │   ├── tushare.py      # TushareProvider
│   │   │   │   ├── akshare.py      # AKShareProvider
│   │   │   │   └── baostock.py     # BaostockProvider
│   │   │   ├── hk/                 # 港股数据源
│   │   │   │   ├── hk_stock.py     # yfinance 港股
│   │   │   │   └── improved_hk.py  # AKShare 港股
│   │   │   └── us/                 # 美股数据源
│   │   │       ├── yfinance.py     # Yahoo Finance
│   │   │       ├── finnhub.py      # Finnhub API
│   │   │       └── optimized.py    # 缓存封装
│   │   ├── cache/                  # 多级缓存系统
│   │   │   ├── file_cache.py       # 文件缓存
│   │   │   ├── db_cache.py         # MongoDB 缓存
│   │   │   ├── adaptive.py         # 自适应缓存策略
│   │   │   └── integrated.py       # 集成缓存管理器
│   │   ├── news/                   # 新闻数据
│   │   │   ├── google_news.py      # Google 新闻
│   │   │   ├── chinese_finance.py  # 中文财经平台
│   │   │   └── reddit.py           # Reddit 新闻
│   │   └── technical/              # 技术指标计算
│   ├── llm_adapters/               # LLM 供应商适配器
│   │   ├── dashscope_openai_adapter.py  # 阿里百炼（OpenAI 兼容）
│   │   ├── deepseek_adapter.py          # DeepSeek
│   │   ├── google_openai_adapter.py     # Google Gemini
│   │   └── openai_compatible_base.py    # 通用 OpenAI 兼容适配器
│   ├── config/                     # 配置管理
│   │   ├── config_manager.py       # ConfigManager 单例
│   │   ├── default_config.py       # 默认配置
│   │   ├── providers_config.py     # LLM 供应商注册表
│   │   └── runtime_settings.py     # 运行时可变设置
│   ├── models/                     # 数据模型
│   ├── utils/                      # 公用工具
│   │   ├── logging_manager.py      # 日志管理器
│   │   ├── logging_init.py         # 日志初始化
│   │   ├── stock_utils.py          # StockUtils（股票代码/市场识别）
│   │   └── tool_logging.py         # 工具调用日志装饰器
│   └── default_config.py           # DEFAULT_CONFIG 基线配置字典
│
├── app/                            # FastAPI 后端（专有许可）
│   ├── main.py                     # ★ 应用入口：lifespan、中间件、路由注册、定时任务
│   ├── core/                       # 基础设施
│   │   ├── config.py               # Pydantic Settings（环境变量映射）
│   │   ├── database.py             # MongoDB + Redis 连接池
│   │   ├── config_bridge.py        # ★ 配置桥接（MongoDB → 环境变量）
│   │   ├── unified_config.py       # 运行时配置访问
│   │   └── startup_validator.py    # 启动配置校验
│   ├── routers/                    # 35+ API 路由
│   │   ├── analysis.py             # 股票分析 API
│   │   ├── auth_db.py              # 认证鉴权
│   │   ├── screening.py            # 智能筛选
│   │   ├── queue.py                # 批量分析队列
│   │   ├── favorites.py            # 自选股管理
│   │   ├── config.py               # 系统配置
│   │   ├── reports.py              # 报告导出
│   │   ├── notifications.py        # SSE 通知
│   │   ├── websocket_notifications.py  # WebSocket 通知
│   │   ├── scheduler.py            # 定时任务管理
│   │   ├── stocks.py               # 股票数据
│   │   ├── paper.py                # 模拟交易
│   │   └── ...                     # 其他路由
│   ├── services/                   # 业务逻辑
│   │   ├── analysis/               # 分析执行服务
│   │   ├── config_service.py       # 配置管理服务
│   │   ├── screening/              # 筛选服务
│   │   ├── queue/                  # 队列服务
│   │   ├── progress/               # 进度追踪
│   │   ├── data_sources/           # 数据源管理
│   │   └── basics_sync/            # 基础数据同步
│   ├── worker/                     # 后台工作器
│   │   ├── tushare_sync_service.py # Tushare 数据同步
│   │   ├── akshare_sync_service.py # AKShare 数据同步
│   │   └── baostock_sync_service.py# BaoStock 数据同步
│   ├── models/                     # Pydantic 模型
│   ├── middleware/                 # 中间件
│   └── utils/                      # 工具函数
│
├── frontend/                       # Vue 3 前端（专有许可）
│   └── src/
│       ├── views/                  # 页面
│       │   ├── Analysis/           # 分析页面
│       │   ├── Screening/          # 筛选页面
│       │   ├── Favorites/          # 自选股
│       │   ├── Reports/            # 报告
│       │   ├── Settings/           # 系统设置
│       │   ├── PaperTrading/       # 模拟交易
│       │   ├── Dashboard/          # 仪表盘
│       │   └── ...
│       ├── api/                    # API 封装
│       ├── stores/                 # Pinia 状态管理
│       ├── router/                 # Vue Router
│       └── components/             # 公用组件
│
├── cli/                            # 命令行界面
│   ├── main.py                     # ★ Typer CLI 入口
│   ├── models.py                   # 分析师类型枚举
│   └── utils.py                    # 交互式选择工具
│
├── tests/                          # 测试套件
│   ├── pytest.ini                  # pytest 配置（在 tests/ 目录内）
│   ├── conftest.py                 # 项目根路径注入
│   ├── unit/                       # 单元测试
│   ├── integration/                # 集成测试
│   ├── config/                     # 配置测试
│   ├── services/                   # 服务测试
│   └── dataflows/                  # 数据流测试
│
├── docker/                         # Docker 配置文件
├── nginx/                          # Nginx 反向代理配置
├── scripts/                        # 运维脚本
├── docs/                           # 文档
├── pyproject.toml                  # Python 项目配置
├── requirements.txt                # 依赖声明
├── docker-compose.yml              # Docker 编排
├── main.py                         # CLI 入口点（tradingagents 命令）
└── tradingAgent/                   # Python 虚拟环境（非源码，注意大写 A）
```

---

## 4. 业务数据流

### 4.1 主要业务流程：股票分析

用户通过 Web UI 或 CLI 发起分析请求，数据流经以下路径：

```
用户请求（Web UI / CLI）
    │
    ├─── Web UI 路径 ─────────────────────────────────────────────┐
    │    POST /api/analysis                                       │
    │        ↓                                                    │
    │    app/routers/analysis.py                                  │
    │        ↓                                                    │
    │    BackgroundTasks.add_task(run_analysis_task)               │
    │        ↓  （立即返回 HTTP 200，分析在后台执行）                 │
    │    SimpleAnalysisService.execute_analysis_background()       │
    │        ↓                                                    │
    │    ThreadPoolExecutor → _run_analysis_sync()  [线程池]       │
    │        ↓                                                    │
    │    create_analysis_config()  ← 从 MongoDB 读取 LLM 配置     │
    │        ↓                                                    │
    ├─── CLI 路径 ───────────────────────────────────────────┐    │
    │    cli/main.py: analyze 命令                            │    │
    │        ↓  （交互式选择 LLM/模型/分析师）                  │    │
    │    构建 config 字典                                      │    │
    │        ↓                                                │    │
    ├────── 共同路径 ──────────────────────────────────────────┘    │
    │                                                              │
    │    TradingAgentsGraph(config=config)                         │
    │        ├── 创建 quick_thinking_llm + deep_thinking_llm       │
    │        ├── 初始化 Toolkit（统一工具注册）                      │
    │        ├── 初始化 5 个 FinancialSituationMemory（ChromaDB）   │
    │        ├── 创建 4 个 ToolNode（market/social/news/fundamentals）│
    │        └── GraphSetup.setup_graph() → 编译 StateGraph         │
    │                                                              │
    │    graph.propagate(stock_code, date, progress_callback)      │
    │        ↓                                                     │
    │    graph.stream(init_agent_state)                             │
    │                                                              │
    │    ┌──────── LangGraph 工作流 ────────────────────────────┐  │
    │    │                                                      │  │
    │    │  [Market Analyst] ↔ [tools_market] → [Msg Clear]     │  │
    │    │       ↓                                              │  │
    │    │  [News Analyst] ↔ [tools_news] → [Msg Clear]         │  │
    │    │       ↓                                              │  │
    │    │  [Fundamentals Analyst] ↔ [tools_fund] → [Msg Clear] │  │
    │    │       ↓                                              │  │
    │    │  [Social Analyst] ↔ [tools_social] → [Msg Clear]     │  │
    │    │       ↓                                              │  │
    │    │  [Bull Researcher] ↔ [Bear Researcher]  (N 轮辩论)   │  │
    │    │       ↓                                              │  │
    │    │  [Research Manager]  ← deep_thinking_llm 综合裁定    │  │
    │    │       ↓                                              │  │
    │    │  [Trader]  生成交易计划                               │  │
    │    │       ↓                                              │  │
    │    │  [Risky] ↔ [Safe] ↔ [Neutral]  (N 轮风险讨论)        │  │
    │    │       ↓                                              │  │
    │    │  [Risk Judge]  ← deep_thinking_llm 最终决策          │  │
    │    │       ↓                                              │  │
    │    │  END → final_state + decision                        │  │
    │    └──────────────────────────────────────────────────────┘  │
    │                                                              │
    │    返回 (final_state, decision)                               │
    │        ↓                                                     │
    │    ├── Web UI：写入 MongoDB，通过 SSE/WebSocket 推送进度      │
    │    └── CLI：Rich 终端渲染结果                                 │
    └──────────────────────────────────────────────────────────────┘
```

### 4.2 关键数据实体与转换节点

#### AgentState（核心状态）

整个工作流共享的状态对象，在 LangGraph 节点间传递：

```python
class AgentState(MessagesState):
    company_of_interest: str          # 分析标的（股票代码）
    trade_date: str                   # 分析日期

    # 分析师报告（每个分析师写入对应字段）
    market_report: str                # 市场分析报告
    sentiment_report: str             # 社交媒体情绪报告
    news_report: str                  # 新闻分析报告
    fundamentals_report: str          # 基本面分析报告

    # 死循环断路器
    market_tool_call_count: int       # 各分析师工具调用计数
    news_tool_call_count: int
    sentiment_tool_call_count: int
    fundamentals_tool_call_count: int

    # 研究辩论状态
    investment_debate_state: InvestDebateState  # 多空辩论历史与裁定
    investment_plan: str              # 研究经理的投资建议

    # 交易与风险状态
    trader_investment_plan: str       # 交易员的交易计划
    risk_debate_state: RiskDebateState  # 风险讨论历史
    final_trade_decision: str         # 最终交易决策
```

#### 数据转换节点

| 节点 | 输入 | 处理 | 输出 |
|------|------|------|------|
| 分析师（4 个） | `company_of_interest` + `trade_date` | LLM 调用工具获取数据 → 生成分析报告 | 对应 `*_report` 字段 |
| 消息清理（4 个） | 全部 `messages` | 删除所有消息 + 添加占位符 | 清空的 `messages` |
| 看涨/看跌研究员 | 4 份报告 | 辩论式分析 | `investment_debate_state` |
| 研究经理 | 辩论历史 | 深度 LLM 综合判断 | `investment_plan` |
| 交易员 | `investment_plan` | 生成交易计划 | `trader_investment_plan` |
| 风险分析师（3 个） | 交易计划 | 多角度风险讨论 | `risk_debate_state` |
| 风险经理 | 风险讨论历史 | 深度 LLM 最终裁定 | `final_trade_decision` |

### 4.3 外部系统与 API 集成点

```
┌─────────────────── 外部 LLM API ───────────────────────┐
│  OpenAI API         ← ChatOpenAI                       │
│  Anthropic API      ← ChatAnthropic                    │
│  Google AI API      ← ChatGoogleOpenAI (REST 适配器)    │
│  DashScope API      ← ChatDashScopeOpenAI (OpenAI 兼容) │
│  DeepSeek API       ← ChatDeepSeek                     │
│  智谱 AI API        ← ChatZhipuOpenAI                  │
│  SiliconFlow/OpenRouter/Ollama ← ChatOpenAI (兼容模式)  │
└────────────────────────────────────────────────────────┘

┌─────────────── 外部金融数据源 ─────────────────────────┐
│                                                        │
│  A 股数据源（级联降级）:                                 │
│    Tushare Pro API → AKShare → BaoStock                │
│    优先级由 MongoDB system_configs.data_source_configs   │
│    控制，支持运行时调整                                   │
│                                                        │
│  港股数据源:                                             │
│    AKShare（东方财富港股）→ yfinance                     │
│    优先级从 MongoDB 实时读取                              │
│                                                        │
│  美股数据源:                                             │
│    yfinance → Finnhub API                              │
│                                                        │
│  新闻数据源:                                             │
│    东方财富（AKShare）→ Google News → Finnhub News       │
│    → Reddit → OpenAI Web Search                        │
│                                                        │
│  社交媒体:                                               │
│    Reddit（PRAW）→ 中文平台（开发中）                    │
│                                                        │
└────────────────────────────────────────────────────────┘

┌─────────────── 内部存储系统 ───────────────────────────┐
│  MongoDB                                               │
│    ├── analysis_tasks      # 分析任务及结果              │
│    ├── system_configs      # 系统配置（LLM/数据源）       │
│    ├── users               # 用户信息                   │
│    ├── favorites           # 自选股                     │
│    ├── stock_basics        # 股票基础信息缓存            │
│    ├── stock_quotes        # 行情数据                   │
│    ├── stock_historical    # 历史数据                   │
│    ├── stock_news          # 新闻数据缓存               │
│    └── operation_logs      # 操作日志                   │
│                                                        │
│  Redis                                                 │
│    ├── session:*           # 用户会话                   │
│    ├── cache:*             # 通用缓存                   │
│    └── notification:*      # 通知推送                   │
│                                                        │
│  ChromaDB (本地文件)                                    │
│    ├── bull_memory          # 看涨研究员记忆             │
│    ├── bear_memory          # 看跌研究员记忆             │
│    ├── trader_memory        # 交易员记忆                │
│    ├── invest_judge_memory  # 研究经理记忆              │
│    └── risk_manager_memory  # 风险经理记忆              │
└────────────────────────────────────────────────────────┘
```

### 4.4 数据同步机制

FastAPI 后端通过 APScheduler 定时同步金融数据：

| 数据源 | 同步类型 | 调度方式 |
|--------|----------|----------|
| Tushare | 基础信息/行情/历史/财务/状态检查（5 个 Job） | Cron 表达式，可通过 `TUSHARE_UNIFIED_ENABLED` 开关暂停 |
| AKShare | 同上 5 类 | Cron 表达式，可通过 `AKSHARE_UNIFIED_ENABLED` 开关暂停 |
| BaoStock | 基础信息/日 K 线/历史/状态检查（4 个 Job） | Cron 表达式，可通过 `BAOSTOCK_UNIFIED_ENABLED` 开关暂停 |
| 新闻 | 自选股新闻同步 | Cron 表达式，可通过 `NEWS_SYNC_ENABLED` 开关 |
| 港股/美股 | 按需获取 + 缓存 | 无定时任务，请求时实时获取并缓存 |

---

## 5. 核心代码逻辑

### 5.1 TradingAgentsGraph 类（核心引擎）

**文件位置：** `tradingagents/graph/trading_graph.py`

#### 概述

`TradingAgentsGraph` 是整个系统的编排中枢。它负责初始化 LLM、工具、记忆、图结构，并提供 `propagate()` 方法执行分析流程。

#### 构造函数 `__init__()` 逻辑

```python
def __init__(self, selected_analysts, debug, config):
    # 1. 配置初始化
    self.config = config or DEFAULT_CONFIG
    set_config(self.config)  # 同步配置到 dataflows 层

    # 2. 双 LLM 实例化（核心决策点）
    quick_provider = config.get("quick_provider")
    deep_provider = config.get("deep_provider")
    if quick_provider != deep_provider:
        # 混合模式：不同供应商的 LLM
        self.quick_thinking_llm = create_llm_by_provider(quick_provider, ...)
        self.deep_thinking_llm = create_llm_by_provider(deep_provider, ...)
    else:
        # 单一供应商模式：大 if/elif 链按 llm_provider 分支创建

    # 3. 工具层初始化
    self.toolkit = Toolkit(config=self.config)

    # 4. 记忆系统初始化（5 个 ChromaDB Collection）
    if config.get("memory_enabled", True):
        self.bull_memory = FinancialSituationMemory("bull_memory", config)
        # ... 其他 4 个角色的记忆

    # 5. 工具节点创建（每个分析师对应一个 ToolNode）
    self.tool_nodes = self._create_tool_nodes()

    # 6. 编排组件
    self.conditional_logic = ConditionalLogic(max_debate_rounds, max_risk_rounds)
    self.graph_setup = GraphSetup(quick_llm, deep_llm, toolkit, tool_nodes, ...)
    self.graph = self.graph_setup.setup_graph(selected_analysts)
```

#### `propagate()` 方法 — 执行分析

```
输入：company_name (str)、trade_date (str)、progress_callback (可选)、task_id (可选)
输出：(final_state: dict, decision: dict)
```

**处理流程：**

1. 创建初始状态 `init_agent_state`，设置 `company_of_interest` 和 `trade_date`
2. 通过 `graph.stream(init_agent_state)` 流式执行图
3. 逐节点处理 chunk，记录每个节点的执行耗时（`node_timings` 字典）
4. 如果有 `progress_callback`：
   - 使用 `updates` 模式获取节点级别进度
   - 通过 `_send_progress_update()` 映射节点名到中文进度消息（如 `"Market Analyst"` → `"📊 市场分析师"`）
   - 累积状态更新到 `final_state`
5. 分析完成后：
   - 构建性能数据（按团队分类统计耗时和占比）
   - 调用 `process_signal()` 将 LLM 输出解析为结构化决策（包含操作建议、置信度等）
   - 记录状态日志到 `eval_results/{ticker}/` 目录
6. 返回 `(final_state, decision)`

**关键特性：**
- 始终使用 `stream()` 模式（即使非 debug），以支持节点级别计时
- `_send_progress_update()` 通过节点名称映射表过滤掉工具节点和消息清理节点，只推送有意义的进度
- 性能数据按团队分类（分析师/工具/消息清理/研究/交易/风险），计算每个团队的时间占比

### 5.2 GraphSetup.setup_graph() — 工作流拓扑构建

**文件位置：** `tradingagents/graph/setup.py`

```
输入：selected_analysts (list) — 选中的分析师类型列表，如 ["market", "news", "fundamentals", "social"]
输出：compiled StateGraph — 编译后的 LangGraph 工作流
```

**处理逻辑：**

```python
def setup_graph(self, selected_analysts):
    workflow = StateGraph(AgentState)

    # 1. 动态创建分析师节点（仅选中的分析师）
    for analyst_type in selected_analysts:
        # 每个分析师需要 3 个节点：
        workflow.add_node(f"{Type} Analyst", create_xxx_analyst(quick_llm, toolkit))
        workflow.add_node(f"tools_{type}", ToolNode([...]))
        workflow.add_node(f"Msg Clear {Type}", create_msg_delete())

    # 2. 添加固定节点
    workflow.add_node("Bull Researcher", ...)   # 使用 quick_llm + 记忆
    workflow.add_node("Bear Researcher", ...)   # 使用 quick_llm + 记忆
    workflow.add_node("Research Manager", ...)  # ★ 使用 deep_llm + 记忆
    workflow.add_node("Trader", ...)
    workflow.add_node("Risky/Safe/Neutral Analyst", ...)  # 使用 quick_llm
    workflow.add_node("Risk Judge", ...)        # ★ 使用 deep_llm + 记忆

    # 3. 连接边（分析师按序链接）
    START → 第一个分析师
    分析师 → 条件边（should_continue_{type}）→ 工具节点 或 消息清理
    工具节点 → 回到同一分析师（循环直到无 tool_calls 或超过上限）
    消息清理 → 下一个分析师（或最后一个分析师 → Bull Researcher）

    # 4. 辩论/讨论的条件边
    Bull ↔ Bear（由 should_continue_debate 控制轮数）→ Research Manager
    Risky → Safe → Neutral → Risky（由 should_continue_risk_analysis 控制）→ Risk Judge

    # 5. 固定边
    Research Manager → Trader → Risky Analyst
    Risk Judge → END

    return workflow.compile()
```

**核心设计要点：**
- 分析师是**可选的**——用户可以只选择 1~4 个分析师
- 分析师之间是**顺序执行**（非并行），通过消息清理隔离上下文
- 辩论轮次由配置参数控制（`max_debate_rounds`、`max_risk_discuss_rounds`）
- 条件边使用 `ConditionalLogic` 类方法，动态决定图的走向

### 5.3 ConditionalLogic — 工作流路由控制

**文件位置：** `tradingagents/graph/conditional_logic.py`

#### 分析师工具循环控制（以 `should_continue_fundamentals` 为例）

```
输入：state: AgentState
输出："tools_fundamentals"（继续工具调用）或 "Msg Clear Fundamentals"（结束分析）
```

**判定优先级：**

```
1. 报告已完成检查：如果 fundamentals_report 长度 > 100 字符 → 结束
2. 工具调用请求检查：如果最后消息有 tool_calls 属性：
   2a. 工具调用次数 >= 上限（基本面=1，其他=3）→ 强制结束
   2b. 未超上限 → 继续调用工具
3. 无工具调用 → 正常结束
```

**死循环修复机制：** 每个分析师对应 `AgentState` 中的一个计数器字段。`ToolNode` 执行后状态更新递增计数器。`ConditionalLogic` 检查计数器，超过阈值即使 LLM 仍然请求工具调用也会强制终止。基本面分析师上限为 1 次（因为统一工具一次即可获取全部数据），其他分析师上限为 3 次。

#### 辩论控制（`should_continue_debate`）

```python
def should_continue_debate(self, state):
    current_count = state["investment_debate_state"]["count"]
    max_count = 2 * self.max_debate_rounds
    # 总发言次数 = 2 × 辩论轮数（Bull + Bear 各说一次算一轮）
    if current_count >= max_count:
        return "Research Manager"   # 辩论结束，交给研究经理
    # 根据当前发言者决定下一个发言者
    return "Bear Researcher" if current_speaker.startswith("Bull") else "Bull Researcher"
```

#### 风险讨论控制（`should_continue_risk_analysis`）

```python
def should_continue_risk_analysis(self, state):
    current_count = state["risk_debate_state"]["count"]
    max_count = 3 * self.max_risk_discuss_rounds
    # 总发言次数 = 3 × 讨论轮数（Risky + Safe + Neutral 各说一次算一轮）
    if current_count >= max_count:
        return "Risk Judge"          # 讨论结束，交给风险经理
    # 循环顺序：Risky → Safe → Neutral → Risky → ...
    return next_speaker  # 根据 latest_speaker 确定
```

---

## 附录：许可证说明

- **Apache 2.0 开源**：除 `app/` 和 `frontend/` 外的所有代码
- **专有许可**：`app/`（FastAPI 后端）和 `frontend/`（Vue 前端）目录，商业使用需要单独授权
- **联系授权**：hsliup@163.com
