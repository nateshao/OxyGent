# MCP（Model Context Protocol）使用指南

> 基于项目实际代码分析自动生成，可能随代码演进而过时。

## 1. 项目使用概况

- 依赖版本：`mcp==1.12.3`（见 `requirements.txt`）
- 核心模块：`oxygent/oxy/mcp_tools/`、`mcp_servers/`、`function_hubs/`
- 主要场景：通过 MCP 协议接入外部工具服务（数学、支付、订单、物流、浏览器、K8s 等），供 Agent 作为工具调用

## 2. 配置详情

### 依赖配置

```
mcp==1.12.3
```

### 工具相关配置（`config.json`）

```json
"tool": {
    "mcp_is_keep_alive": true,
    "is_concurrent_init": true,
    "semaphore": 1024,
    "timeout": 60
}
```

> MCP server 的命令、参数、地址由各 MCP Client Oxy 实例在代码/配置中声明。

## 3. 工具类与封装

| 类名 | 路径 | 传输方式 | 说明 |
|------|------|----------|------|
| `BaseMCPClient` | `oxygent/oxy/mcp_tools/base_mcp_client.py` | — | MCP 客户端基类，管理 `ClientSession` 与工具发现 |
| `StdioMCPClient` | `oxygent/oxy/mcp_tools/stdio_mcp_client.py` | stdio | 启动并管理外部进程（如 Node/Python 脚本）作为 MCP server |
| `SSEMCPClient` | `oxygent/oxy/mcp_tools/sse_mcp_client.py` | SSE | 通过 Server-Sent Events 连接远端 MCP server |
| `StreamableMCPClient` | `oxygent/oxy/mcp_tools/streamable_mcp_client.py` | streamable-http | 通过 streamable HTTP 连接 |
| `MCPTool` | `oxygent/oxy/mcp_tools/mcp_tool.py` | — | 单个 MCP 工具的封装 |

## 4. 使用规范（从代码模式推断）

- **多传输方式**：stdio（本地进程）、SSE、streamable-http 三种，按 server 部署形态选择。
- **进程/目录校验**：`StdioMCPClient` 会在启动前校验工具文件/目录存在（`_ensure_directories_exist`），如 `--directory ... run` 模式下缺文件会抛 `FileNotFoundError`。
- **保活与并发**：`tool.mcp_is_keep_alive=true` 保持连接；`is_concurrent_init=true` 并发初始化工具；`semaphore=1024`、`timeout=60` 控制并发与超时。
- **工具发现**：`init(is_fetch_tools=True)` 连接后自动发现并注册 server 提供的工具。
- **优雅退出**：stdio 进程有终止超时 `_TERMINATE_TIMEOUT=2.0s`。

## 5. 项目实际用法示例

### 示例 1: stdio 传输初始化（`oxygent/oxy/mcp_tools/stdio_mcp_client.py`）

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

class StdioMCPClient(BaseMCPClient):
    params: dict[str, Any] = Field(default_factory=dict)  # command, args, env
    async def init(self, is_fetch_tools: bool = True) -> None:
        ...  # 启动外部进程，建立 stdio 通道，发现工具
```

## 6. 禁止事项与注意点

- **工具文件缺失即报错**：使用 `--directory` 运行模式时，目标脚本不存在会直接抛错，需确保文件就位。
- **超时与保活**：长任务需关注 `tool.timeout`；关闭 keep-alive 会导致每次重连。

## 7. 可用 MCP server 参考

见项目 `mcp_servers/`（如 `math_tools.py`、`payment_tools.py`、`order_tools.py`、`logistics_tools.py`、`browser/`、`kubernetes_mcp_server/`）与 `function_hubs/`。