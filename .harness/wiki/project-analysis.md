# OxyGent 项目分析文档

> 基于项目实际代码分析生成，可能随代码演进而过时。
> 生成时间：2026-07-09

## 1. 项目定位

**OxyGent** 是京东（JD.com）开源的**生产级 Python 多智能体系统（Multi-Agent System, MAS）框架**。

核心理念：将**工具（Tool）、模型（Model）、智能体（Agent）**统一抽象为模块化的「**Oxy**」组件，像搭乐高积木一样快速组装、热插拔、跨场景复用，从而无需复杂配置即可通过纯 Python 接口构建、运行和演进多智能体系统。

- 开源地址：`github.com/jd-opensource/OxyGent`（Python）、`JDOxyGent4J`（Java 版）
- 官网：`oxygent.jd.com`
- 许可证：Apache License 2.0
- 基准表现：GAIA benchmark 59.14 分（接近当时最强开源系统 OWL 的 60.8 分）
- 环境要求：Python 3.10+

## 2. 核心抽象：Oxy 组件模型

所有能力都继承自基类 `Oxy`（`oxygent/oxy/base_oxy.py`），统一定义了执行生命周期、消息处理、日志、数据持久化。三大类组件：

### 2.1 Agents（智能体）— `oxygent/oxy/agents/`

| 组件 | 说明 |
|------|------|
| `ReActAgent` | ReAct（推理+行动）范式，LLM 推理与工具调用迭代循环，最常用 |
| `PlanAndSolveAgent` | 先规划再执行 |
| `ParallelAgent` | 并行执行子任务 |
| `ChatAgent` | 纯对话 |
| `RAGAgent` | 检索增强生成 |
| `A2AClientAgent` | 通过 A2A 协议调用远端 Agent |
| `ShellUseAgent` | 执行 Shell 命令 |
| `SkillAgent` | 技能型 Agent |
| `SSEOxyGent` / `WorkflowAgent` | 流式/工作流 Agent |

### 2.2 Models / LLM — `oxygent/oxy/llms/`

| 组件 | 说明 |
|------|------|
| `HttpLLM` | 纯 HTTP 调用（demo 默认使用）|
| `OpenAILLM` | 基于官方 AsyncOpenAI，支持流式 |
| `LiteLLM` | LiteLLM 多后端 |
| `LocalLLM` | 本地模型 |
| `MockLLM` / `ActorLLM` | 测试与角色扮演 |

支持任意 OpenAI 兼容 API，含流式输出、思考链（`<think>`）处理、Token 用量统计。

### 2.3 Tools（工具）— `oxygent/oxy/mcp_tools/`、`function_tools/`、`api_tools/`

| 组件 | 说明 |
|------|------|
| `MCPTool` / `StdioMCPClient` / `SSEMCPClient` / `StreamableMCPClient` | 通过 MCP 协议接入外部工具（三种传输）|
| `FunctionHub` / `FunctionTool` | 将 Python 函数封装为工具 |
| `HttpTool` | HTTP 接口工具 |

### 2.4 Flows（编排流程）— `oxygent/oxy/flows/`

`PlanAndSolve`（分解→执行→评估→重规划）、`Reflexion`/`MathReflexion`（反思）、`Workflow`、`ParallelFlow`。

## 3. 运行机制

核心运行时是 `MAS`（`oxygent/mas.py`）：

1. 开发者定义 `oxy_space`（Oxy 组件列表）
2. `MAS(oxy_space=...)` 组装成协作网络，指定一个 `is_master=True` 的主 Agent 作为入口调度器
3. 主 Agent 根据任务动态调度子 Agent / 工具协作完成
4. 通过 `mas.start_web_service()` 以 FastAPI + WebSocket 提供 Web 交互与流式输出

### 典型示例（`demo.py`）

```python
oxy_space = [
    oxy.HttpLLM(name="default_llm", ...),           # 模型
    preset_tools.time_tools,                        # 工具
    oxy.ReActAgent(name="time_agent", tools=["time_tools"]),   # 子 Agent
    oxy.ReActAgent(name="file_agent", tools=["file_tools"]),
    oxy.ReActAgent(name="math_agent", tools=["math_tools"]),
    oxy.ReActAgent(is_master=True, name="master_agent",
                   sub_agents=["time_agent", "file_agent", "math_agent"]),
]
async with MAS(oxy_space=oxy_space) as mas:
    await mas.start_web_service(first_query="现在几点？保存到 time.txt")
```

主 Agent 会先调 time_agent 查时间，再调 file_agent 写文件，形成多智能体协作链路。

## 4. 关键特性

| 特性 | 说明 |
|------|------|
| **高效开发** | 标准化 Oxy 组件，热插拔、跨场景复用，纯 Python 接口 |
| **智能协作** | 动态规划范式，Agent 自主分解任务、协商方案、实时适应 |
| **弹性架构** | 支持任意 Agent 拓扑（从简单 ReAct 到复杂混合规划），自动依赖映射 + 可视化调试 |
| **持续进化** | 内置评估引擎（`evaluation_manager.py`）自动生成训练数据，知识反馈闭环 |
| **可扩展** | 分布式调度器，成本线性增长、协作智能指数提升 |
| **全链路可审计** | 每次调用的 trace/node/history 记录到 Elasticsearch |

## 5. 技术栈与中间件

| 类别 | 技术 |
|------|------|
| Web 服务 | FastAPI + Uvicorn + WebSocket |
| 数据存储 | Elasticsearch（trace/node/history，支持 JES/Local/Memory 降级）|
| 缓存/队列 | Redis（JimDB / 本地内存降级）|
| 向量检索 | Vearch（工具语义召回）|
| 模型 | OpenAI SDK / httpx / LiteLLM |
| 工具协议 | MCP（mcp-sdk）|
| Agent 通信 | A2A（a2a-sdk）|

> 中间件详细使用指南见 `.harness/wiki/middleware/`。

## 6. 目录结构速览

| 目录 | 用途 |
|------|------|
| `oxygent/oxy/` | 核心 Oxy 组件（agents / llms / mcp_tools / flows / function_tools）|
| `oxygent/databases/` | ES / Redis / Vector 封装 |
| `oxygent/transport/a2a/` | A2A 通信 |
| `oxygent/mas.py` | 多智能体系统运行时 |
| `oxygent/config.py` | 配置读取 |
| `examples/` | 各类示例（a2a / agents / flows / distributed / ecommerce 等）|
| `mcp_servers/` / `function_hubs/` | 可复用的 MCP 工具服务与函数工具 |
| `applications/` | 完整应用示例（easybank / oxybank）|

## 7. 适用场景

- **开发者**：专注业务逻辑，无需重复造轮子
- **企业**：用统一框架替代孤岛式 AI 系统，降低沟通成本
- **用户**：获得智能体生态的无缝协作体验

典型落地：银行客服（oxybank）、电商、分布式多 Agent 协作等。