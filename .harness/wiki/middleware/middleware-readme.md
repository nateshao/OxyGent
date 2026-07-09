# 中间件使用 Wiki

> 基于项目实际代码分析自动生成
> 生成时间：2026-07-09
> 项目：OxyGent（Python 多智能体框架）

## 中间件索引

### 数据存储
- [Elasticsearch](./elasticsearch-guide.md) — 存储调用链路/节点/历史/消息等 MAS 运行数据，支持 JES/Local/Memory 三种引擎
- [Redis](./redis-guide.md) — 键值缓存与带过期列表操作，支持 JimDB 与本地内存降级
- [Vearch 向量数据库](./vearch-guide.md) — 向量存储与相似度检索，用于工具语义召回

### 模型与工具
- [LLM（大模型调用）](./llm-guide.md) — OpenAI/HTTP/LiteLLM 等多后端，支持流式与 Token 统计
- [MCP（Model Context Protocol）](./mcp-guide.md) — stdio/SSE/streamable-http 三种传输接入外部工具

### 服务通信
- [A2A（Agent-to-Agent）](./a2a-guide.md) — 基于 a2a-sdk 的跨 Agent 协作通信
- [Web 服务（FastAPI/WebSocket）](./web-service-guide.md) — HTTP API 与实时消息通道

## 未使用 / 仅配置的中间件

- **MySQL/关系型数据库**：`function_hubs/sql_tools.py` 等为示例工具，框架本体未内置 ORM 或统一 MySQL 连接池，不属于核心中间件。
- **消息队列**：项目使用 Redis list 作为轻量队列，未引入 Kafka/RocketMQ/RabbitMQ 等独立 MQ。
- **配置中心/限流熔断/定时任务**：框架本体未发现 Nacos/Sentinel/XXL-Job 等使用。

## 核心规范速查

| 类别 | 规范 |
|------|------|
| 数据库抽象 | 所有 DB 客户端继承 `BaseDB`（`oxygent/databases/base_db.py`），公开方法自动套重试装饰器 |
| 单例管理 | ES/Vector 等经 `DBFactory.get_instance()` 缓存为单例，避免重复建连 |
| 环境降级 | 未配置真实中间件时自动降级为本地实现（LocalEs/MemoryEs/LocalRedis），便于开发测试 |
| 异常处理 | 统一 `logger.error(..., exc_info=True)`；DB 操作失败多返回 `None`，调用方需判空 |
| 凭据管理 | 连接凭据一律通过 `${ENV_VAR}` 环境变量注入，禁止硬编码 |
| 命名规范 | ES 索引统一 `{app_name}_` 前缀；Redis key 运行时动态拼接 |
| 异步优先 | 同步客户端（如 ES）通过 `asyncio.to_thread` 包装；HTTP 用 `httpx.AsyncClient` |

## 目录约定

- 数据库封装：`oxygent/databases/`（`db_es` / `db_redis` / `db_vector`）
- LLM/工具封装：`oxygent/oxy/`（`llms` / `mcp_tools` / `agents`）
- 传输层：`oxygent/transport/`（`a2a`）
- 配置读取：`oxygent/config.py`（`Config.get_*` 系列方法）