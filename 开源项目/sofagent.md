# sofagent 项目系统分析报告

## 1. 项目定位

### 1.1 这个项目是做什么的？

sofagent 是一个 **FDE（Field Delivery Engineer）Agent**——一个 AI Agent 行为审计与治理引擎。它的核心工作方式是：像 git hook 一样，在每次 AI 生成的 commit 进入仓库之前，自动检查 Agent 是否越界、泄漏密钥或盲目修改。

具体来说，它做三件事：

- **梳理工作流**：把能自动化的环节变成 AI 节点，部署后 7×24 自己跑
- **审计每次变更**：AI 每次干活都自动检查（越界就拦、出事能回滚）
- **经验自动沉淀**：越用越好，越用越懂业务

### 1.2 它解决了什么业务问题？

核心问题：**AI 越能干，企业越不敢放手**。大模型和 Agent 平台提供了“水”（智力）和“河床”（调度能力），但水是原水，企业不敢直接喝。sofagent 解决的是 **“让 AI 从能用变成敢用”** 的问题。

具体场景包括：

- Agent 写错代码、泄漏机密、改乱文件，没人知道
- 换了 Agent 平台或模型，治理规则失效
- 每次踩过的坑，下次 Agent 还踩
- 出了事不知道谁负责、能不能回滚

### 1.3 目标用户是谁？

| 用户类型                  | 使用场景                                             |
| :------------------------ | :--------------------------------------------------- |
| **企业用户/业务负责人**   | 用 FDE 方法论管 Agent，零代码、零依赖                |
| **开发者/技术团队**       | 装审计引擎（git hook），给团队装护栏                 |
| **现场交付工程师（FDE）** | 进场梳理工作流、部署 AI 节点、离场后治理规则持续生效 |
| **个人开发者**            | 用 sofagent Lite 快速建立 Agent 安全边界             |

### 1.4 它属于哪一类 Agent？

**工作流 Agent + 审计 Agent 的混合体**，更准确地说是一个 **Agent 治理基础设施**。

它不是客服 Agent、代码 Agent（虽然包含代码审计能力）、数据分析 Agent 或浏览器 Agent。它属于 **“Agent 的 Agent”**——管理和治理其他 Agent 行为的元 Agent。

### 1.5 为什么这个项目需要用 Agent，而不是普通应用？

普通应用（如 pre-commit、gitleaks）只能检查“代码写得好不好”——lint、格式、密钥扫描。但 AI Agent 特有的失败模式远不止这些：

- **越界编辑**：Agent 改了任务范围外的文件
- **盲改**：Agent 不读代码就直接改
- **注入攻击**：Agent 执行了命令注入
- **数据外传**：Agent 用 curl 把敏感数据传出去
- **持久化后门**：Agent 写 LaunchAgent/systemd 后门

这些行为是 **AI 自主决策的结果**，不是代码质量问题。普通工具查不出来，必须有一个 **能理解 Agent 行为模式、动态审计、持续进化的 Agent 系统**。这就是 sofagent 必须用 Agent 而非普通应用的原因。

## 2. 业务流程

### 2.1 用户输入什么？

用户**不需要直接与 sofagent 对话**。sofagent 是被动触发的治理层——用户在 AI Agent 工具（如 WorkBuddy、Claude Code、Codex）中正常对话、让 AI 干活，sofagent 在后台自动拦截审计。

### 2.2 系统输出什么？

- **合规拦截**：违规 commit 被阻止，显示具体违规规则（如“A1 敏感文件被提交”）
- **审计报告**：每次审计后生成记录，保存在 `~/.sofagent/data/`
- **快照**：每次变更自动 git snapshot，支持一键回滚
- **Dashboard**：网页版/终端版可视化面板，显示 AI 在干什么

### 2.3 从用户输入到最终结果，中间经历了哪些步骤？

| 步骤 | 做了什么                                                | 目的         | 涉及模块                        |
| :--- | :------------------------------------------------------ | :----------- | :------------------------------ |
| 1    | 用户在 Agent 平台（WorkBuddy/Claude Code）发起对话/任务 | 触发 AI 干活 | 外部 Agent 平台                 |
| 2    | Agent 执行任务，产生代码/文件变更                       | AI 干活      | 外部 Agent 平台                 |
| 3    | 用户执行 `git commit`                                   | 提交变更     | git                             |
| 4    | sofagent 的 git hook（commit-msg/post-commit）自动触发  | 拦截点       | `engine/audit`                  |
| 5    | 审计引擎解析 git diff，对照 24 条规则逐条检查           | 行为审计     | `engine/audit` + `engine/rules` |
| 6    | 违规 → 拦截 commit + 记录快照 + 推送通知                | 拦截与回滚   | `engine/audit` + 回溯引擎       |
| 7    | 合规 → commit 通过，自动 git snapshot                   | 存档         | 回溯引擎                        |
| 8    | think.md 自动写入反思教训                               | 经验沉淀     | `engine/think` + 进化引擎       |
| 9    | daemon 后台持续监控文件变更                             | 持续审计     | `engine/daemon`                 |

### 2.4 具体例子：用户让 AI 修改配置文件

**用户**：在 Claude Code 中说“帮我改一下 nginx.conf”

**AI**：生成修改，用户执行 `git add nginx.conf && git commit -m "update nginx"`

**sofagent 介入**：

1. git hook 触发，审计引擎读取 git diff
2. 规则 A3“越界编辑”检测到：用户任务说的是改 nginx.conf，但 AI 同时修改了 `.env`
3. 审计引擎判定违规 → 拦截 commit
4. 自动 git snapshot 存档
5. 用户收到通知：“⛔ A3 越界编辑：AI 修改了任务范围外的文件 .env”
6. 用户可执行 `sofagent-audit --revert <sha>` 一键回滚
7. think.md 自动记录：“今天改 nginx.conf 时差点把 .env 也改了，下次要限定文件范围”

## 3. 技术架构

### 3.1 前端、后端、Agent 服务分别用什么技术？

| 层级           | 技术栈                                                       |
| :------------- | :----------------------------------------------------------- |
| **前端**       | HTML Dashboard（静态）+ Bash 终端面板                        |
| **后端/引擎**  | **Node.js ≥ 18** + TypeScript，Monorepo 架构（npm workspaces） |
| **Agent 编排** | LangGraph（`createReactAgent`）                              |
| **LLM 适配**   | `@langchain/anthropic` 作为可选依赖，运行时动态加载          |
| **审计引擎**   | 纯 git-diff 解析（自研，非第三方包），**零 token 消耗**      |
| **MCP Server** | JSON-RPC 2.0                                                 |

### 3.2 是否使用 LangChain、LangGraph、Deep Agents、GraphRAG、Harness Engineering、Loop Engineering、Function Calling、MCP 等技术？

| 技术                    | 是否使用 | 说明                                                         |
| :---------------------- | :------- | :----------------------------------------------------------- |
| **LangChain**           | ✅ 部分   | `@langchain/anthropic` 作为可选依赖，用于 Anthropic 模型适配 |
| **LangGraph**           | ✅ 是     | 编排引擎用 `createReactAgent` 组装节点流转                   |
| **Deep Agents**         | ❌ 未使用 | 项目未采用 LangChain Deep Agents 框架                        |
| **GraphRAG**            | ❌ 未使用 | 无 GraphRAG 实现                                             |
| **Harness Engineering** | ✅ 核心   | “约束底座”本质就是 Harness 中间件——在 Agent 启动时注入行为约束规则 |
| **Loop Engineering**    | ✅ 是     | FORGE LOOP 流水线（plan→engineer→audit→review→confirm）      |
| **Function Calling**    | ✅ 是     | 编排引擎通过 LangGraph 的 React Agent 实现工具调用           |
| **MCP**                 | ✅ 是     | `@sofagent/mcp` 包提供 MCP Server（JSON-RPC 2.0）            |

### 3.3 是否使用数据库或向量数据库？

**否**。sofagent 的数据存储完全是**文件系统**：

| 数据类型     | 存储位置                         | 格式          |
| :----------- | :------------------------------- | :------------ |
| 审计数据     | `~/.sofagent/data/`              | Markdown 明文 |
| 审计历史     | `~/.sofagent/data/history.jsonl` |               |
| 反思日志     | `think.md`                       | Markdown      |
| 知识库       | `knowledge/*.md`                 | Markdown      |
| A/B 调度状态 | `ab-scheduler-state.json`        |               |
| 审计指标     | `ab-history.jsonl`               |               |

**无向量数据库**，知识检索采用简单的**按 mtime 排序取 Top-N** 的朴素方案。

### 3.4 项目的核心目录分别负责什么？

| 模块/目录              | 作用                                                         | 关键代码                                         | 是否重点阅读 |
| :--------------------- | :----------------------------------------------------------- | :----------------------------------------------- | :----------- |
| `engine/harness/`      | **约束底座**：四层加载链（SKILL.md→fde.md→think.md→knowledge/），生成约束 prompt | `src/index.ts` 的 `buildConstrainedSystemPrompt` | ✅ 是         |
| `engine/audit/`        | **审计引擎**：24 条规则（17 默认+7 扩展），git-diff 解析，违规拦截 | `src/` 下的审计核心逻辑                          | ✅ 是         |
| `engine/rules/`        | **规则定义**：24 条审计规则的注册表、严重度、判定逻辑        | 规则 schema                                      | ✅ 是         |
| `engine/orchestrator/` | **编排引擎**：多 Agent 协作、任务拆解、A/B 调度器            | `ab-scheduler.ts` 四阶段状态机                   | ✅ 是         |
| `engine/daemon/`       | **守护进程**：文件监控、定时巡检、后台常驻                   | `src/cli.ts`                                     | 是           |
| `engine/mcp/`          | **MCP Server**：JSON-RPC 2.0 协议实现                        | `mcp-server.ts`（71KB）                          | 是           |
| `engine/core/`         | **运行时诊断**：doctor/verify 等工具                         | —                                                | 否           |
| `engine/think/`        | **反思引擎**：自动生成 think.md 教训                         | —                                                | 否           |
| `engine/eval/`         | **质量评估**：量化指标、评分逻辑                             | —                                                | 否           |
| `engine/skillopt/`     | **Skill 优化**：失败模式聚类 → 自动触发 Skill 优化           | —                                                | 否           |
| `engine/ab-test/`      | **A/B 测试**：Skill 文件对比评估                             | —                                                | 否           |
| `SKILL/`               | **Agent 定义**：AGENTS.md、SKILL.md 宪法层                   | SKILL.md（4 底线+7 铁律）                        | ✅ 是         |
| `FDE/`                 | **FDE 方法论**：企业 AI 治理方法论、四阶段十二步             | GUIDE.md（39KB）                                 | 是           |
| `FORGE/`               | **自迭代工具链**：LOOP 流水线（内部开发用）                  | —                                                | 否           |
| `tools/`               | **工具脚本**：Dashboard、测试工具等                          | `sofagent-dashboard.sh`                          | 否           |

### 3.5 请求入口、Agent 核心代码、Prompt、Tools、RAG 逻辑分别在哪里？

| 组件               | 位置                                                         | 说明                                                         |
| :----------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **请求入口**       | `engine/audit/src/`（CLI 入口 `sofagent-audit`）+ `engine/daemon/src/cli.ts` | CLI 命令 + git hook 触发                                     |
| **Agent 核心代码** | `engine/orchestrator/src/`（编排引擎）+ `SKILL/agents/`（Agent 定义） | LangGraph `createReactAgent` 组装                            |
| **Prompt 加载**    | `engine/harness/src/index.ts` 的 `buildConstrainedSystemPrompt` | 四层加载链：SKILL.md → fde.md → think.md → knowledge/        |
| **Tools**          | `engine/orchestrator/` 内置 6 个工具：read/write/edit/bash/search/test | 通过 LangGraph 工具调用机制                                  |
| **RAG 逻辑**       | `engine/harness/src/index.ts` 的 `listKnowledgeTopN`         | 朴素实现：按 mtime 排序取前 N 个 .md 文件，每篇截取前 2000 字符 |
| **审计规则**       | `engine/rules/` + `engine/audit/README.md`                   | 24 条规则注册表                                              |

## 4. Agent 核心设计

### 4.1 Agent 的角色是什么？

sofagent 本身是一个 **“Agent 的 Agent”** ——它不直接服务于终端用户，而是**治理和管理其他 Agent 的行为**。它的角色是：

- **审计员**：检查 AI Agent 每次变更是否合规
- **守门员**：在 commit 前拦截违规行为
- **教练**：通过 think.md 让 Agent 从错误中学习
- **管家**：后台 daemon 持续监控，7×24 运行

### 4.2 Agent 如何理解用户任务？

sofagent **不直接理解用户任务**。任务理解由上层的 Agent 平台（WorkBuddy、Claude Code 等）完成。sofagent 通过在 **Agent 启动时注入约束 prompt**（四层加载链）来“理解”任务的边界：

1. **SKILL.md（宪法层）** ：4 底线 + 7 铁律，不可改
2. **fde.md（规范层）** ：企业专属规则，可改
3. **think.md（反思层）** ：历史踩坑教训，自动生成
4. **knowledge/（知识层）** ：持续积累的业务知识

### 4.3 Agent 是否会进行任务拆解或规划？

**是**。编排引擎通过 **LangGraph `createReactAgent`** 实现任务拆解：

- 把任务描述变成编排方案 YAML
- 支持多 SubAgent **串行**编排（当前是串行状态机，非并行 DAG）
- 每个节点有 checkpoint，支持中断恢复

FORGE LOOP 更是典型的任务拆解+多 Agent 协作流水线：

text

```
plan（规划）→ engineer（写代码）→ audit（审计）→ review（审查）→ confirm（确认）
```



### 4.4 Agent 如何决定是否调用工具？

通过 **LangGraph 的 React Agent 机制**：

- Agent 在 `createReactAgent` 中注册了 6 个内置工具（read/write/edit/bash/search/test）
- Agent 自主决定调用哪些工具、按什么顺序调用
- **ToolGate 事前拦截**：在工具调用前进行安全校验

### 4.5 项目中有哪些 tools？

| 工具                             | 来源                   | 说明                                        |
| :------------------------------- | :--------------------- | :------------------------------------------ |
| read/write/edit/bash/search/test | `engine/orchestrator/` | 6 个内置工具，通过 LangGraph 调用           |
| 审计规则（24 条）                | `engine/rules/`        | 不是 LLM 工具，是硬编码的 git-diff 检查规则 |
| MCP Server                       | `engine/mcp/`          | 通过 JSON-RPC 2.0 暴露工具                  |

**注意**：审计引擎的核心规则（24 条）**不是 LLM 工具**，而是**硬编码的 git-diff 检查**，不消耗 token。

### 4.6 是否使用 RAG、MCP 或 Skills？

| 能力       | 是否使用 | 实现方式                                                     |
| :--------- | :------- | :----------------------------------------------------------- |
| **RAG**    | ⚠️ 极简版 | `knowledge/` 目录按 mtime 排序取前 N 个 .md，每篇截取 2000 字符。**无向量检索、无 embedding** |
| **MCP**    | ✅ 是     | `@sofagent/mcp` 包提供 JSON-RPC 2.0 Server                   |
| **Skills** | ✅ 是     | `SKILL/` 目录定义 Agent Skill（SKILL.md、AGENTS.md）；sofagent-loop 是 Openclaw Skills |

### 4.7 Agent 如何生成最终结果？

sofagent 的“最终结果”不是回答用户问题，而是**审计决策**：

1. **git hook 触发** → 审计引擎读取 git diff
2. **规则匹配** → 24 条规则逐条检查（16 条纯 git-diff + 4 条混合 + 1 条文件系统）
3. **判定** → 违规/合规
4. **输出**：
   - 违规 → 拦截 commit + 记录快照 + 推送通知
   - 合规 → 放行 + 自动快照
5. **think.md 写入** → 自动记录教训

整个过程 **不调用 LLM**（审计引擎核心规则零 token）。

### 4.8 是否支持多轮对话、记忆或上下文管理？

| 能力           | 是否支持     | 说明                                                         |
| :------------- | :----------- | :----------------------------------------------------------- |
| **多轮对话**   | ❌ 不直接支持 | sofagent 不面向终端用户对话，通过 git hook 触发，是**事件驱动**而非对话驱动 |
| **记忆**       | ✅ 是         | `think.md` 自动记录教训；`knowledge/` 积累知识；审计历史 `history.jsonl` |
| **上下文管理** | ✅ 是         | 四层加载链在 Agent 启动时注入约束 prompt；编排引擎支持 checkpoint 中断恢复 |

## 5. 工程化与可落地性

| 维度                 | 当前实现                                                     | 不足                                                         | 改进建议                                                     |
| :------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **异常处理**         | 审计引擎有完整的测试覆盖（1441 测试/12 包）；文件读取有 try-catch 静默跳过 | 缺少统一的全局异常处理机制；部分异常静默吞掉                 | 引入结构化日志 + 统一错误码；异常上报到监控系统              |
| **日志和监控**       | daemon 每天/周/月自动生成 Markdown 审计报告；Dashboard 实时显示状态 | 无结构化日志（JSON 格式）；无指标暴露（Prometheus）；无告警配置 | 增加 JSON 格式日志；暴露 Prometheus metrics；配置告警规则    |
| **权限控制**         | 数据目录 `~/.sofagent/` 有权限加固；建议生产环境用 APFS/LUKS 加密 | 无 RBAC；无多用户隔离；无操作审计（谁做了什么）              | 增加用户认证和角色管理；操作日志留存                         |
| **工具调用安全**     | ToolGate 事前拦截；24 条规则覆盖注入攻击、数据外传、后门等   | `git commit --no-verify` 可绕过本地 hook；明文存储审计数据   | CI/CD 侧加 `sofagent-audit --diff` 兜底；v1.4.0 引入静态加密 |
| **RAG 没命中时处理** | 知识库未命中时静默跳过（返回空数组）                         | 无 fallback 机制；无“未找到知识”的提示                       | 增加知识库命中率监控；未命中时记录并告警                     |
| **扩展新工具/场景**  | Monorepo 架构，模块可独立扩展；新规则可在 `engine/rules/` 注册；Skill 可自定义 | 扩展新工具需要改核心代码；无插件机制                         | 设计插件系统；工具通过配置文件注册                           |
| **测试覆盖**         | 1441 测试 / 12 包；A/B 调度器全链路 mock 测试                | 部分测试在特定环境预期失败                                   | 修复环境依赖的测试；增加集成测试                             |
| **部署**             | 支持全量安装、仅底座安装、npx 零安装；30 秒快速开始          | Windows 仅实验性；依赖 Node.js ≥ 18 + git                    | 完善 Windows 支持；容器化部署                                |

## 6. 总结与学习价值

### 6.1 核心亮点

1. **零 token 审计**：16 条核心规则纯 git-diff 解析，不调用 LLM，不消耗额度
2. **“一底座·四引擎”架构**：约束底座 + 审计/回溯/编排/进化四引擎，覆盖 Agent 全生命周期
3. **四层约束加载链**：SKILL.md（宪法）→ fde.md（规范）→ think.md（反思）→ knowledge/（知识），在 Agent 启动时注入行为底线
4. **git hook 原生集成**：像 git hook 一样工作，开发者零学习成本
5. **A/B 自动调度器**：真实任务驱动的编排拆解策略自动优化
6. **自迭代开发循环（FORGE LOOP）** ：plan→engineer→audit→review→confirm，Agent 自己写代码、自己审计、自己审查

### 6.2 不足

1. **无向量数据库**：知识检索采用简单的按时间排序，非语义检索
2. **明文存储**：审计数据以 Markdown 明文存储，v1.4.0 才计划引入加密
3. **hook 可绕过**：`git commit --no-verify` 可绕过本地 hook
4. **编排引擎仅串行**：当前是串行状态机，非并行 DAG 调度
5. **进化引擎实验性**：Dream Cycle 知识回灌当前为内存态队列，重启即丢
6. **Windows 支持不完善**

### 6.3 哪些设计值得学习？

1. **Harness 中间件模式**：在 Agent 启动时注入约束，而非事后补救——这是“预防优于治疗”的经典设计
2. **四层加载链**：宪法→规范→反思→知识，分层清晰，职责明确
3. **零 token 审计**：能不用 LLM 就不用 LLM——这是工程化 Agent 的核心原则
4. **git 作为审计基础设施**：利用 git diff 做行为审计，巧妙且轻量
5. **A/B 自动调度**：真实任务驱动的策略优化，而非离线评估——更贴合实际业务
6. **Monorepo 模块化**：12 个 workspace 独立可安装、可测试

### 6.4 哪些设计不适合直接照搬？

1. **无向量数据库的“RAG”** ：按 mtime 排序取知识，不适合需要语义检索的场景
2. **明文数据存储**：企业合规场景必须加密
3. **文件系统作为唯一存储**：大规模多用户场景需要数据库
4. **依赖 git 作为审计基础设施**：非 git 项目不适用
5. **本地部署优先**：SaaS 多租户场景需要改造

### 6.5 如果基于它二次开发，优先改什么？

1. **引入向量数据库**：替换朴素的 `knowledge/` 检索为真正的 RAG
2. **数据加密**：在 v1.4.0 之前自行引入 age 加密
3. **CI/CD 集成**：在 CI pipeline 侧加 `sofagent-audit --diff` 兜底
4. **并行 DAG 调度**：将编排引擎从串行改为并行
5. **结构化日志 + 监控**：增加 Prometheus metrics 和告警
6. **插件系统**：让工具扩展无需改核心代码

### 6.6 面试表达

> **项目背景**：
>
> sofagent 是一个 FDE（Field Delivery Engineer）Agent，本质上是一个 **AI Agent 行为审计与治理引擎**。它解决的核心问题是：AI 越能干，企业越不敢放手——AI 写错代码、泄漏密钥、改乱文件，出了事不知道谁负责、能不能回滚。sofagent 像 git hook 一样工作，在每次 AI 生成的 commit 进入仓库之前检查 Agent 是否越界、泄漏密钥或盲目修改。
>
> **技术架构**：
>
> 项目采用 **Node.js + TypeScript Monorepo** 架构，12 个 npm workspaces。核心是 **“一底座·四引擎”** ——约束底座在 Agent 启动时注入行为底线，审计引擎用 24 条规则拦截违规变更（16 条纯 git-diff，**零 token 消耗**），回溯引擎自动 git snapshot 支持一键回滚，编排引擎用 **LangGraph `createReactAgent`** 做任务拆解和多 Agent 串行编排，进化引擎通过 think.md 让 Agent 从错误中学习。
>
> **Agent 工作流**：
>
> 用户在 Agent 平台（如 Claude Code）中正常对话，AI 产生代码变更后用户执行 `git commit`，sofagent 的 git hook 自动触发。审计引擎解析 git diff，对照 24 条规则逐条检查——违规则拦截 commit、自动快照、推送通知；合规则放行并存档。每次审计后自动写 think.md 反思教训，让 Agent 下次不再犯同样的错。
>
> **Tools / RAG / MCP / Skills 使用情况**：
>
> - **Tools**：编排引擎内置 6 个工具（read/write/edit/bash/search/test），通过 LangGraph 工具调用机制执行
> - **RAG**：极简版——从 `knowledge/` 目录按修改时间排序取前 N 个 .md 文件，每篇截取 2000 字符注入 prompt
> - **MCP**：`@sofagent/mcp` 包提供 JSON-RPC 2.0 Server
> - **Skills**：`SKILL/` 目录定义 Agent Skill（SKILL.md 含 4 底线+7 铁律）
>
> **项目亮点**：
>
> 一是 **零 token 审计**——16 条核心规则纯 git-diff 解析，不消耗 LLM 额度；二是 **四层约束加载链**——宪法→规范→反思→知识，分层清晰，在 Agent 启动时注入而非事后补救；三是 **Harness 中间件模式**——不是造更聪明的模型，而是给已有的聪明加一套闸门；四是 **自迭代开发循环**——FORGE LOOP 让 Agent 自己写代码、自己审计、自己审查。
>
> **企业级落地思考**：
>
> 当前版本适合中小团队快速建立 AI 治理能力，但企业级落地还需补强：**数据加密**（当前明文存储，v1.4.0 才计划引入）；**hook 可绕过问题**（需在 CI/CD 侧加 `sofagent-audit --diff` 兜底）；**知识检索**（当前无向量数据库，建议引入真正的 RAG）；**监控告警**（需增加结构化日志和 Prometheus metrics）；**权限管理**（需增加 RBAC 和多租户隔离）。总体而言，sofagent 的架构思路极具参考价值，但直接生产使用需要根据企业安全合规要求做针对性加固。