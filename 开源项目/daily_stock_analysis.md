## 1. 项目定位

### 1.1 这个项目是做什么的？

`daily_stock_analysis` 是一个 **LLM 驱动的多市场股票智能分析系统**。它能够对用户自选股（覆盖 A 股、港股、美股、日股、韩股、台股及 ETF）进行每日自动分析，生成结构化的「决策仪表盘」报告，并通过企业微信、飞书、Telegram、Discord、Slack、邮箱等渠道推送。

### 1.2 它解决了什么业务问题？

这个项目解决的核心痛点是：**个人投资者每日复盘效率低**——盯盘、看新闻、查公告、读研报，一套流程下来耗时数小时。项目通过 AI 自动完成行情聚合、技术指标计算、新闻检索、公告筛选、基本面分析，最终输出包含核心结论、评分、趋势判断、买卖点位、风险警报、催化因素、操作检查清单的决策报告。

### 1.3 目标用户是谁？

- 个人投资者/散户：需要快速获取自选股的每日分析
- 量化交易爱好者：希望将 AI 分析集成到自己的交易流程
- 开源社区开发者：学习或二次开发 AI Agent 应用

### 1.4 它属于哪一类 Agent？

**数据分析 Agent + 工作流 Agent 的混合体**。

更准确地说，它是一个 **多 Agent 系统**。项目内部包含多个子 Agent：TechnicalAgent（技术分析）、IntelAgent（情报/新闻分析）、RiskAgent（风险评估）、DecisionAgent（决策合成）、PortfolioAgent（持仓分析）、SkillAgent（策略问股）。这些子 Agent 通过编排 Pipeline 协同工作，最终生成综合决策报告。

### 1.5 为什么这个项目需要用 Agent，而不是普通应用？

| 维度     | 普通应用             | Agent 方案                               |
| :------- | :------------------- | :--------------------------------------- |
| 数据理解 | 按固定规则解析数据   | LLM 理解非结构化新闻、公告语义           |
| 分析维度 | 预设指标计算         | 多维度综合分析（技术+基本面+新闻+风险）  |
| 交互方式 | 单次查询返回固定结果 | 多轮追问、策略切换                       |
| 策略扩展 | 硬编码新策略         | 15 种内置策略 + 自定义 YAML 策略         |
| 推理能力 | 无自主推理           | Agent 自主判断调用哪些工具、如何综合信息 |

股票分析涉及大量非结构化信息（新闻情绪、公告解读、市场传闻），普通应用只能做结构化数据展示，无法完成「综合判断」和「自主推理」——这正是 Agent 的价值所在。

## 2. 业务流程

### 2.1 用户输入什么？

- **自选股列表**：股票代码或名称（支持 A 股、港股、美股等多市场）
- **分析指令**：可以是定时触发的每日分析，也可以是 Web/Bot/API 发起的即时问股
- **策略选择**（Agent 模式下）：从 15 种内置策略中选择，如均线、缠论、波浪、趋势、热点、事件、成长、预期等

### 2.2 系统输出什么？

- **决策仪表盘**：核心结论、综合评分、趋势判断、买卖点位、风险警报、催化因素、操作检查清单
- **推送通知**：通过企业微信/飞书/Telegram/Discord/Slack/邮箱发送
- **多轮对话回复**（Agent 问股模式）：基于策略的追问与回答

### 2.3 从用户输入到最终结果，中间经历了哪些步骤？

| 步骤 | 做了什么                                                     | 目的                          | 涉及模块                                                     |
| :--- | :----------------------------------------------------------- | :---------------------------- | :----------------------------------------------------------- |
| 1    | 用户配置自选股或发起分析请求                                 | 确定分析标的                  | Web/Bot/API 入口                                             |
| 2    | 系统解析股票代码，识别市场类型（A/H/US 等）                  | 确定数据源和分析口径          | `api/`、代码解析模块                                         |
| 3    | 多数据源行情聚合（AkShare、Tushare、YFinance、Longbridge 等） | 获取 K 线、技术指标、实时行情 | `data_provider/`                                             |
| 4    | 新闻/公告检索                                                | 获取市场情绪和事件驱动信息    | 新闻搜索模块                                                 |
| 5    | 多 Agent 编排 Pipeline 执行                                  | 各子 Agent 独立分析并汇总     | TechnicalAgent、IntelAgent、RiskAgent、DecisionAgent、PortfolioAgent |
| 6    | LLM 生成结构化决策报告                                       | 综合所有数据输出最终结论      | LLM 服务（支持 Gemini、OpenAI、DeepSeek、通义千问、Claude、Ollama 等） |
| 7    | 报告推送或返回给用户                                         | 交付分析结果                  | 推送模块（企业微信/飞书等）                                  |

### 2.4 完整流程示例

> **用户场景**：张三每天早上 9:00 通过 GitHub Actions 定时触发分析，自选股包括「贵州茅台（A股）」「腾讯控股（港股）」「苹果（美股）」。

1. **触发**：GitHub Actions 定时任务启动每日分析工作流
2. **数据采集**：系统并行调用 `data_provider/` 下的多个 Fetcher：
   - 贵州茅台 → AkShare/Tushare 获取 A 股数据
   - 腾讯控股 → Efinance/Longbridge 获取港股数据
   - 苹果 → YFinance 获取美股数据
3. **新闻检索**：通过 Anspire 等新闻搜索服务获取三只股票的相关新闻
4. **多 Agent 分析**：
   - TechnicalAgent 计算均线、缠论、波浪等技术指标
   - IntelAgent 分析新闻情绪和事件催化
   - RiskAgent 评估风险等级
   - DecisionAgent 综合生成决策建议
5. **LLM 生成报告**：调用配置的大模型（如 DeepSeek）生成含评分、买卖点位的结构化报告
6. **推送**：报告通过企业微信机器人推送到张三的手机

## 3. 技术架构

### 3.1 前端、后端、Agent 服务分别用什么技术？

| 层级       | 技术栈                                      |
| :--------- | :------------------------------------------ |
| 前端       | React（`apps/dsa-web/`）                    |
| 桌面端     | Electron（`apps/dsa-desktop/`）             |
| 后端 API   | FastAPI（`api/app.py`）                     |
| Agent 服务 | Python 原生实现，未使用 LangChain/LangGraph |
| 数据层     | Pandas、NumPy                               |
| 部署       | Docker、GitHub Actions                      |

### 3.2 是否使用 LangChain、LangGraph、Deep Agents 等技术？

**明确不使用 LangChain/LangGraph**。

根据社区分析文章指出：「这里没有实体关系抽取，没有向量检索增强，没有 RAG 流水线。有的只是三段清晰的指令、明确的长度约束、严格的格式要求」。项目采用的是**自研的轻量级 Agent 框架**，而非依赖 LangChain 等重型框架。

项目中的「多 Agent 编排 Pipeline」是自研实现，通过环境变量为每个子 Agent 配置独立超时钳位。

### 3.3 是否使用数据库或向量数据库？

- **数据库**：推测使用 SQLite 或类似轻量级数据库存储历史报告、用户配置等（从 Issue 中可见数据库操作相关描述）
- **向量数据库**：**不使用**。项目未实现 RAG 流水线

### 3.4 核心目录结构与职责

| 模块/目录           | 作用                                                 | 关键代码                                                     | 是否重点阅读 |
| :------------------ | :--------------------------------------------------- | :----------------------------------------------------------- | :----------- |
| `api/`              | FastAPI 后端服务，提供 REST API、路由注册、CORS 配置 | `app.py`（应用工厂）、`deps.py`（依赖注入）                  | ✅ 是         |
| `api/v1/`           | API 版本路由                                         | 各业务路由                                                   | ✅ 是         |
| `apps/dsa-web/`     | React 前端                                           | 前端源码                                                     | 可选         |
| `apps/dsa-desktop/` | Electron 桌面端                                      | 桌面端源码                                                   | 可选         |
| `bot/`              | Bot 交互模块，命令分发与处理                         | `dispatcher.py`（命令分发器）、`handler.py`（处理器）、`commands/`（命令实现） | ✅ 是         |
| `data_provider/`    | 多源数据适配器，策略模式管理数据源                   | 各 Fetcher（akshare、tushare、yfinance 等）                  | ✅ 是         |
| `docker/`           | Docker 配置                                          | Dockerfile、docker-compose                                   | 可选         |
| `.claude/skills/`   | Claude Code Agent Skill 定义                         | SKILL.md                                                     | 可选         |

### 3.5 请求入口、Agent 核心代码、Prompt、Tools、RAG 逻辑分别在哪里？

| 组件               | 位置                                                         |
| :----------------- | :----------------------------------------------------------- |
| **请求入口**       | `api/app.py`（FastAPI 应用）、`bot/dispatcher.py`（Bot 命令分发）、`main.py`（CLI 入口） |
| **Agent 核心代码** | 分散在多个模块，核心是「多 Agent 编排 Pipeline」（具体实现需查看 `pipeline/` 或类似目录） |
| **Prompt**         | 通过环境变量或配置文件注入，LLM Prompt 会按股票市场动态注入上下文 |
| **Tools**          | `data_provider/` 中的各数据源 Fetcher 可视为 Tools；CLI/API/MCP/Agent 四通道操作 |
| **RAG 逻辑**       | **无 RAG 实现**                                              |

## 4. Agent 核心设计

### 4.1 Agent 的角色是什么？

系统包含 **6 个子 Agent**，各司其职：

| Agent          | 职责                           |
| :------------- | :----------------------------- |
| TechnicalAgent | 技术分析（均线、缠论、波浪等） |
| IntelAgent     | 情报分析（新闻、公告、事件）   |
| RiskAgent      | 风险评估                       |
| DecisionAgent  | 决策合成与建议生成             |
| PortfolioAgent | 持仓分析                       |
| SkillAgent     | 策略问股（15 种内置策略）      |

### 4.2 Agent 如何理解用户任务？

通过 **Prompt 工程** 实现：

- 系统 Prompt 会根据股票代码识别市场类型（A 股/港股/美股），注入对应的角色描述与交易规则提示
- 采用「三段清晰的指令、明确的长度约束、严格的格式要求」控制输出
- 对关键行为反复强调（如「中性」「不承诺」「不缩写」）

### 4.3 Agent 是否会进行任务拆解或规划？

**会**。项目采用「多 Agent 编排 Pipeline」：

1. 先进行 Input Preparation（输入准备）
2. 然后执行 Risk Application（风险应用）
3. 最后进行 Post-Risk Finalization（风险后处理）
4. Pipeline 完成后补充结构、资金流、市场阶段、每日市场上下文的护栏

### 4.4 Agent 如何决定是否调用工具？

工具调用逻辑内置于各子 Agent 的实现中。以 `data_provider/` 为例：

- 采用**策略模式 + 优先级降级**：优先级数字越小越优先，同优先级按初始化顺序排列
- 配置了 TUSHARE_TOKEN 时，TushareFetcher 优先级最高；未配置时 EfinanceFetcher 优先
- 腾讯直连日 K 作为最终兜底

### 4.5 项目中有哪些 Tools？

| Tool                | 数据源          | 覆盖市场      |
| :------------------ | :-------------- | :------------ |
| TushareFetcher      | Tushare         | A 股          |
| EfinanceFetcher     | efinance        | A 股          |
| AkshareFetcher      | AkShare         | A 股          |
| PytdxFetcher        | pytdx（通达信） | A 股          |
| BaostockFetcher     | baostock        | A 股          |
| YfinanceFetcher     | yfinance        | 美股          |
| LongbridgeFetcher   | 长桥 OpenAPI    | 美股/港股     |
| TencentFetcher      | 腾讯直连        | 日 K 最终兜底 |
| FinnhubFetcher      | Finnhub         | 美股          |
| AlphaVantageFetcher | Alpha Vantage   | 美股          |

此外，新闻搜索通过 Anspire 等服务实现。项目还提供 MCP Server，暴露 14 个工具。

### 4.6 是否使用 RAG、MCP 或 Skills？

| 技术       | 是否使用 | 说明                                                         |
| :--------- | :------- | :----------------------------------------------------------- |
| **RAG**    | ❌ 不使用 | 无向量数据库，无检索增强流水线                               |
| **MCP**    | ✅ 使用   | 项目提供 MCP Server，暴露 14 个工具                          |
| **Skills** | ✅ 使用   | 项目可作为 Claude Code Agent Skill 安装；内部也有 SkillAgent 处理策略问股 |

### 4.7 Agent 如何生成最终结果？

1. 各子 Agent 独立分析 → 产出结构化中间结果
2. Pipeline 汇总所有子 Agent 输出
3. 调用 LLM（支持 Gemini、OpenAI、DeepSeek、通义千问、Claude、Ollama 等）生成最终报告
4. 报告包含：核心结论、评分、趋势、买卖点位、风险警报、催化因素、操作检查清单

### 4.8 是否支持多轮对话、记忆或上下文管理？

- **多轮对话**：✅ 支持。Agent 策略问股模式支持多轮追问
- **记忆**：✅ 支持。Agent Chat Skill 选择按会话持久化，支持刷新和会话切换恢复
- **上下文管理**：✅ 支持。LLM Prompt 按股票市场动态注入上下文

## 5. 工程化与可落地性

| 维度                 | 当前实现                                         | 不足                                     | 改进建议                                             |
| :------------------- | :----------------------------------------------- | :--------------------------------------- | :--------------------------------------------------- |
| **异常处理**         | 多数据源自动故障切换；子 Agent 独立超时钳位      | 缺少全局异常统一处理机制                 | 增加全局 Exception Handler 和降级策略                |
| **日志和监控**       | 使用 Python logging                              | 缺少结构化日志、链路追踪、指标采集       | 接入 ELK/Loki + Prometheus/Grafana                   |
| **权限控制**         | Bot 支持管理员用户列表；Discord Webhook 签名验证 | 缺少细粒度 RBAC、API Key 管理            | 增加 API 认证（JWT/API Key）和用户角色管理           |
| **工具调用安全**     | 数据源只读（行情/新闻），不涉及交易操作          | 工具调用缺少审计日志                     | 增加工具调用审计和异常行为告警                       |
| **RAG 没命中时处理** | 项目无 RAG，不适用                               | 无语义检索能力                           | 如需增强，可引入 ChromaDB 等向量数据库做新闻语义检索 |
| **扩展性**           | 支持自定义策略 YAML；支持多数据源插件式扩展      | 缺少标准化的 Tool 注册机制               | 抽象统一的 Tool/Plugin 接口，支持热加载              |
| **企业级补充**       | Docker 部署；多渠道推送                          | 缺少高可用、水平扩展、配置中心、灰度发布 | 增加 Redis 缓存、消息队列、K8s 部署支持              |

## 6. 总结与学习价值

### 6.1 核心亮点

1. **轻量级自研 Agent 框架**：不依赖 LangChain/LangGraph，用最少的抽象实现多 Agent 编排
2. **多数据源智能降级**：10+ 数据源按优先级自动切换，腾讯日 K 作为最终兜底
3. **多市场覆盖**：A 股、港股、美股、日股、韩股、台股、ETF
4. **多渠道推送**：企业微信、飞书、Telegram、Discord、Slack、邮箱
5. **零成本定时运行**：GitHub Actions 免费跑
6. **MCP + Skills 双协议支持**：既可作为 MCP Server，也可作为 Claude Code Skill

### 6.2 不足

1. **无 RAG 能力**：缺少向量检索，新闻语义搜索能力弱
2. **无 LangChain/LangGraph 生态**：自研框架虽然轻量，但缺少生态工具支持
3. **缺少企业级可观测性**：日志、监控、链路追踪不完善
4. **无持久化存储方案**：历史数据管理能力有限

### 6.3 值得学习的设计

1. **策略模式管理多数据源**：`data_provider/` 的设计优雅，优先级降级逻辑清晰
2. **命令分发器模式**：`bot/dispatcher.py` 实现了可扩展的命令注册与分发
3. **频率限制器**：基于滑动窗口的 RateLimiter 实现
4. **多 Agent 编排 Pipeline**：清晰的阶段划分（Input Prep → Risk → Finalization）
5. **Prompt 动态注入**：根据市场类型动态调整 Prompt

### 6.4 不适合直接照搬的设计

1. **自研 Agent 框架**：如果团队对 LangChain 更熟悉，建议直接用 LangGraph，生态更完善
2. **无 RAG 的设计**：对于需要语义检索的场景，应引入向量数据库
3. **Prompt 硬编码**：当前 Prompt 管理方式不够灵活，建议使用 Prompt 模板管理工具

### 6.5 二次开发优先改造建议

1. **引入向量数据库**：增加新闻/公告的语义检索能力（ChromaDB + Embedding）
2. **完善可观测性**：接入 OpenTelemetry + Prometheus + Grafana
3. **抽象 Tool 注册机制**：标准化工具定义和调用接口
4. **增加 API 认证**：JWT/API Key 管理，支持多用户隔离
5. **引入消息队列**：异步处理分析任务，提升并发能力

### 6.6 面试表达

> **项目背景**：`daily_stock_analysis` 是一个 LLM 驱动的多市场股票智能分析系统，覆盖 A 股、港股、美股等 6 个市场。它解决了个人投资者每日复盘效率低的问题——通过 AI 自动聚合行情、新闻、公告，生成结构化的决策仪表盘并推送到企业微信、飞书等渠道。
>
> **技术架构**：后端使用 FastAPI 提供 REST API，前端是 React，数据层通过 `data_provider/` 策略模式管理 10+ 数据源（AkShare、Tushare、YFinance 等），支持优先级自动降级。部署支持 Docker 和 GitHub Actions 零成本定时运行。
>
> **Agent 工作流**：项目采用自研的多 Agent 编排 Pipeline，包含 TechnicalAgent、IntelAgent、RiskAgent、DecisionAgent、PortfolioAgent、SkillAgent 六个子 Agent。流程分为 Input Preparation → Risk Application → Post-Risk Finalization 三个阶段。各子 Agent 独立分析后，由 LLM 综合生成最终报告。
>
> **Tools / RAG / MCP / Skills**：项目不使用 LangChain/LangGraph，而是自研轻量级框架。Tools 层是 `data_provider/` 中的各数据源 Fetcher。项目**未使用 RAG**，没有向量数据库。但支持 MCP 协议（暴露 14 个工具）和 Claude Code Skills。
>
> **项目亮点**：一是多数据源智能降级，保证高可用；二是零成本定时运行（GitHub Actions）；三是多市场 + 多渠道推送的完整闭环。
>
> **企业级落地思考**：如果要用于企业场景，还需要补充 RAG 能力（引入 ChromaDB 做新闻语义检索）、完善可观测性（接入 Prometheus/Grafana）、增加 API 认证和权限控制、引入消息队列做异步任务处理。