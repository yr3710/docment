# 1. Skills 工作原理
前面我们讲解了 Agent Skills 的基础。本质上，它类似于一个增强版本的“Prompt 提示词”，但与之不同的是，Skills 不仅支持按需加载 References 文件，还支持执行脚本等一系列操作。

“按需”就是“按照需要”的意思——需要的时候再读取或使用，不需要的时候就不进行操作。其实“按需”这个词语在 Agent Skills 中有一个很专业 & 唬人的词语，叫做“渐进式披露（Progressive Disclosure）”。
![skill加载.png](reference/skill/skill%E5%8A%A0%E8%BD%BD.png)

# 2. 渐进式披露
## 2.1 渐进式披露的三个过程
渐进式披露分为三个过程：
1. 发现阶段：当对话开始时，Agent（例如：Codex） 会扫描所有可用的技能文件夹，只读取每个skill的元信息。这是最轻量的信息，足以让代理判断某个技能是否可能与当前任务相关。
2. 激活阶段：当 Agent 判断某个技能的描述与用户请求相匹配时，会将完整的 SKILL.md 文件内容加载到上下文中。这时代理会读取完整的指令和说明。
3. 执行阶段（Execution）：Agent 按照 SKILL.md 中的指令执行任务。在这个过程中，代理可能会调用技能附带的脚本、读取参考资料或使用其他资源。
### 2.1.1 发现阶段（Discovery）
当对话开始时，Agent（例如：Codex） 会扫描所有可用的技能文件夹，只读取每个skill的元信息。这是最轻量的信息，足以让代理判断某个技能是否可能与当前任务相关。
![skill激活.png](reference/skill/skill%E6%BF%80%E6%B4%BB.png)

### 2.1.2 激活阶段（Activation）
当 Agent 判断某个技能的描述与用户请求相匹配时，会将完整的 SKILL.md 文件内容加载到上下文中。这时代理会读取完整的指令和说明。
### 2.1.3 执行阶段（Execution）
Agent 按照 SKILL.md 中的指令执行任务。在这个过程中，代理可能会调用技能附带的脚本、读取参考资料或使用其他资源。
![skill执行.png](reference/skill/skill%E6%89%A7%E8%A1%8C.png)

## 2.2 渐进式披露的优势
![渐进式披露.png](reference/skill/%E6%B8%90%E8%BF%9B%E5%BC%8F%E6%8A%AB%E9%9C%B2.png)
# 3. 天气查询 Skill
![天气查询skill.png](reference/skill/%E5%A4%A9%E6%B0%94%E6%9F%A5%E8%AF%A2skill.png)
```markdown
      ---
      name: weather-query
      description: >
        查询指定城市的实时天气信息。
        当用户询问天气、温度、天气状况、空气质量等信息时使用该技能。
      ---
      
      # Weather Query Skill
      
      ## Instructions
      
      你是一个天气查询助手。
      
      当用户需要查询天气时：
      
      1. 提取用户想查询的城市名称。
      2. 调用 `scripts/get_weather.py` 脚本获取天气数据。
      3. 根据脚本返回结果整理成自然语言回复。
      4. 如果用户没有提供城市，需要询问用户具体城市。
      ---
      
      ## Script
      天气查询脚本：scripts/get_weather.py
      
      调用方式：
      
      ```bash
      python ./scripts/get_weather.py <city>
      ```
      ---
      
      ## Output Format
      
      请按照以下格式回复：
      城市：xxx
      
      天气：xxx
      温度：xxx
      湿度：xxx
      ---
      
      ## Notes
      不要编造天气数据。
      必须通过脚本获取天气信息。
      如果查询失败，需要明确告诉用户。

```
```python
import sys
import json
import random


def get_weather(city: str):
    weather_list = [
        "晴",
        "多云",
        "小雨",
        "阴"
    ]

    return {
        "city": city,
        "temperature": f"{random.randint(100, 200)}℃",
        "weather": random.choice(weather_list),
        "humidity": f"{random.randint(30, 80)}%"
    }


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(json.dumps({
            "error": "please input the city name"
        }, ensure_ascii=False))
        sys.exit(1)

    city = sys.argv[1]

    result = get_weather(city)

    print(
        json.dumps(
            result,
            ensure_ascii=False,
            indent=2
        )
    )
```

# 4. Function Calling & MCP & Agent SKILLS 的区别
一句话总结：Function Calling 是调用机制，MCP 是连接协议，Agent Skills 是任务经验和工作流程的封装。

---
这三者不在同一个技术层级：
- Function Calling：解决“模型怎样选择并调用一个函数” 
- MCP：解决“不同 AI 客户端怎样标准化连接外部工具和数据” 
- Agent Skills：解决“怎样把一套专业任务流程封装成可复用能力”

## 4.1 Function Calling：告诉模型“有哪些函数可以调用”
Function Calling 会把函数名称、功能描述和参数结构提供给模型。模型根据用户问题，判断是否需要调用函数，并生成结构化的函数调用请求。OpenAI 官方将其定义为模型连接外部系统、访问外部数据和执行操作的一种方式，函数参数通常通过 JSON Schema 描述。

例如，应用向模型提供：
```python
@tool
def get_weather(city: str) -> str:
    """查询指定城市的天气"""
    ... 代码内容
```
用户提问：北京今天的天气怎么样？

模型可能生成：
```json
{
  "name": "get_weather",
  "arguments": {
    "city": "北京"
  }
}
```

需要特别注意：

模型只负责生成调用意图和参数，真正的 Python 函数仍然需要由应用程序执行。

基本流程是：
```
        用户问题
           ↓
        模型判断需要调用工具
           ↓
        模型生成函数名和参数
           ↓
        应用程序执行函数
           ↓
        将函数结果返回给模型
           ↓
        模型生成最终回答

```
因此，Function Calling 更像是模型与程序代码之间的调用接口。


## 4.2 MCP：统一 AI 客户端与工具之间的连接方式
如果只使用 Function Calling，每个 Agent 项目都需要自己编写工具注册、参数转换、连接管理和调用逻辑。

MCP，即 Model Context Protocol，是一种连接 AI 应用和外部系统的开放标准。MCP Server 可以向客户端暴露工具、数据资源和提示词，MCP Client 通过统一协议发现并调用这些能力。

例如，天气服务可以被封装成一个 MCP Server：
```
        Cursor ───────┐
        Cline ────────┼── MCP 协议 ──> Weather MCP Server
        Claude Code ──┤                     └── get_weather
        Codex ────────┘
```

MCP Server 暴露的工具可能是：

```python
        @mcp.tool()
        def get_weather(city: str) -> str:
            return query_weather_api(city)
```

不同 MCP Client 都可以通过统一协议：
1. 获取工具列表； 
2. 查看工具参数； 
3. 调用指定工具； 
4. 获取工具执行结果。 

![mcp 调用流程.png](reference/mcp%20%E8%B0%83%E7%94%A8%E6%B5%81%E7%A8%8B.png)

MCP 官方规范中，Tool 具有唯一名称以及描述参数结构的元数据，可以用于数据库查询、API 调用和计算等外部操作。

因此，MCP 更像 AI 世界中的：

USB-C 接口：统一不同 AI 客户端连接外部工具的方式。

## 4.3  Agent Skills：告诉 Agent“这类任务应该怎么完成”
Agent Skills 不是一种网络协议，也不只是一个函数。

它是一套可以被 Agent 按需加载的专业任务说明书。一个 Skill 通常以文件夹形式存在，核心文件是 SKILL.md，还可以包含脚本、参考资料和资源文件。OpenAI 官方将 Skill 描述为由文件和 SKILL.md 清单组成的版本化能力包，用于固化可复用的任务流程。

例如：
```
        weather_query/
        ├── SKILL.md
        ├── scripts/
        │   └── get_weather.py
        ├── references/
        │   └── weather_api.md
        └── assets/
```

SKILL.md可以定义为:
```
   ---
   name: weather-query
   description: 查询指定城市的实时天气。
   ---
   
   # Instructions
   
   当用户查询天气时：
   
   1. 提取城市名称。
      2. 调用 scripts/get_weather.py。
      3. 检查脚本执行结果。
      4. 按指定格式返回天气信息。
      5. 不得编造天气数据。
```

这里封装的不只是一个 get_weather() 函数，还包括：
-  什么时候触发； 
-  应该先做什么； 
-  应该调用哪个脚本； 
-  如何处理异常； 
-  最终按照什么格式输出； 
-  哪些规则必须遵守。 
Codex 会先读取 Skill 的名称和描述进行能力匹配，只有确定需要使用该 Skill 时，才会加载完整的 SKILL.md 指令，这种方式可以减少不必要的上下文占用。

因此，Agent Skills 更像是给 Agent 准备的：

标准操作流程 SOP + 专业知识 + 可执行脚本。

## 4.4 总结
![skill总结.png](reference/skill/skill%E6%80%BB%E7%BB%93.png)

# 5. Skills 标准目录结构
https://www.runoob.com/skills/skills-dir.html
![skill标准的目录结构.png](reference/skill/skill%E6%A0%87%E5%87%86%E7%9A%84%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.png)

# 6. skill-creator 使用
https://www.runoob.com/claude-code/skill-creator-usage.html

![创建skill的skill.png](reference/skill/%E5%88%9B%E5%BB%BAskill%E7%9A%84skill.png)
