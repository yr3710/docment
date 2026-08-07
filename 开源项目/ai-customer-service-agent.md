## 1. 项目定位

### 1.1 项目是做什么的？

这是一个基于 **LangGraph** 构建的**多 Agent 电商智能客服系统**，涵盖意图分类、RAG 知识库检索、Function Calling 工具调用、工单升级等完整客服流程。项目提供了 Web UI 和 CLI 两种交互方式。

### 1.2 它解决了什么业务问题？

解决了电商场景下客服系统的三个核心痛点：

- **响应效率低**：人工客服无法 7×24 小时即时响应，通过 AI Agent 实现自动化应答
- **知识管理难**：产品信息、退货政策等分散在不同文档，通过 RAG 实现统一知识检索
- **复杂查询处理难**：订单查询、物流追踪等需要对接业务系统的操作，通过 Function Calling 实现自动化

### 1.3 目标用户是谁？

- **中小电商企业**：希望快速搭建 AI 客服系统但缺乏 AI 研发能力
- **开发者/技术团队**：希望学习或二次开发 Agent 应用
- **SaaS 服务商**：希望将客服 Agent 作为产品功能嵌入现有系统

### 1.4 属于哪一类 Agent？

**客服 Agent（Customer Service Agent）**。具体采用了多 Agent 协作架构，包含 Router（路由）、Knowledge（知识库）、Tool（工具）、Escalation（工单升级）、Summary（汇总）五个子 Agent。

### 1.5 为什么需要用 Agent，而不是普通应用？

| 普通应用                         | Agent 方案                               |
| :------------------------------- | :--------------------------------------- |
| 固定规则匹配，无法理解开放性问题 | LLM 理解自然语言，支持开放式问答         |
| 意图识别靠关键词，准确率低       | Router Agent 用 LLM 做语义级别的意图分类 |
| 知识更新需要改代码               | RAG 实现知识库与代码解耦                 |
| 无法主动调用外部系统             | Function Calling 动态调用订单/物流接口   |
| 无法处理复杂多步骤任务           | LangGraph 状态图支持多步推理和条件分支   |

------

## 2. 业务流程

### 2.1 用户输入什么？

用户在聊天框输入自然语言问题，例如：

- 产品查询：`"你们有什么产品？"`
- 订单查询：`"查询订单 ORD-001"`
- 物流追踪：`"ORD-001 到哪里了？"`
- 退货咨询：`"怎么退货？"`
- 投诉：`"我要投诉"`
- 打招呼：`"你好"`

### 2.2 系统输出什么？

系统输出 AI 客服的自然语言回复，同时在前端展示 **"Agent 处理过程"** 的思考链，用户可以展开查看每个 Agent 的处理细节。

### 2.3 完整流程

| 步骤 | 做了什么                   | 目的                                                         | 涉及模块                                 |
| :--- | :------------------------- | :----------------------------------------------------------- | :--------------------------------------- |
| 1    | 用户发送消息               | 输入用户问题                                                 | 前端 → `/api/chat` API                   |
| 2    | Router Agent 进行意图分类  | 判断用户意图属于 `order_query` / `product_inquiry` / `complaint` / `general` 等 | `router_agent.py`                        |
| 3    | 条件路由：根据意图分流     | 决定下一步走 Knowledge、Tool 还是 Escalation 路径            | `graph.py` 的条件边                      |
| 4a   | Knowledge Agent 检索向量库 | 从知识库检索相关文档（top 3）                                | `knowledge_agent.py` + `vector_store.py` |
| 4b   | Tool Agent 调用函数        | 执行 `query_order` / `track_shipment` 等工具调用             | `tool_agent.py` + `tools` 定义           |
| 4c   | Escalation Agent 生成工单  | 投诉场景生成结构化工单（摘要、分类、紧急度）                 | `escalation_agent.py`                    |
| 5    | Summary Agent 汇总结果     | 整合各 Agent 的输出，生成最终自然语言回复                    | `summary_agent.py`                       |
| 6    | 返回结果并展示思考链       | 用户看到回复 + 可展开的 Agent 处理过程                       | 前端渲染                                 |

**具体例子**：用户输入 `"查询订单 ORD-001"`

text

```
用户 → Router Agent（识别为 order_query）→ Tool Agent（调用 query_order("ORD-001")）
→ Summary Agent（整合订单状态信息）→ 返回"您的订单 ORD-001 当前状态为已发货..."
```



------

## 3. 技术架构

### 3.1 技术栈

| 层级       | 技术                                                     | 说明                    |
| :--------- | :------------------------------------------------------- | :---------------------- |
| 前端       | Next.js + React + TypeScript                             | 提供 Web UI             |
| 后端       | FastAPI + Python                                         | 提供 REST API           |
| Agent 框架 | LangGraph                                                | 构建多 Agent 状态图     |
| LLM        | 兼容 OpenAI API 的模型（DeepSeek / OpenAI / 通义千问等） | 通过 `LLM_API_KEY` 配置 |
| 向量检索   | 内存向量存储 + 关键词匹配                                | 轻量级 RAG 实现         |
| 部署       | Docker / Docker Compose                                  | 一键启动                |

### 3.2 关键技术使用情况

| 技术             | 是否使用 | 说明                           |
| :--------------- | :------- | :----------------------------- |
| LangChain        | ✅        | 用于 LLM 调用封装              |
| LangGraph        | ✅        | 核心：构建多 Agent 状态图      |
| Deep Agents      | ❌        | 未使用                         |
| GraphRAG         | ❌        | 使用简单 RAG，非 GraphRAG      |
| Function Calling | ✅        | Tool Agent 通过 tools 定义调用 |
| MCP              | ❌        | 未使用                         |
| RAG              | ✅        | 知识库检索                     |

### 3.3 数据库与向量数据库

- **未使用传统数据库**：订单/物流数据通过 `DataProvider` 接口提供演示数据
- **向量存储**：使用内存中的简单向量存储，通过关键词分词 + 集合匹配实现检索
- **设计理念**：**"可插拔数据源"**——接入真实电商数据库只需实现一个接口，无需修改业务代码

### 3.4 核心目录结构

| 模块/目录                                    | 作用                           | 关键代码                                 | 是否重点阅读 |
| :------------------------------------------- | :----------------------------- | :--------------------------------------- | :----------- |
| `backend/app/main.py`                        | FastAPI 入口，`/api/chat` 接口 | `chat()` 异步函数                        | ✅            |
| `backend/app/agents/graph.py`                | LangGraph 状态图定义           | `build_graph()` 构建 5 个节点 + 条件边   | ✅            |
| `backend/app/agents/router_agent.py`         | 意图分类 Agent                 | `ROUTER_SYSTEM_PROMPT` + `router_node()` | ✅            |
| `backend/app/agents/knowledge_agent.py`      | RAG 检索 Agent                 | 调用 `vector_store.query()`              | ✅            |
| `backend/app/agents/tool_agent.py`           | 工具调用 Agent                 | `TOOLS` 定义 + Function Calling          | ✅            |
| `backend/app/agents/escalation_agent.py`     | 工单升级 Agent                 | 生成结构化工单                           | ✅            |
| `backend/app/agents/summary_agent.py`        | 结果汇总 Agent                 | 整合各节点输出生成最终回复               | ✅            |
| `backend/app/knowledge_base/vector_store.py` | 向量存储与检索                 | `_load_docs()` + `_tokenize()`           | ✅            |
| `backend/app/models/schemas.py`              | Pydantic 数据模型              | `AgentState`、`ChatRequest`              | 可选         |
| `backend/app/data/config.py`                 | 数据提供者配置                 | `get_data_provider()` 工厂方法           | 可选         |
| `frontend/app/page.tsx`                      | 聊天界面主组件                 | 消息发送 + Agent 思考链展示              | 可选         |

------

## 4. Agent 核心设计

### 4.1 Agent 的角色是什么？

系统采用 **5 个专职 Agent 协作** 的架构：

| Agent      | 角色       | 职责                             |
| :--------- | :--------- | :------------------------------- |
| Router     | 路由决策者 | 理解用户意图，决定走哪条处理路径 |
| Knowledge  | 知识检索者 | 从知识库检索相关信息             |
| Tool       | 工具执行者 | 调用外部函数（订单、物流等）     |
| Escalation | 工单生成者 | 投诉场景生成结构化工单           |
| Summary    | 结果汇总者 | 整合所有信息生成最终回复         |

### 4.2 Agent 如何理解用户任务？

**Router Agent** 通过 System Prompt 引导 LLM 进行意图分类：

text

```
你是一个电商客服意图分类器。请根据用户的输入，判断以下类别：
- order_query: 查询订单、物流、退款
- product_inquiry: 询问产品、库存、价格
- complaint: 投诉、不满、差评
- general: 其他一般性对话[reference:26]
```



LLM 返回结构化 JSON：`{"intent": "<类别>", "reason": "<原因>"}`

### 4.3 Agent 是否会进行任务拆解或规划？

**会**。LangGraph 状态图本身就是一个任务规划器：

1. **第一步**：Router 做意图分类（单步决策）
2. **第二步**：根据意图**条件路由**到不同节点
3. **第三步**：Knowledge 或 Tool 节点执行具体任务
4. **第四步**：Summary 汇总所有中间结果

这种设计本质上是将复杂任务拆解为"分类 → 执行 → 汇总"三个阶段。

### 4.4 Agent 如何决定是否调用工具？

**Tool Agent** 通过 **Function Calling** 机制决定：

python

```
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "query_order",
            "description": "查询订单信息",
            "parameters": {
                "properties": {
                    "order_id": {"type": "string", "description": "订单编号"}
                }
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "track_shipment",
            "description": "追踪物流信息",
            ...
        }
    }
]
```



LLM 根据用户问题自主决定是否调用工具、调用哪个工具、传入什么参数——这是典型的 **ReAct（Reasoning + Acting）模式**。

### 4.5 项目中有哪些 Tools？

| Tool 名称        | 功能         | 参数       |
| :--------------- | :----------- | :--------- |
| `query_order`    | 查询订单信息 | `order_id` |
| `track_shipment` | 追踪物流信息 | `order_id` |

### 4.6 是否使用 RAG、MCP 或 Skills？

| 技术       | 使用情况                                                   |
| :--------- | :--------------------------------------------------------- |
| **RAG**    | ✅ 使用。通过 `knowledge_agent.py` 从向量库检索 top 3 文档  |
| **MCP**    | ❌ 未使用                                                   |
| **Skills** | ❌ 未使用（工具通过 Function Calling 实现，非 Skills 模式） |

### 4.7 Agent 如何生成最终结果？

**Summary Agent** 汇总所有中间状态：

python

```
def summary_node(state):
    user_msg = state["user_message"]
    intent = state.get("intent", "general")
    retrieved_docs = state.get("retrieved_docs", [])
    tool_results = state.get("tool_results", [])
    ticket = state.get("escalation_ticket")
    
    # 构建上下文 → 调用 LLM → 生成最终回复
```



### 4.8 是否支持多轮对话、记忆或上下文管理？

从代码结构看，当前实现**以单轮对话为主**：

- `AgentState` 包含 `messages` 字段，但主要用于记录而非多轮记忆
- 未发现明显的会话持久化或历史记忆机制
- 多轮对话能力需要额外扩展

### 核心调用链路

text

```
用户问题 
    → Router Agent（LLM 意图分类） 
    → 条件路由 
        ├─ product_inquiry → Knowledge Agent（向量检索）→ Summary
        ├─ order_query → Tool Agent（Function Calling）→ Summary
        ├─ complaint → Escalation Agent（生成工单）→ Summary
        └─ general → Summary（直接回复）
    → Summary Agent（LLM 汇总生成）
    → 返回结果
```



------

## 5. 工程化与可落地性

| 维度               | 当前实现                        | 不足                                    | 改进建议                                                     |
| :----------------- | :------------------------------ | :-------------------------------------- | :----------------------------------------------------------- |
| **异常处理**       | `main.py` 中有 try-except 包裹  | 颗粒度粗，缺少细粒度异常分类            | 增加各类异常的专门处理（LLM 超时、工具调用失败、RAG 无结果） |
| **日志和监控**     | 基础 print 输出                 | 无结构化日志、无链路追踪、无指标采集    | 接入 logging 模块 + OpenTelemetry + Prometheus               |
| **权限控制**       | 无                              | 无用户认证、无 API 鉴权                 | 增加 JWT 认证 + RBAC 权限控制                                |
| **工具调用安全**   | 工具函数直接执行                | 无参数校验、无调用审计、无速率限制      | 增加参数白名单校验、调用日志、限流器                         |
| **RAG 没命中处理** | 检索不到时可能返回空            | 未见明确的 fallback 逻辑                | 增加"未找到相关信息，请换个问法"的降级回复                   |
| **扩展性**         | DataProvider 接口支持数据源替换 | 新增工具需要改代码、新增 Agent 需要改图 | 设计插件化 Tool 注册机制、可视化 Agent 编排                  |
| **会话管理**       | 无持久化                        | 重启丢失会话、无法跨设备同步            | 增加 Redis 会话存储 + 数据库历史记录                         |
| **多租户**         | 无                              | 单租户设计                              | 增加 tenant_id 隔离                                          |
| **评测体系**       | 无                              | 无法评估 Agent 回答质量                 | 增加离线评测数据集 + 人工标注反馈闭环                        |
| **成本控制**       | 无                              | 每次请求都调用 LLM，无缓存              | 增加语义缓存（Semantic Cache）减少重复调用                   |

### 企业级落地补充能力清单

1. **用户认证与鉴权**：OAuth2 / JWT
2. **会话持久化**：Redis + PostgreSQL
3. **可观测性**：结构化日志 + 分布式追踪 + 业务指标
4. **高可用**：服务多实例 + 负载均衡 + 健康检查
5. **灰度发布**：A/B 测试不同 Agent 版本
6. **数据隐私**：敏感信息脱敏、数据合规（GDPR）
7. **知识库管理**：可视化知识库编辑、版本控制、自动更新
8. **人机协同**：Agent 无法处理时转人工的平滑过渡

------

## 6. 总结与学习价值

### 6.1 核心亮点

1. **架构清晰**：5 个专职 Agent 各司其职，LangGraph 状态图一目了然
2. **可插拔数据源**：通过 DataProvider 接口实现业务数据解耦，设计优雅
3. **思考链可视化**：前端展示 Agent 处理过程，用户体验好、可调试性强
4. **多模型兼容**：支持任何兼容 OpenAI API 的 LLM
5. **开箱即用**：Docker 一键启动 + CLI 模式，降低使用门槛
6. **代码精简**：核心 Agent 逻辑集中，适合学习和二次开发

### 6.2 不足

1. **向量检索过于简单**：使用关键词分词 + 集合匹配，非真正语义向量检索
2. **无持久化存储**：会话、知识库、工单都是内存存储，重启丢失
3. **缺少评测体系**：无法客观评估 Agent 回答质量
4. **多轮对话能力弱**：当前以单轮为主
5. **缺少安全机制**：无认证、无审计、无限流

### 6.3 值得学习的设计

1. **LangGraph 状态图模式**：用图来表达 Agent 工作流，比手写 if-else 优雅得多
2. **Router → Executor → Summary 的三段式架构**：可复用到其他 Agent 场景
3. **DataProvider 接口抽象**：业务数据与 Agent 逻辑解耦
4. **Function Calling 的标准用法**：tools 定义的 JSON Schema 写法
5. **思考链透传**：将 Agent 内部状态传递给前端，提升可解释性

### 6.4 不适合直接照搬的设计

1. **内存向量存储**：生产环境应使用 Chroma / Milvus / Pinecone 等
2. **无持久化设计**：生产环境必须有数据库
3. **单轮对话模式**：真实客服场景需要多轮上下文
4. **硬编码的 Tools**：生产环境需要动态注册机制

### 6.5 二次开发优先改造点

1. **第一期（MVP 增强）**：接入真实向量数据库（Chroma）+ 持久化会话（Redis）
2. **第二期（企业就绪）**：增加用户认证（JWT）+ 结构化日志 + 异常处理完善
3. **第三期（智能化升级）**：增加多轮对话记忆、工具动态注册、人机协同转人工
4. **第四期（规模化）**：增加评测体系、灰度发布、多租户支持

### 6.6 面试表达参考

> **项目背景**：这是一个基于 LangGraph 构建的电商智能客服系统，采用多 Agent 协作架构，涵盖意图分类、RAG 知识库检索、Function Calling 工具调用和工单升级等完整客服流程。
>
> **技术架构**：前端使用 Next.js + React，后端使用 FastAPI + Python，Agent 框架基于 LangGraph 构建状态图，LLM 层兼容 DeepSeek、OpenAI 等任何支持 OpenAI API 的模型。数据层设计了可插拔的 DataProvider 接口，接入真实业务数据无需修改核心代码。
>
> **Agent 工作流**：系统包含 Router、Knowledge、Tool、Escalation、Summary 五个专职 Agent。用户消息先由 Router 做意图分类，然后根据意图条件路由到不同节点——产品咨询走 RAG 检索知识库，订单查询走 Function Calling 调用业务接口，投诉走工单升级流程，最后由 Summary Agent 汇总所有中间结果生成最终回复。整个流程通过 LangGraph 的状态图编排，每个节点的输入输出都在 AgentState 中流转。
>
> **Tools / RAG 使用情况**：Tools 层面通过标准的 Function Calling 机制定义了订单查询和物流追踪两个工具；RAG 层面使用向量存储进行知识库检索，当前是内存实现，生产环境可替换为 Chroma 或 Milvus；项目暂未使用 MCP 和 Skills。
>
> **项目亮点**：第一是架构清晰，多 Agent 各司其职，LangGraph 状态图可读性强；第二是可插拔数据源设计，通过接口抽象实现业务解耦；第三是思考链可视化，用户可以看到 Agent 的完整推理过程，提升了可解释性和调试效率；第四是开箱即用，支持 Docker 一键部署和 CLI 两种模式。
>
> **企业级落地思考**：如果要投入生产，我会优先补充三点——一是接入真正的向量数据库和关系型数据库，解决内存存储的局限性；二是增加用户认证、审计日志和异常处理体系；三是建立离线评测数据集和人工反馈闭环，持续优化 Agent 的回复质量。此外，多轮对话记忆、工具动态注册、人机协同转人工等能力也需要根据业务需求逐步完善。