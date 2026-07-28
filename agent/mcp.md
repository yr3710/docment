## **1. Function Calling的流程**
![functioncall流程.png](reference/functioncall%E6%B5%81%E7%A8%8B.png)

## **2. Function Calling的局限性**
![functioncall局限性.png](reference/functioncall%E5%B1%80%E9%99%90%E6%80%A7.png)

Function Calling 在实际使用中存在一个明显的局限性：不同开发者往往各自实现一套功能相同但命名不同的工具函数，导致重复开发和维护成本上升。尽管函数名称和参数命名有所差异，其核心业务逻辑并无本质区别。

### **2.1 解决办法**
解决办法是将天气查询等功能逻辑抽象成独立、可复用的工具模块。开发者只需在各自构建的智能体中调用该模块的接口，即可完成天气查询等操作，从而避免重复实现相似的业务逻辑。
![functioncall解决.png](reference/functioncall%E8%A7%A3%E5%86%B3.png)

### **2.2 MCP 的核心思想**
MCP 的核心思想是：将独立的工具代码封装成一个标准化的服务，Agent 通过网络协议与之通信，而非直接调用本地函数。

MCP: 就是将本地的工具注册为全局的工具！
![mcp逻辑.png](reference/mcp%E9%80%BB%E8%BE%91.png)

### **2.2 MCP Server架构**
![MCP Server架构.png](reference/MCP%20Server%E6%9E%B6%E6%9E%84.png)


### **2.4 MCP Server 代码开发**
**1. 查询天气的 MCP Server 服务**
```python
#!/usr/bin/env python3
"""
天气查询 MCP Server - Mock 版本
"""

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather")

@mcp.tool()
def get_current_weather(city: str) -> str:
    """
    获取指定城市的当前天气
    :param city: 城市地址
    :return: 天气信息
    """
    return f"{city}: 100°C, 风速 0km/h, 天气状况: ☀️ 超级热"

if __name__ == "__main__":
    print("Server Started")
    mcp.run(transport="stdio")
```

**2. 查询数据库的 MCP 工具**
```python
#!/usr/bin/env python3
"""
MySQL MCP Server - Mock版本
"""

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("MySQL")

# Mock数据
TABLES = {
    "users": "id, username, email, created_at",
    "orders": "id, user_id, product, amount, status",
    "products": "id, name, price, stock"
}

@mcp.tool()
def list_tables() -> str:
    """
    列出所有表
    :return: 所有的表名
    """
    return f"📋 数据库表: {', '.join(TABLES.keys())}"

@mcp.tool()
def describe_table(table_name: str) -> str:
    """
    查看表结构
    :param table_name: 表名
    :return: 表的描述信息
    """
    if table_name in TABLES:
        return f"📊 表 {table_name} 的字段: {TABLES[table_name]}"
    return f"❌ 表 {table_name} 不存在"

if __name__ == "__main__":
    print("MCP Server Started")
    mcp.run(transport="stdio")
```

### **2.5 谁来调用 MCP Server 服务**
MCP Server 的服务可以被自行编写的 create_agent 实例调用，也可以被 Cursor、Claude Code、Codex 等外部工具调用。

比如：使用 create_agent 进行 MCP调用的代码：https://langchain-doc.cn/v1/python/langchain/mcp.html#%E4%BD%BF%E7%94%A8-mcp-%E5%B7%A5%E5%85%B7
![MCP Server调用.png](reference/MCP%20Server%E8%B0%83%E7%94%A8.png)

## **3. MCP 调用流程**
**从用户提问到mcp调用到模型回复**
![mcp 调用流程.png](reference/mcp%20%E8%B0%83%E7%94%A8%E6%B5%81%E7%A8%8B.png)

## **4. MCP是什么**
MCP（模型上下文协议，Model Context Protocol）是一个开放标准，目的是为AI应用（特别是大语言模型）提供一个统一的接口（可以时理解为是一种统一的方式），让它们能方便地连接和交互各种外部数据源与工具

MCP就像是AI应用的"USB-C接口"，它提供了一个统一的标准，让AI模型可以：
- 连接到各种数据源（数据库、文件、API等）
- 使用外部工具（搜索、计算、文件操作等）
- 访问资源和上下文信息

### **4.1 MCP架构**
```python
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   AI应用     │ ◄─────► │  MCP协议     │ ◄─────► │  MCP服务器  │
│  (Cline)    │         │  (Stdio/HTTP)│        │  (工具提供者)│
└─────────────┘         └─────────────┘         └─────────────┘
                                                          │
                                                          ▼
                                                  ┌─────────────┐
                                                  │ 外部资源      │
                                                  │ (API/DB/文件)│
                                                  └─────────────┘
```
### **4.2 MCP Server调用流程**
![mcp 调用流程.png](reference/mcp%20%E8%B0%83%E7%94%A8%E6%B5%81%E7%A8%8B.png)

### **4.3 MCP Server创建**
![mcp server创建.png](reference/mcp%20server%E5%88%9B%E5%BB%BA.png)

#### **4.3.1 stdio 标准输入输出**
标准输入/输出（Standard Input/Output，缩写为 stdio） 是一种进程间通信机制。在此模式下，MCP 服务器作为客户端（例如 Cline、Claude Code 等）的子进程被启动，双方通过标准输入流（stdin）与标准输出流（stdout）进行数据交换。该方式适用于本地工具集成及简易配置环境。

---
在Linux、macOS和Windows等操作系统中，标准输入输出是操作系统预留给每个程序的三个“数据通道”。当一个程序（比如你的MCP Server）通过命令行启动时，系统会自动为它打开这三个通道：
1. 标准输入（stdin）：编号为0。程序读取数据的通道，默认来自键盘。
2. 标准输出（stdout）：编号为1。程序输出正常结果的通道，默认显示在终端屏幕。
3. 标准错误（stderr）：编号为2。程序输出错误信息的通道，也默认显示在终端屏幕。
> 之所以把stdout和stderr分开，是为了让程序能区分“正常结果”和“错误信息”，方便父进程或用户做不同的处理。

![标准流.png](reference/%E6%A0%87%E5%87%86%E6%B5%81.png)
在MCP中的实际运作：进程间通信（IPC）

在 MCP Server 的 STDIO 模式 下，标准输入输出的用途发生了关键变化：它不再是人和程序交互，而是两个程序（Client 和 Server）之间进行通信的“管道”。
- 当你启动 Claude Desktop 等客户端时，它会以子进程的方式启动 MCP Server。
- 接着，客户端会接管 MCP Server 的 stdin 和 stdout。这意味着：
  - 客户端通过向 Server 的 stdin 写入数据来发送请求（比如调用工具）。
  - Server通过向自己的 stdout 写入数据来返回响应（比如工具执行结果）。
  - stderr 则保持不变，一般用来输出日志信息，供开发者调试查看，不会干扰正常的MCP协议通信。

![客户端配置mcp.png](reference/%E5%AE%A2%E6%88%B7%E7%AB%AF%E9%85%8D%E7%BD%AEmcp.png)

#### **4.3.2 拦截 MCP 的输入输出**

![MCP通信流程对比.png](reference/MCP%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B%E5%AF%B9%E6%AF%94.png)
中间人代码：
```python
#!/usr/bin/env python3
"""
MCP 中间代理

仅记录：
1. Cline -> MCP Server
2. MCP Server -> Cline
"""

import argparse
import os
import subprocess
import sys
import threading
from datetime import datetime


# 日志文件路径
LOG_FILE = os.path.join(
    os.path.dirname(os.path.realpath(__file__)),
    "mcp_io.log"
)


# =========================
# 参数解析
# =========================

parser = argparse.ArgumentParser(
    description="拦截并记录 Cline 与 MCP Server 之间的输入输出",
    usage="%(prog)s <command> [args...]"
)

parser.add_argument(
    "command",
    nargs=argparse.REMAINDER,
    help="MCP Server 启动命令及其参数"
)

if len(sys.argv) == 1:
    parser.print_help(sys.stderr)
    sys.exit(1)

args = parser.parse_args()

if not args.command:
    print("Error: No command provided.", file=sys.stderr)
    parser.print_help(sys.stderr)
    sys.exit(1)

target_command = args.command


# =========================
# 输入输出转发
# =========================

def forward_and_log_stdin(proxy_stdin, target_stdin, log_file):
    """
    读取 Cline 的标准输入，记录日志，然后转发给 MCP Server。
    """
    try:
        while True:
            line_bytes = proxy_stdin.readline()

            if not line_bytes:
                break

            try:
                line_str = line_bytes.decode("utf-8").strip()
                timestamp = datetime.now().isoformat()

                log_file.write(
                    f"[{timestamp}] "
                    f"Cline -> MCP Server: {line_str}\n"
                )
                log_file.flush()

            except UnicodeDecodeError:
                timestamp = datetime.now().isoformat()

                log_file.write(
                    f"[{timestamp}] "
                    f"Cline -> MCP Server: "
                    f"[Binary data, {len(line_bytes)} bytes]\n"
                )
                log_file.flush()

            target_stdin.write(line_bytes)
            target_stdin.flush()

    except Exception as e:
        try:
            log_file.write(
                f"[{datetime.now().isoformat()}] "
                f"STDIN Forwarding Error: {e}\n"
            )
            log_file.flush()
        except Exception:
            pass

    finally:
        try:
            target_stdin.close()

            log_file.write(
                f"[{datetime.now().isoformat()}] "
                f"--- STDIN stream closed ---\n"
            )
            log_file.flush()

        except Exception:
            pass


def forward_and_log_stdout(
    target_stdout,
    proxy_stdout,
    log_file
):
    """
    读取 MCP Server 的标准输出，记录日志，然后转发给 Cline。
    """
    try:
        while True:
            line_bytes = target_stdout.readline()

            if not line_bytes:
                break

            try:
                line_str = line_bytes.decode("utf-8").strip()
                timestamp = datetime.now().isoformat()

                log_file.write(
                    f"[{timestamp}] "
                    f"MCP Server -> Cline: {line_str}\n"
                )
                log_file.flush()

            except UnicodeDecodeError:
                timestamp = datetime.now().isoformat()

                log_file.write(
                    f"[{timestamp}] "
                    f"MCP Server -> Cline: "
                    f"[Binary data, {len(line_bytes)} bytes]\n"
                )
                log_file.flush()

            proxy_stdout.write(line_bytes)
            proxy_stdout.flush()

    except Exception as e:
        try:
            log_file.write(
                f"[{datetime.now().isoformat()}] "
                f"STDOUT Forwarding Error: {e}\n"
            )
            log_file.flush()
        except Exception:
            pass

    finally:
        try:
            log_file.flush()
        except Exception:
            pass


# =========================
# 主程序
# =========================

process = None
log_f = None
exit_code = 1

try:
    log_f = open(
        LOG_FILE,
        "a",
        encoding="utf-8"
    )

    log_f.write(f"\n{'=' * 80}\n")
    log_f.write(
        f"[{datetime.now().isoformat()}] MCP代理启动\n"
    )
    log_f.write(
        f"[{datetime.now().isoformat()}] "
        f"命令: {' '.join(target_command)}\n"
    )
    log_f.write(f"{'=' * 80}\n\n")
    log_f.flush()

    # 启动真正的 MCP Server
    process = subprocess.Popen(
        target_command,
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,

        # 丢弃 MCP Server 的所有 stderr 信息
        stderr=subprocess.DEVNULL,

        # 无缓冲二进制 I/O
        bufsize=0
    )

    stdin_thread = threading.Thread(
        target=forward_and_log_stdin,
        args=(
            sys.stdin.buffer,
            process.stdin,
            log_f
        ),
        daemon=True
    )

    stdout_thread = threading.Thread(
        target=forward_and_log_stdout,
        args=(
            process.stdout,
            sys.stdout.buffer,
            log_f
        ),
        daemon=True
    )

    stdin_thread.start()
    stdout_thread.start()

    process.wait()
    exit_code = process.returncode

    stdin_thread.join(timeout=1.0)
    stdout_thread.join(timeout=1.0)

except Exception as e:
    print(
        f"MCP代理错误: {e}",
        file=sys.stderr
    )

    if log_f and not log_f.closed:
        try:
            log_f.write(
                f"[{datetime.now().isoformat()}] "
                f"主程序错误: {e}\n"
            )
            log_f.flush()
        except Exception:
            pass

    exit_code = 1

finally:
    if process and process.poll() is None:
        try:
            process.terminate()
            process.wait(timeout=1.0)
        except Exception:
            pass

        if process.poll() is None:
            try:
                process.kill()
            except Exception:
                pass

    if log_f and not log_f.closed:
        try:
            log_f.write(
                f"\n[{datetime.now().isoformat()}] "
                f"MCP代理停止，退出码: {exit_code}\n"
            )
            log_f.write(f"{'=' * 80}\n\n")
            log_f.close()
        except Exception:
            pass

    sys.exit(exit_code)

```
![mcp server配置-01.png](reference/mcp%20server%E9%85%8D%E7%BD%AE-01.png) 
![mcp server配置-02.png](reference/mcp%20server%E9%85%8D%E7%BD%AE-02.png)

中间人代码执行过程：
![中间人代码执行过程.png](reference/%E4%B8%AD%E9%97%B4%E4%BA%BA%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B.png)

![mcp tool调用.png](reference/mcp%20tool%E8%B0%83%E7%94%A8.png)
### **4.3.3 Streamable HTTP & SSE**
MCP 支持用于客户端-服务器通信的不同传输机制：
- stdio：客户端将服务器作为子进程启动，并通过标准输入/输出进行通信。最适合本地工具和简单的设置。
- Streamable HTTP：服务器作为独立进程运行，处理 HTTP 请求。支持远程连接和多个客户端。
- Server-Sent Events (SSE)：Streamable HTTP 的一个变体，针对实时流式通信进行了优化。

### **4.3.3 LangChain 连接MCP Server**
```python
import asyncio

from langchain_community.chat_models import ChatTongyi
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent


async def main():
    # 1. 创建客户端
    client = MultiServerMCPClient(
        {
            "weather": {
                "transport": "stdio",
                "command": "python",
                "args": [
                    "E:\\python\\k-ai-knowledge-2.0\\agent\\mcp_server\\weather.py"
                ]
            }
        }
    )

    # 2. 获取工具（使用 await）
    tools = await client.get_tools()

    # 3. 创建 agent
    agent = create_agent(
        model=ChatTongyi(model="qwen3-max"),
        tools=tools,
        system_prompt="你是一个助手，可以调用工具帮助用户解决问题"
    )

    # 4. 使用 agent
    result = await agent.ainvoke({"messages": [("user", "北京天气怎么样？")]})
    print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

## **5. MCP支持的三种传输类型**
![mcp支持的三种传输类型.png](reference/mcp%E6%94%AF%E6%8C%81%E7%9A%84%E4%B8%89%E7%A7%8D%E4%BC%A0%E8%BE%93%E7%B1%BB%E5%9E%8B.png)


### **5.1 stdio - 标准输入输出**
在 stdio 模式下，MCP Client 会把 MCP Server 作为一个子进程启动：
![stdio通信.png](reference/stdio%E9%80%9A%E4%BF%A1.png)
MCP Server：
-  从 stdin 读取 JSON-RPC 消息； 
-  向 stdout 写入 JSON-RPC 消息； 
-  日志只能写入 stderr； 
-  不能向 stdout 输出普通日志，否则会破坏 MCP 消息流。
![mcp 调用流程.png](reference/mcp%20%E8%B0%83%E7%94%A8%E6%B5%81%E7%A8%8B.png)

#### **5.1.1 stdio 优缺点**
![mcp stdio优缺点.png](reference/mcp%20stdio%E4%BC%98%E7%BC%BA%E7%82%B9.png)

#### **5.1.1 stdio 适用场景**
适合：
```python
Cursor
Claude Desktop
Claude Code
Cline
Codex CLI
本地 Agent
        │
        ▼
本地 MCP Server
```
典型功能：
-  读取本地项目文件； 
-  执行本地 Git 命令； 
-  连接开发机上的 MySQL； 
-  操作本地浏览器； 
-  调用只允许本机访问的服务。

### **5.2 Server-Sent Events (SSE)**
SSE：Server Send Event，SSE（这里的 SSE 指的是旧版 MCP 的 HTTP+SSE transport） 是一种基于 HTTP 的轻量级实时通信技术，允许服务器主动向客户端推送数据。它的核心思想是：客户端发起一个普通 HTTP 请求，服务器不立刻结束响应，而是保持连接打开，并持续发送数据块。

它通常需要两个 Endpoint：

```python
GET  /sse
POST /messages
```
其中：
- GET /sse：建立服务端到客户端的 SSE 长连接。
- POST /messages：客户端向服务端发送 JSON-RPC 消息。
旧版 MCP 规范要求 Server 同时提供这两个 Endpoint；客户端连接 SSE Endpoint 后，Server 首先发送一个 endpoint 事件，告诉客户端后续应该向哪个地址 POST 消息。

### **5.2.1 SSE的通信结构**
```python
MCP Client                                MCP Server
│                                         │
│ ① GET /sse                              │
│ Accept: text/event-stream               │
├────────────────────────────────────────▶│
│                                         │
│◀══════════ 建立 SSE 长连接 ═════════════│
│                                         │
│ ② endpoint 事件                         │
│    告知客户端 POST 地址                  │
│◀════════════════════════════════════════│
│                                         │
│ ③ POST /messages?session_id=xxx         │
│    发送 JSON-RPC 请求                    │
├────────────────────────────────────────▶│
│                                         │
│ ④ HTTP 202 Accepted                     │
│◀────────────────────────────────────────┤
│                                         │
│ ⑤ JSON-RPC 结果通过 SSE 长连接返回       │
│◀════════════════════════════════════════│
```
可以概括为：

1：客户端 → Server：普通 HTTP POST

2：Server → 客户端：SSE 长连接

**第 1 步：客户端建立 SSE 连接**
客户端先向 SSE Endpoint 发起一个 HTTP GET 请求：
```http
GET /sse HTTP/1.1
Host: 127.0.0.1:8000
Accept: text/event-stream
```
server返回：
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```
这次 HTTP Response 不会立即结束，而是保持打开状态：
```
Client ◀════════════ SSE 长连接 ═══════════ Server
```
SSE本质上是单向的，Server 可以持续向 Client 发送数据，而 Client 不能通过这个连接发送数据。Client 需要通过另一个 HTTP POST 请求来发送消息。
```
Server → Client
```
**第 2 步：Server 发送 endpoint 事件**

建立 SSE 连接后，Server 首先通过 SSE 发送一个特殊的 endpoint 事件：
```http
event: endpoint
data: /messages/?session_id=abc123
```
它的作用是告诉客户端：
```
后续 JSON-RPC 消息请发送到：
/messages/?session_id=abc123
```
其中的：
```
session_id=abc123
```
用于将后续 POST 请求和当前 SSE 长连接关联起来。

因此，Server 内部会形成类似映射：
```
session_id=abc123
        │
        ▼
对应某条 SSE 长连接
```

旧版规范明确规定，连接建立后 Server 必须发送 endpoint 事件，其中包含客户端后续发送消息所使用的 URI。

**第 3 步：客户端发送 initialize 请求**

客户端获得 POST 地址后，向该地址发送 JSON-RPC 消息：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
      "name": "demo-client",
      "version": "1.0.0"
    }
  }
}
```

此时 POST 请求只负责：把 JSON-RPC 消息送进 MCP Server。
它通常不会直接在 HTTP Response 中返回 MCP 执行结果。
Server 接收成功后，可以返回：
```http
HTTP/1.1 202 Accepted
Content-Length: 0
```
202 Accepted 的含义是：消息已经被 Server 接收， 但真正的 JSON-RPC Response 会通过 SSE 通道返回。
**第 4 步：Server 通过 SSE 返回 initialize 结果**

Server 处理完 initialize 后，不通过刚才的 POST Response 返回，而是通过之前建立的 /sse 长连接发送：
```http
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{}},"serverInfo":{"name":"database","version":"1.0.0"}}}
```
这里：event: message 表示这是一条普通 MCP 消息。而：data: {...}里面存放 JSON-RPC 消息。

旧版规范规定，Server 消息通过 SSE 的 message 事件发送，JSON-RPC 内容编码在 SSE Event 的 data 字段中。
**第 5 步：客户端发送 initialized 通知**
初始化响应成功后，客户端再通过 POST Endpoint 发送：
```http
POST /messages/?session_id=abc123
Content-Type: application/json
```
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```
这是 Notification，没有 id，所以不要求 JSON-RPC Response。

Server 接收后通常返回：HTTP/1.1 202 Accepted
**第 6 步：客户端查询工具列表**
客户端再次通过 POST Endpoint 发送：
```http
POST /messages/?session_id=abc123
Content-Type: application/json
```
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```
Server 的 HTTP Response 通常仍只是：HTTP/1.1 202 Accepted

真正的工具列表通过 SSE 发送：
```http
event: message
data: {"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"add","description":"计算两个整数之和","inputSchema":{"type":"object","properties":{"a":{"type":"integer"},"b":{"type":"integer"}},"required":["a","b"]}}]}}
```
**第 7 步：客户端调用工具**
客户端发送：

```http
POST /messages/?session_id=abc123
Content-Type: application/json
```
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "add",
    "arguments": {
      "a": 10,
      "b": 20
    }
  }
}
```

Server 接收后返回：HTTP/1.1 202 Accepted

随后通过 SSE 长连接返回：
```http
event: message
data: {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"30"}],"isError":false}}
```

整个过程是：
```
POST 请求负责发送 tools/call
              │
              ▼
MCP Server 执行 add(10, 20)
              │
              ▼
通过 SSE 长连接发送 tools/call Response

```
#### **5.2.2 请求与响应如何关联**
因为 POST 请求和 SSE 返回结果走的是不同连接，所以必须通过 JSON-RPC 的 id 关联。

客户端发送：
```json
{
  "id": 3,
  "method": "tools/call"
}
```
Server 返回：
```json
{
  "id": 3,
  "result": {}
}
```
客户端看到相同的id = 3，就知道这是对应 tools/call 请求的响应。

因此，旧版 HTTP+SSE 中存在两层关联：
```
session_id
    │
    └── 关联 POST Endpoint 与 SSE 连接

JSON-RPC id
    │
    └── 关联具体请求与具体响应
    
```
#### **5.2.3 为什么必须有 session_id**
假设一个 Server 同时连接多个 Client：
```
Client A ──GET /sse──▶ Server
Client B ──GET /sse──▶ Server
Client C ──GET /sse──▶ Server
```

Server 必须知道某个 POST 请求的结果应该发到哪条 SSE 连接。

例如：
```
Client A → session_id=A001
Client B → session_id=B002
Client C → session_id=C003
```
Client A 发送：POST /messages?session_id=A001

Server 执行完成后，就把结果写入：A001 对应的 SSE 连接

而不会发送给 Client B 或 Client C。

#### **5.2.4 连接断开后怎么办**
SSE 是长连接，但网络中断、代理超时或 Server 重启都可能导致连接断开。

典型处理方式是：
```
SSE 连接断开
      │
      ▼
客户端重新 GET /sse
      │
      ▼
Server 建立新的 SSE 会话
      │
      ▼
重新获得 endpoint 和 session_id
```
旧版 transport 的 Session 通常和 SSE 连接绑定得比较紧，因此断线重连和多实例部署会更加复杂。
#### **5.2.5 完整过程图**

```
┌──────────────┐                              ┌──────────────┐
│  MCP Client  │                              │  MCP Server  │
└──────┬───────┘                              └──────┬───────┘
       │                                             │
       │ 1. GET /sse                                 │
       │ Accept: text/event-stream                   │
       ├────────────────────────────────────────────▶│
       │                                             │
       │ 2. SSE 连接建立                             │
       │◀════════════════════════════════════════════│
       │                                             │
       │ 3. endpoint 事件                            │
       │ data: /messages?session_id=abc123           │
       │◀════════════════════════════════════════════│
       │                                             │
       │ 4. POST /messages?session_id=abc123         │
       │ initialize                                  │
       ├────────────────────────────────────────────▶│
       │                                             │
       │ 5. 202 Accepted                             │
       │◀────────────────────────────────────────────┤
       │                                             │
       │ 6. SSE message：InitializeResult            │
       │◀════════════════════════════════════════════│
       │                                             │
       │ 7. POST /messages?session_id=abc123         │
       │ notifications/initialized                   │
       ├────────────────────────────────────────────▶│
       │                                             │
       │ 8. 202 Accepted                             │
       │◀────────────────────────────────────────────┤
       │                                             │
       │ 9. POST /messages?session_id=abc123         │
       │ tools/call                                  │
       ├────────────────────────────────────────────▶│
       │                                             │
       │ 10. 202 Accepted                            │
       │◀────────────────────────────────────────────┤
       │                                             │
       │ 11. SSE message：tools/call result          │
       │◀════════════════════════════════════════════│
       │                                             │
```
##### **5.2.6 传统 HTTP+SSE 的关键特征**
1. 两个 Endpoint
```
GET  /sse
POST /messages
```

2. 两条通信通道
```
POST：Client → Server
SSE： Server → Client
```
3. 请求与响应不在同一次 HTTP 交换中
```
POST Response：通常是 202 Accepted
真正结果：通过 SSE 长连接返回
```
4. SSE 长连接必须提前建立

客户端不能直接先发送 tools/call，而是需要先：
```
GET /sse
    ↓
获取 endpoint
    ↓
再 POST JSON-RPC 请求
```
5. 使用 Session 关联连接
```
session_id
```
用于确定某条 JSON-RPC 响应应该发送到哪一个 Client 的 SSE 流。

传统 HTTP+SSE transport 先通过 GET /sse 建立 Server 到 Client 的长期接收通道，Server 再通过 endpoint 事件告诉 Client 消息提交地址；Client 使用普通 HTTP POST 发送 JSON-RPC 请求，而 Server 将真正的 JSON-RPC 响应通过原来的 SSE 长连接返回。

该 HTTP+SSE transport 属于 MCP 2024-11-05 版本的旧传输方式，当前规范已由 Streamable HTTP 替代，但仍可保留两个旧 Endpoint 以兼容旧客户端


### **5.3 Streamable HTTP**
Streamable HTTP 是 MCP（模型上下文协议，Model Context Protocol）在 2025 年引入的一种全新传输机制，旨在替代原有的 "HTTP+SSE" 方案。它本质上是将标准 HTTP 请求与灵活的流式传输能力结合在了一起。

它的核心在于解决了老方案的一些关键痛点，比如服务端必须维护高消耗的长连接、连接断开后无法恢复，以及复杂的双通道通信等问题。
![streamable http.png](reference/streamable%20http.png)

Streamable HTTP 将 MCP 通信集中到一个 MCP Endpoint：
```
http://127.0.0.1:8000/mcp
```
MCP Client 和 MCP Server 通过 HTTP 传输 JSON-RPC 消息，消息类型如下：
```
initialize
tools/list
tools/call
resources/read
prompts/get
```

Server 可以根据请求选择两种响应：
```
方式一：application/json 
方式二：text/event-stream
```

整体结构：
```
                         POST /mcp
┌────────────┐  JSON-RPC 请求  ┌────────────┐
│ MCP Client │ ───────────────▶ │ MCP Server │
│            │                  │            │
│            │ ◀─────────────── │            │
└────────────┘  JSON 或 SSE 流   └────────────┘
```


#### **5.3.1 服务端代码 - 一次性返回**
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(
    "database",
    host="127.0.0.1",
    port=8000,
    
    # Streamable HTTP 的访问路径     
    streamable_http_path="/mcp",
    
    # 无状态模式：每个请求独立处理，适合生产部署
    stateless_http=True,
    
    # True：普通 JSON 响应     
    # False：允许通过 SSE 流式返回响应
    json_response=True,
)


@mcp.tool()
def add(a: int, b: int) -> int:
    return a + b


if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```
![streamable测试.png](reference/streamable%E6%B5%8B%E8%AF%95.png)
代码测试：
```bash
curl -X POST http://127.0.0.1:8000/mcp \
  -H "Accept: text/event-stream, application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 1
  }'
```
#### **5.3.2 通信过程**

```
MCP Client                            MCP Server
    │                                     │
    │ POST /mcp                           │
    │ JSON-RPC: tools/call                │
    ├────────────────────────────────────▶│
    │                                     │
    │ Content-Type: application/json      │
    │ 一个 JSON-RPC Response              │
    │◀────────────────────────────────────┤
    │                                     │
  HTTP 请求结束
```
每个客户端消息请求就是一次新的 HTTP POST；json_response=True 表示该请求的响应体是一个普通 JSON 对象。

#### **5.3.2 json_response=False 实现 SSE 返回数据**

```python

mcp = FastMCP(
    "database",
    stateless_http=True,
    json_response=False,
)

mcp.run(transport="streamable-http")
```

通信过程：

```
MCP Client                            MCP Server
    │                                     │
    │ POST /mcp                           │
    │ JSON-RPC: tools/call                │
    ├────────────────────────────────────▶│
    │                                     │
    │ Content-Type: text/event-stream     │
    │◀──────── SSE 消息 1 ────────────────┤
    │◀──────── SSE 消息 2 ────────────────┤
    │◀──────── 最终 JSON-RPC Response ─────┤
    │                                     │
  SSE 流结束，HTTP 请求完成
```
此时的响应头是：
```
Content-Type: text/event-stream
```

服务器只是在这个 POST 请求的 HTTP Response 中使用 SSE 格式发送一条或多条消息。当前规范规定，Streamable HTTP 的服务器可以对 JSON-RPC 请求返回普通 JSON，也可以开启 SSE 流；发送最终 JSON-RPC Response 后，通常应结束该 SSE 流。

#### 5.3.3 为什么它不等于传统 SSE transport？
因为两者虽然都使用了 SSE 技术，但通信架构不同。

旧版 SSE transport：

旧版 SSE 必须先建立一条独立的 SSE 长连接：

```
MCP Client                            MCP Server
    │                                     │
    │ GET /sse                            │
    ├────────────────────────────────────▶│
    │◀════════ SSE 长连接 ════════════════│
    │                                     │
    │ POST /messages?session_id=xxx       │
    │ JSON-RPC Request                    │
    ├────────────────────────────────────▶│
    │                                     │
    │ 202 Accepted                        │
    │◀────────────────────────────────────┤
    │                                     │
    │◀════ JSON-RPC Response 通过 /sse ═══│
```
它有两个 Endpoint：
```
GET  /sse
POST /messages/
```
客户端发送请求和服务器返回结果，走的是两条不同通道。旧版规范明确要求一个 SSE Endpoint 和一个普通 POST Endpoint。

#### 5.3.4 Streamable HTTP + SSE 响应
```
MCP Client                            MCP Server
    │                                     │
    │ POST /mcp                           │
    │ JSON-RPC Request                    │
    ├────────────────────────────────────▶│
    │                                     │
    │◀════ SSE Response ══════════════════│
    │                                     │
```
它使用统一 Endpoint：
```
POST /mcp
```
请求和响应属于同一次 HTTP 交换：
```
一个 POST Request
       +
一个 SSE Response
```

## 总结
![mcp总结.png](reference/mcp%E6%80%BB%E7%BB%93.png)