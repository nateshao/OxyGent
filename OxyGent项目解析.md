# OxyGent 全面解析

---

## 一、这是什么项目

OxyGent 是京东开源的**面向生产环境的多智能体协作框架**，核心理念是将工具（Tool）、模型（LLM）、智能体（Agent）统一抽象为可插拔的原子算子 **Oxy**，像搭乐高一样构建多智能体系统。论文被 ACL 2026 接收，GAIA 基准 59.14 分。

---

## 二、与传统 Agent 框架的区别

| 维度 | 传统框架 (LangChain/AutoGen/CrewAI) | OxyGent |
|------|--------------------------------------|---------|
| **统一抽象** | Agent/Tool/LLM 接口各异 | 一切皆 `Oxy` 基类，统一生命周期 |
| **编排方式** | 静态 Pipeline / 硬编码 | 动态规划 + 多种协作拓扑自由组合 |
| **可观测性** | 黑盒执行 | 全链路 Trace + ES 持久化，每步决策可审计 |
| **进化能力** | 无内置反馈闭环 | 评估引擎 + LivePrompt 热更新，持续自我进化 |
| **安全管控** | 缺失 | OxyFactory 安全校验 + 权限白名单 |
| **工具生态** | 有限内置工具 | FunctionHub、HttpTool、MCP 协议、BankTool 等 |
| **可扩展性** | 新增能力需改框架代码 | Oxy 组件热插拔，零配置接入，分布式线性扩容 |

### 核心优势

1. **统一抽象（Oxy）**：Agent、Tool、LLM、Flow 共享同一基类和执行生命周期（`_pre_process → _execute → _post_process`），极大降低心智负担和集成成本
2. **弹性拓扑**：从简单 ReAct 单智能体到 PlanAndSolve 多步规划、Workflow 编排、Parallel 并行，乃至分布式远程 Agent（A2A），均可自由组合
3. **生产就绪**：内置 ES/Redis/Vearch 持久化、SSE 流式输出、FastAPI Web 服务、Batch 批处理、CLI 模式，开箱即用于生产环境
4. **持续进化闭环**：评估引擎自动采集用户评分 → 生成训练数据 → Prompt 热更新 → 智能体自我改进，形成完整进化回路
5. **安全管控**：OxyFactory 对危险组件做安全校验，权限系统（`permitted_tools`）控制智能体可用能力范围

---

## 三、怎么使用

### 1. 安装

```bash
pip install oxygent
```

### 2. 编写代码（以 demo.py 为例）

```python
import asyncio
import os
from oxygent import MAS, Config, oxy, preset_tools

Config.set_agent_llm_model("default_llm")

oxy_space = [
    # 1) 注册 LLM
    oxy.HttpLLM(
        name="default_llm",
        api_key=os.getenv("DEFAULT_LLM_API_KEY"),
        base_url=os.getenv("DEFAULT_LLM_BASE_URL"),
        model_name=os.getenv("DEFAULT_LLM_MODEL_NAME"),
    ),
    # 2) 注册工具
    preset_tools.time_tools,
    preset_tools.file_tools,
    preset_tools.math_tools,
    # 3) 注册 Agent（绑定工具）
    oxy.ReActAgent(name="time_agent", desc="查询时间", tools=["time_tools"]),
    oxy.ReActAgent(name="file_agent", desc="操作文件", tools=["file_tools"]),
    oxy.ReActAgent(name="math_agent", desc="数学计算", tools=["math_tools"]),
    # 4) 注册 Master Agent（绑定子Agent）
    oxy.ReActAgent(
        is_master=True,
        name="master_agent",
        sub_agents=["time_agent", "file_agent", "math_agent"],
    ),
]

# 5) 启动 MAS
async def main():
    async with MAS(oxy_space=oxy_space) as mas:
        await mas.start_web_service(
            first_query="现在几点？请保存到time.txt"
        )

if __name__ == "__main__":
    asyncio.run(main())
```

### 3. 设置环境变量

```bash
export DEFAULT_LLM_API_KEY="your_api_key"
export DEFAULT_LLM_BASE_URL="your_base_url"
export DEFAULT_LLM_MODEL_NAME="your_model_name"
```

### 4. 运行

```bash
python demo.py
```

### 5. 运行模式

| 模式 | 方法 | 说明 |
|------|------|------|
| Web 服务 | `mas.start_web_service()` | FastAPI + SSE + 内置 Web UI |
| 交互式 | `mas.start_cli_mode()` | REPL 命令行交互 |
| 批处理 | `mas.start_batch_processing()` | 并发批量执行 |
| 编程接口 | `mas.chat_with_agent()` | 单次调用返回结果 |

---

## 四、模块结构

```
oxygent/
├── mas.py                    # MAS 运行时容器（注册/初始化/调度/持久化）
├── config.py                 # 全局配置
├── oxy_factory.py            # Oxy 工厂（安全创建组件）
├── schemas/                  # 数据模型
│   ├── oxy.py                #   OxyRequest / OxyResponse / OxyState
│   ├── memory.py             #   Memory / Message
│   ├── llm.py                #   LLMResponse / LLMState
│   ├── observation.py        #   ExecResult / Observation
│   ├── message.py            #   SSEMessage
│   └── evaluation.py         #   评分相关模型
│
├── oxy/                      # 核心 Oxy 组件
│   ├── base_oxy.py           # Oxy 基类（统一生命周期、权限、拦截器）
│   ├── base_flow.py          # Flow 基类
│   ├── base_tool.py          # Tool 基类
│   │
│   ├── agents/               # ── Agent 模块 ──
│   │   ├── local_agent.py    #   本地 Agent（工具/子Agent/记忆管理）
│   │   ├── react_agent.py    #   ReAct Agent（推理-行动循环）
│   │   ├── plan_and_solve_agent.py  # 规划-求解 Agent
│   │   ├── parallel_agent.py        # 并行 Agent
│   │   ├── chat_agent.py            # 对话 Agent
│   │   ├── rag_agent.py             # RAG Agent
│   │   ├── workflow_agent.py        # 自定义工作流 Agent
│   │   ├── skill_agent.py           # 技能 Agent
│   │   ├── shell_use_agent.py       # Shell Agent
│   │   ├── a2a_client_agent.py      # A2A 远程协作 Agent
│   │   ├── sse_oxy_agent.py         # SSE 流式 Agent
│   │   ├── remote_agent.py          # 远程 Agent 基类
│   │   └── base_agent.py            # Agent 基类（Trace/持久化）
│   │
│   ├── flows/                # ── Flow 模块（协作编排）──
│   │   ├── plan_and_solve.py #   规划-求解流（分解→执行→重规划）
│   │   ├── reflexion.py      #   反思流（生成→评估→改进循环）
│   │   ├── workflow.py       #   自定义工作流
│   │   └── parallel_flow.py  #   并行流
│   │
│   ├── llms/                 # ── LLM 接入 ──
│   │   ├── base_llm.py       #   LLM 基类
│   │   ├── openai_llm.py     #   OpenAI 兼容
│   │   ├── http_llm.py       #   HTTP API 调用
│   │   ├── lite_llm.py       #   LiteLLM 多模型
│   │   ├── local_llm.py      #   本地模型
│   │   ├── actor_llm.py      #   Actor LLM
│   │   ├── remote_llm.py     #   远程 LLM
│   │   └── mock_llm.py       #   Mock 测试用
│   │
│   ├── api_tools/            # ── HTTP 工具 ──
│   │   └── http_tool.py      #   HTTP API 调用工具
│   │
│   ├── mcp_tools/            # ── MCP 协议工具 ──
│   │   ├── mcp_tool.py       #   MCP 工具封装
│   │   ├── base_mcp_client.py #  MCP 客户端基类
│   │   ├── stdio_mcp_client.py #  Stdio MCP 客户端
│   │   ├── sse_mcp_client.py   #  SSE MCP 客户端
│   │   └── streamable_mcp_client.py # Streamable MCP 客户端
│   │
│   ├── function_tools/       # ── 函数工具 ──
│   │   ├── function_hub.py   #   FunctionHub（工具集容器）
│   │   └── function_tool.py  #   FunctionTool（单函数工具）
│   │
│   └── bank_tools/           # ── Bank 工具 ──
│       ├── bank_tool.py      #   Bank 工具
│       ├── bank_client.py    #   Bank 客户端
│       └── base_bank.py      #   Bank 基类
│
├── preset_tools/             # 预置工具集
│   ├── math_tools.py         #   数学计算
│   ├── file_tools.py         #   文件操作
│   ├── time_tools.py         #   时间查询
│   ├── http_tools.py         #   HTTP 请求
│   ├── string_tools.py       #   字符串处理
│   ├── system_tools.py       #   系统信息
│   ├── shell_tools.py        #   Shell 命令
│   ├── python_tools.py       #   Python 执行
│   ├── image_gen_tools.py    #   图片生成
│   ├── ssh_tools.py          #   SSH 连接
│   └── oxy_manage_tools.py   #   Oxy 管理
│
├── live_prompt/              # Prompt 热更新
│   ├── manager.py            #   Prompt 管理器（ES 存储）
│   ├── optimizer.py          #   Prompt 优化器
│   ├── version.py            #   版本同步协调器
│   └── wrapper.py            #   热更新包装器
│
├── evaluation_manager.py     # 评估管理器（评分采集+统计）
├── embedding_cache.py        # Embedding 缓存
│
├── databases/                # 数据库层
│   ├── db_es.py              #   Elasticsearch（JesEs/LocalEs/MemoryEs）
│   ├── db_redis.py           #   Redis（JimdbApRedis/LocalRedis）
│   └── db_vector.py          #   向量数据库（VearchDB）
│
├── routes.py                 # FastAPI 路由（Web API）
├── transport/                # 传输层
└── utils/                    # 工具函数
```

---

## 五、业务流转

### 完整请求生命周期

```
用户请求
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  MAS.chat_with_agent()                              │
│  构造 OxyRequest → 路由到 master_agent               │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│  Oxy.execute(oxy_request)                           │
│  ┌──────────────────────────────────────────────┐   │
│  │ 1. _pre_process:       生成 node_id、压入调用栈│   │
│  │ 2. _request_interceptor: 重启/重放拦截        │   │
│  │ 3. _pre_log:           日志记录               │   │
│  │ 4. _pre_save_data:     ES 持久化 trace/node   │   │
│  │ 5. _execute:           核心执行逻辑           │   │
│  │ 6. _post_process:      输出处理 + ES 更新     │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         ▼                    ▼
   ReActAgent           PlanAndSolveAgent
   (推理-行动循环)        (规划-执行-重规划)
   ┌───────────────┐    ┌───────────────┐
   │ 思考 → 选择   │    │ Planner 分解  │
   │ 工具 → 调用   │    │ Executor 执行 │
   │ 观察 → 再思考 │    │ Replanner 重调│
   │  ... → 回答   │    └───────────────┘
   └───────┬───────┘
           │
           ▼
   oxy_request.call(callee="tool_name")
           │
           ▼
┌───────────────────────────────────────┐
│  MAS 路由到目标 Tool / Agent          │
│  查找 oxy_name_to_oxy 注册表          │
│  递归执行子 Oxy                       │
└──────────────────┬────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│  结果回传 & 后处理                                   │
│  - OxyResponse 逐层返回                              │
│  - SSE 流式推送到前端（Redis 队列）                   │
│  - ES 持久化 trace / node / history                  │
│  - EvaluationManager 采集用户评分                     │
│  - LivePrompt 根据反馈优化 Prompt                     │
└─────────────────────────────────────────────────────┘
```

### 关键流转机制

| 机制 | 说明 | 关键代码位置 |
|------|------|-------------|
| **调用链追踪** | `OxyRequest.call_stack` 记录完整调用路径，如 `user → master_agent → time_agent → time_tools` | `schemas/oxy.py` |
| **权限控制** | Agent 只能调用 `permitted_tool_name_list` 白名单中的工具 | `oxy/base_oxy.py` |
| **数据共享** | `OxyRequest.shared_data` 在同一 trace 的上下游间传递上下文 | `schemas/oxy.py` |
| **持久化** | 每次调用生成 trace → node → history 三层 ES 记录，支持断点重启 | `mas.py` |
| **进化闭环** | 用户评分 → EvaluationManager → 训练数据 → PromptOptimizer → LivePrompt 热更新 | `evaluation_manager.py` / `live_prompt/` |

### ES 持久化索引

| 索引名 | 用途 |
|--------|------|
| `{app}_trace` | 会话追踪记录 |
| `{app}_node` | 每个执行节点的输入/输出/状态 |
| `{app}_history` | 读写操作的会话历史 |
| `{app}_message` | SSE 消息记录 |
| `{app}_prompt` | Prompt 版本管理 |
| `{app}_prompt_history` | Prompt 变更历史 |
| `{app}_rating` | 用户评分记录 |
| `{app}_rating_stats` | 评分统计聚合 |

---

## 六、Agent 类型速览

| Agent | 类 | 核心逻辑 |
|-------|-----|---------|
| ReAct Agent | `ReActAgent` | 推理-行动循环：LLM 思考 → 选择工具 → 执行 → 观察 → 再思考，最多 `max_react_rounds` 轮 |
| PlanAndSolve Agent | `PlanAndSolveAgent` | Planner 分解任务 → Executor 逐步执行 → 失败时 Replanner 重规划 |
| Parallel Agent | `ParallelAgent` | 同一任务分发给所有 team member 并行执行，LLM 汇总结果 |
| Chat Agent | `ChatAgent` | 纯对话，无工具调用 |
| RAG Agent | `RAGAgent` | 检索增强生成 |
| Workflow Agent | `WorkflowAgent` | 执行用户自定义工作流函数 |
| Skill Agent | `SkillAgent` | 技能型 Agent |
| Shell Agent | `ShellUseAgent` | 操作系统 Shell 交互 |
| A2A Agent | `A2AClientAgent` | Agent-to-Agent 远程协作协议 |
| SSE Agent | `SSEOxyGent` | SSE 流式输出 Agent |

---

## 七、Flow 类型速览

| Flow | 类 | 核心逻辑 |
|------|-----|---------|
| PlanAndSolve | `PlanAndSolve` | 分解目标为有序步骤 → 逐步执行 → 可选重规划，最多 `max_replan_rounds` 轮 |
| Reflexion | `Reflexion` | Worker 生成答案 → Reflexion Agent 评估 → 不满意则改进反馈 → 再生成，最多 `max_reflexion_rounds` 轮 |
| Workflow | `Workflow` | 执行用户自定义工作流函数 |
| Parallel | `ParallelFlow` | 并行执行多个子任务 |

---

## 八、工具接入方式

| 方式 | 类 | 说明 |
|------|-----|------|
| 预置工具 | `preset_tools.*` | 开箱即用的 FunctionHub（math/file/time/shell 等） |
| 自定义函数 | `FunctionTool` / `FunctionHub` | 用 Python 函数定义工具 |
| HTTP API | `HttpTool` | 封装 REST API 为工具 |
| MCP 协议 | `MCPTool` / `StdioMCPClient` / `SSEMCPClient` | 接入 MCP 标准工具服务 |
| Bank 工具 | `BankTool` / `BankClient` | 业务服务编排 |