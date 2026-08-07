## 1. 项目定位

### 1.1 项目是做什么的？

TGO 是一个**开源的 AI Agent 智能客服平台**，旨在帮助企业“为客服场景构建 AI Agent 团队”。它整合了多渠道接入、Agent 编排、知识库管理（RAG）和人机协作能力，是一套完整的企业级 AI 客服解决方案。

### 1.2 解决了什么业务问题？

- **客服人力成本高**：通过 AI Agent 自动处理大量重复性咨询，降低对人工客服的依赖。
- **多渠道管理碎片化**：统一接入网站、微信、企业微信、Slack、Telegram 等多个渠道，一个后台集中管理。
- **知识分散、响应不准确**：通过 RAG 知识库（文档、QA 对、网站爬取）提升回答准确性。
- **复杂问题需人工介入**：支持智能转人工（Human-AI Collaboration），复杂场景平滑交接。

### 1.3 目标用户是谁？

- **企业客服团队**：需要降本增效的客服运营部门。
- **SaaS 平台运营商**：希望为商户提供 AI 客服能力的平台方。
- **开发者/系统集成商**：需要二次开发或定制 AI 客服方案的技术团队。

### 1.4 属于哪一类 Agent？

**客服 Agent（Customer Service Agent）**。

核心场景是客服对话，同时融合了工作流 Agent（可视化 DAG 编排）和多 Agent 系统（Multi-Agent Support）的特征。

### 1.5 为什么需要 Agent，而不是普通应用？

| 维度     | 普通应用        | Agent 方案               |
| :------- | :-------------- | :----------------------- |
| 意图理解 | 固定关键词/菜单 | LLM 语义理解，开放式对话 |
| 知识检索 | 静态 FAQ 匹配   | RAG 动态检索+生成        |
| 工具调用 | 硬编码逻辑      | Agent 自主决策调用工具   |
| 多轮对话 | 有限状态机      | 上下文记忆+连贯对话      |
| 多渠道   | 各渠道独立开发  | 统一归一化+Agent 编排    |
| 复杂流程 | 代码写死        | 可视化 DAG 编排          |

客服场景的**开放性、多变性、长尾问题**，决定了硬编码无法覆盖，必须依赖 Agent 的自主推理和工具调用能力。

------

## 2. 业务流程

### 2.1 用户输入什么？

用户通过**网页聊天组件、微信小程序、微信公众号、企业微信、Slack、Telegram**等渠道输入自然语言问题。

### 2.2 系统输出什么？

- 流式/非流式的 AI 回答（文本、卡片、订单/商品结构化展示）
- 必要时转接人工客服
- 多轮对话中的上下文追问

### 2.3 完整流程示例

**场景**：用户通过网页聊天组件问“我的订单什么时候发货？”

| 步骤 | 做了什么                                       | 目的                     | 涉及模块                  |
| :--- | :--------------------------------------------- | :----------------------- | :------------------------ |
| 1    | 用户在网页聊天框输入问题                       | 发起客服请求             | tgo-widget-js（前端组件） |
| 2    | tgo-platform 接收消息，归一化为内部格式        | 统一多渠道消息格式       | tgo-platform              |
| 3    | tgo-api 进行会话路由和用户鉴权                 | 确定会话归属、多租户隔离 | tgo-api                   |
| 4    | tgo-ai 的 Agent 接收请求，结合上下文理解意图   | 判断用户问的是物流查询   | tgo-ai                    |
| 5    | Agent 决策：需要查订单状态 → 调用订单查询 Tool | 获取真实数据             | runtime/tools             |
| 6    | 如需知识支撑，调用 tgo-rag 进行向量检索        | 补充发货政策等信息       | tgo-rag                   |
| 7    | LLM 综合 Tool 结果 + RAG 上下文生成回答        | 生成准确、自然的回复     | tgo-ai（LLM 推理）        |
| 8    | 流式返回结果，前端渲染为文本/卡片              | 提升用户体验             | SSE / WebSocket           |
| 9    | 会话历史存入数据库，更新上下文                 | 支持多轮对话记忆         | PostgreSQL                |

------

## 3. 技术架构

### 3.1 前端、后端、Agent 服务分别用什么技术？

项目采用**微服务架构**，共包含 **8 个 Python 后端服务、1 个 Go Agent、2 个 React 前端、1 个微信小程序组件和 2 个 Node.js CLI 工具**：

| 服务                       | 技术栈                                                       | 端口       | 职责                    |
| :------------------------- | :----------------------------------------------------------- | :--------- | :---------------------- |
| **tgo-web**                | React 19 + TypeScript + Vite                                 | 5173       | 客服工作台前端          |
| **tgo-widget-js**          | React 18 + TypeScript + Vite                                 | 5173       | 网页嵌入式聊天组件      |
| **tgo-widget-miniprogram** | 微信小程序 / JS                                              | npm 包     | 小程序聊天组件          |
| **tgo-api**                | Python 3.11+ / FastAPI / PostgreSQL / SQLAlchemy / Redis / Kafka | 8000, 8001 | 核心业务 API            |
| **tgo-ai**                 | Python 3.11+ / FastAPI / PostgreSQL / agno / MCP             | 8081       | AI Agent 管理、工具集成 |
| **tgo-platform**           | Python 3.11+ / FastAPI / PostgreSQL / Redis                  | 8003       | 多渠道消息归一化        |
| **tgo-rag**                | Python / FastAPI / PostgreSQL+pgvector / Celery / Redis / LangChain | 8082       | RAG 检索增强生成        |
| **tgo-workflow**           | Python / FastAPI / Celery / Redis                            | 8000       | 工作流引擎（DAG 编排）  |
| **tgo-device-agent**       | Go                                                           | TCP        | 设备端 Agent            |

### 3.2 是否使用 LangChain、LangGraph 等技术？

- **LangChain**：tgo-rag 使用 LangChain + Unstructured 进行多格式文档解析
- **agno**：tgo-ai 使用 agno 作为 Agent 框架
- **MCP（Model Context Protocol）**：tgo-ai 集成 MCP，支持工具商店和自定义工具
- **pgvector**：tgo-rag 使用 PostgreSQL + pgvector 进行向量存储和混合搜索
- **Celery + Redis**：tgo-rag 和 tgo-workflow 使用 Celery 处理异步任务

**未明确使用**：LangGraph（未在文档中提及）、Deep Agents、GraphRAG。

### 3.3 是否使用数据库或向量数据库？

- **PostgreSQL**：所有后端服务共用，存储业务数据
- **pgvector**：tgo-rag 使用，支持向量存储和语义搜索
- **Redis**：缓存、消息队列、Celery broker
- **Kafka**：tgo-api 用于消息队列

### 3.4 核心目录职责

| 模块/目录                              | 作用                                    | 关键代码                                                     | 是否重点阅读 |
| :------------------------------------- | :-------------------------------------- | :----------------------------------------------------------- | :----------- |
| `repos/tgo-ai/app/api/v1/`             | API 路由（agents、chat、tools、skills） | `chat.py`, `agents.py`                                       | ✅ 是         |
| `repos/tgo-ai/app/runtime/supervisor/` | Agent 监督与编排核心                    | `agents/builder.py`, `agents/runner.py`, `orchestration/`    | ✅ 是         |
| `repos/tgo-ai/app/runtime/tools/`      | 工具构建、执行、自定义工具              | `builder/`, `executor/`, `custom/`                           | ✅ 是         |
| `repos/tgo-ai/app/services/`           | 业务逻辑层（Agent、Chat、MCP、LLM）     | `agent_runtime_service.py`, `chat_service.py`, `mcp_service.py` | ✅ 是         |
| `repos/tgo-ai/app/models/`             | SQLAlchemy ORM 模型                     | Agent/Tool/KnowledgeBase 模型                                | 否           |
| `repos/tgo-rag/app/`                   | RAG 核心：文档处理、向量化、混合搜索    | 文档解析、embedding、检索                                    | ✅ 是         |
| `repos/tgo-workflow/app/engine/`       | 工作流引擎核心：DAG 拓扑排序、节点执行  | `nodes/` (llm/api/condition/classifier)                      | ✅ 是         |
| `repos/tgo-api/app/`                   | 核心业务：用户、访客、会话、标签        | `api/v1/endpoints/`, `services/`                             | 否           |

### 3.5 请求入口、Agent 核心代码、Prompt、Tools、RAG 逻辑位置

| 组件               | 位置                                                         |
| :----------------- | :----------------------------------------------------------- |
| **请求入口**       | `tgo-ai/app/api/v1/chat.py`（对话接口）、`tgo-ai/app/api/v1/agents.py`（Agent 管理） |
| **Agent 核心代码** | `tgo-ai/app/runtime/supervisor/agents/`（Agent 构建与运行）  |
| **Agent 编排**     | `tgo-ai/app/runtime/supervisor/orchestration/`               |
| **Tools**          | `tgo-ai/app/runtime/tools/`（builder/executor/custom）       |
| **RAG 逻辑**       | `tgo-rag/`（独立微服务），通过 API 调用                      |
| **工作流**         | `tgo-workflow/app/engine/nodes/`（LLM/API/Condition/Classifier 节点） |

------

## 4. Agent 核心设计

### 4.1 Agent 的角色是什么？

TGO 的 Agent 是**客服场景下的自主决策实体**，角色定位为“AI 客服员工”。它能够：

- 理解用户意图
- 自主决定调用哪些工具
- 从知识库检索信息
- 生成自然语言回复
- 在无法处理时转交人工

### 4.2 Agent 如何理解用户任务？

通过 **LLM + 上下文记忆** 理解用户任务：

- 用户输入经由 tgo-platform 归一化后传入 tgo-ai
- Agent 结合**当前会话历史**（多轮对话记忆）和 **System Prompt** 进行意图理解
- 支持多模型（OpenAI、Anthropic、Gemini、Qwen 等）

### 4.3 Agent 是否会进行任务拆解或规划？

**是**。通过两种机制实现：

1. **Supervisor 编排**：`runtime/supervisor/` 模块负责 Agent 的监督和任务分发
2. **Workflow 引擎**：支持 DAG 拓扑执行，包含 `llm`、`api`、`condition`、`classifier`、`agent`、`tool` 等节点类型

这意味着复杂任务可以被拆解为多个步骤，按 DAG 拓扑顺序执行。

### 4.4 Agent 如何决定是否调用工具？

Agent 通过 **Function Calling** 机制自主决定：

- Agent 在初始化时绑定可用 Tools 列表
- LLM 根据用户问题推理是否需要调用工具、调用哪个工具
- 工具调用由 `runtime/tools/executor` 执行
- 支持 MCP 协议的工具集成

### 4.5 项目中有哪些 Tools？

| 工具类型                | 说明                                             |
| :---------------------- | :----------------------------------------------- |
| **MCP 工具**            | 工具商店中的标准化 MCP 工具，按需启用            |
| **自定义工具**          | 项目级工具配置和管理                             |
| **OpenAPI Schema 工具** | 自动解析 OpenAPI Schema 生成交互表单             |
| **内置工具**            | `runtime/tools/builder/`、`executor/`、`custom/` |

### 4.6 是否使用 RAG、MCP 或 Skills？

| 技术       | 是否使用 | 说明                                                         |
| :--------- | :------- | :----------------------------------------------------------- |
| **RAG**    | ✅ 是     | 独立 tgo-rag 微服务，支持文档/QA/网站知识库，混合搜索（向量+全文） |
| **MCP**    | ✅ 是     | tgo-ai 集成 MCP，支持工具商店和自定义工具                    |
| **Skills** | ✅ 是     | 有独立的 Skills 系统（`app/api/v1/skills.py`）和 GitHub Skill 下载器 |

### 4.7 Agent 如何生成最终结果？

**用户问题 → Agent 判断 → 工具/RAG 调用 → LLM 生成 → 返回结果**

text

```
用户输入
    ↓
tgo-platform（消息归一化）
    ↓
tgo-api（会话路由/鉴权）
    ↓
tgo-ai Agent Runtime
    ↓
┌─────────────────────────────────────┐
│  Supervisor/Agents（Agent 构建与运行） │
│  ├─ 结合上下文理解意图                │
│  ├─ 判断是否需要 Tool/RAG            │
│  ├─ 调用 Tools（executor）           │
│  ├─ 调用 RAG（tgo-rag API）          │
│  └─ LLM 生成最终回复                 │
└─────────────────────────────────────┘
    ↓
SSE/WebSocket 流式返回
    ↓
前端渲染（文本/卡片）
```



### 4.8 是否支持多轮对话、记忆或上下文管理？

**是**：

- **上下文记忆**：维护对话历史，支持连贯对话
- **多轮对话**：通过会话 ID 关联多轮交互
- **消息同步**：已读/未读状态、投递确认
- **会话历史持久化**：PostgreSQL 存储

------

## 5. 工程化与可落地性

| 维度               | 当前实现                                         | 不足                       | 改进建议                                   |
| :----------------- | :----------------------------------------------- | :------------------------- | :----------------------------------------- |
| **异常处理**       | FastAPI 原生异常处理 + `exceptions.py`           | 未见全局异常拦截和降级策略 | 增加全局 Exception Handler + 熔断降级      |
| **日志和监控**     | structlog 结构化日志；支持 `PROFILES=monitoring` | 监控面板需自行配置         | 集成 Prometheus + Grafana，预置 Dashboard  |
| **权限控制**       | JWT + API Key 双认证；多租户项目级隔离           | 未见细粒度 RBAC 文档       | 补充角色权限（管理员/客服/访客）的详细设计 |
| **工具调用安全**   | MCP 工具按需启用                                 | 工具执行无沙箱隔离         | 增加工具执行的沙箱环境（如 gVisor）        |
| **RAG 未命中处理** | 混合搜索（向量+全文）提升召回率                  | 未见明确的“未命中”降级策略 | 增加“未找到相关知识”的友好提示 + 转人工    |
| **扩展性**         | 支持自定义工具、MCP 工具商店、Skills 系统        | 扩展文档不够详细           | 完善开发者文档和插件 SDK                   |
| **多租户**         | 项目级隔离和访问控制                             | —                          | —                                          |
| **容器化**         | Docker Compose 一键部署                          | 生产级 K8s 配置缺失        | 补充 Helm Charts 和生产部署指南            |

------

## 6. 总结与学习价值

### 6.1 核心亮点

1. **完整的企业级微服务架构**：8 个 Python 服务 + 1 个 Go 服务 + 2 个 React 前端，覆盖 Agent、RAG、工作流、多渠道全链路
2. **MCP 协议集成**：工具商店 + 自定义工具 + OpenAPI Schema 自动解析
3. **混合搜索 RAG**：pgvector 向量搜索 + 全文搜索 + 倒数排名融合
4. **可视化工作流引擎**：DAG 编排，支持 LLM/API/Condition/Classifier/Agent/Tool 节点
5. **多渠道统一接入**：微信公众号、企业微信、Slack、Telegram、邮件、WebSocket

### 6.2 不足

1. **文档不完整**：API 文档虽有 OpenAPI，但架构设计和二次开发文档较薄弱
2. **生产级可观测性缺失**：虽有 structlog 和 monitoring profile，但缺乏预置的监控大盘
3. **工具执行安全**：未见工具执行的沙箱隔离机制
4. **依赖较重**：8 个微服务 + PostgreSQL + Redis + Kafka，中小团队部署成本高

### 6.3 值得学习的设计

1. **微服务拆分粒度**：按领域（API/AI/RAG/Workflow/Platform）拆分，职责清晰
2. **MCP 工具抽象**：标准化的工具接入方式，降低扩展成本
3. **Supervisor 模式**：Agent 监督与编排的抽象设计
4. **混合搜索 RAG**：pgvector + 全文搜索的融合策略
5. **双认证机制**：JWT（用户态）+ API Key（服务间）

### 6.4 不适合直接照搬的设计

1. **微服务数量**：8 个服务对中小团队过重，可考虑合并为 3-4 个核心服务
2. **agno 框架绑定**：如果团队更熟悉 LangChain/LangGraph，可替换
3. **全量 Docker Compose**：生产环境建议用 K8s，需额外改造

### 6.5 二次开发优先改造建议

1. **简化部署**：合并部分微服务，减少运维负担
2. **补充监控**：集成 Prometheus + Grafana，配置告警规则
3. **完善 Prompt 管理**：建立 Prompt 版本管理和 A/B 测试能力
4. **增强工具安全**：增加工具执行的沙箱和审计日志
5. **补充测试覆盖率**：完善单元测试和集成测试

### 6.6 面试表达

> **项目背景**：TGO 是一个开源的 AI Agent 智能客服平台，目标是为企业提供“AI 客服团队”构建能力。它解决了传统客服系统在多渠道管理、知识分散、人工成本高这三个核心痛点。
>
> **技术架构**：项目采用微服务架构，核心是 8 个 Python 服务（基于 FastAPI）和 2 个 React 前端。其中 tgo-ai 是 Agent 核心服务，tgo-rag 负责 RAG 检索，tgo-workflow 是可视化工作流引擎，tgo-platform 负责多渠道消息归一化。数据层使用 PostgreSQL + pgvector 向量数据库 + Redis + Kafka。
>
> **Agent 工作流**：用户通过网页或微信等渠道发送消息 → tgo-platform 归一化 → tgo-api 路由鉴权 → tgo-ai 的 Supervisor 模块构建 Agent 并执行推理 → Agent 自主判断是否需要调用工具或检索 RAG → LLM 生成回复 → SSE/WebSocket 流式返回。整个流程支持多轮对话记忆和上下文管理。
>
> **Tools / RAG / MCP / Skills**：项目完整集成了这四个能力。Tools 通过 MCP 协议接入，支持工具商店和自定义工具；RAG 是独立微服务，支持文档/QA/网站三种知识库，采用向量+全文混合搜索；Skills 有独立的下载和管理系统，支持从 GitHub 拉取 Skill 定义。
>
> **项目亮点**：一是微服务拆分合理，AI、RAG、Workflow 职责清晰；二是 MCP 协议集成降低了工具扩展成本；三是可视化 DAG 工作流引擎支持复杂业务流程编排；四是多渠道统一接入，一套 Agent 服务支撑所有渠道。
>
> **企业级落地思考**：如果要用于生产，需要补充三件事：一是完善可观测性，预置 Prometheus 监控和日志聚合；二是增加工具执行的沙箱隔离，防止恶意工具影响系统；三是简化部署，对中小团队来说 8 个微服务太重，可以考虑合并部分服务或提供 All-in-One 部署模式。