# Elasticsearch 使用指南

> 基于项目实际代码分析自动生成，可能随代码演进而过时。

## 1. 项目使用概况

- 依赖版本：`elasticsearch==7.17.12`（见 `requirements.txt`）
- 核心模块：`oxygent/databases/db_es/`、`oxygent/mas.py`、`oxygent/evaluation_manager.py`、`oxygent/live_prompt/manager.py`
- 主要场景：存储调用链路（trace）、节点日志（node）、读写历史（history）、消息（message）、Prompt 及评分（rating）等 MAS 运行数据

## 2. 配置详情

### 依赖配置

```
elasticsearch==7.17.12
```

### 连接配置（`config.json`，凭据已脱敏）

```json
"es": {
    "hosts": ["${PROD_ES_HOST_1}", "${PROD_ES_HOST_2}", "${PROD_ES_HOST_3}"],
    "user": "${PROD_ES_USER}",
    "password": "***"
},
"storage": { "es_engine": "LocalEs" },
"es_settings": { "number_of_shards": 1, "number_of_replicas": 1 }
```

### 引擎选择逻辑（`oxygent/mas.py` `init_db`）

- 配置了 `es.hosts/user/password` → 使用 `JesEs`（JD Elasticsearch Service，通过 `DBFactory` 单例）
- `storage.es_engine == "MemoryEs"` → 使用 `MemoryEs`（纯内存）
- 否则 → 使用 `LocalEs`（本地文件模拟）

## 3. 工具类与封装

| 类名 | 路径 | 关键方法 | 说明 |
|------|------|----------|------|
| `BaseEs` | `oxygent/databases/db_es/base_es.py` | `create_index` / `index` / `update` / `upsert` / `search` / `exists` / `delete` / `close` | 抽象接口，继承 `BaseDB` 获得自动重试能力 |
| `JesEs` | `oxygent/databases/db_es/jes_es.py` | 同上 | 官方 `Elasticsearch` 同步客户端 + `asyncio.to_thread` 包装为异步 |
| `LocalEs` | `oxygent/databases/db_es/local_es.py` | 同上 | 本地文件后端，用于开发/测试 |
| `MemoryEs` | `oxygent/databases/db_es/memory_es.py` | 同上 | 纯内存后端，功能等价 LocalEs |
| `BaseDB` | `oxygent/databases/base_db.py` | `try_decorator` | 自动为子类所有公开方法套上重试装饰器 |

## 4. 使用规范（从代码模式推断）

- **异常处理**：`BaseDB.__init_subclass__` 自动为所有公开方法应用 `try_decorator`（默认重试 1 次、间隔 0.1s），失败返回 `None` 并记录 `logger.error`。
- **同步转异步**：`JesEs` 通过 `asyncio.to_thread` 执行同步 ES 客户端调用，避免阻塞事件循环。
- **索引命名规范**：统一以 `{app_name}_` 为前缀，如 `{app}_trace`、`{app}_node`、`{app}_history`、`{app}_message`、`{app}_prompt`、`{app}_rating`。
- **幂等写入**：更新采用 `upsert`（`doc_as_upsert: true`）语义，存在则更新、不存在则创建。
- **单例复用**：ES 客户端经 `DBFactory.get_instance()` 缓存为单例，避免重复建连。

## 5. 项目实际用法示例

### 示例 1: 初始化 ES 并创建核心索引（`oxygent/mas.py`）

```python
db_factory = DBFactory()
if Config.get_es_config():
    jes_config = Config.get_es_config()
    self.es_client = db_factory.get_instance(
        JesEs, jes_config["hosts"], jes_config["user"], jes_config["password"]
    )
elif Config.get_storage_es_engine() == "MemoryEs":
    self.es_client = MemoryEs()
else:
    self.es_client = db_factory.get_instance(LocalEs)

app = Config.get_app_name()
await self.es_client.create_index(app + "_trace", self._trace_index_schema(settings))
await self.es_client.create_index(app + "_node", self._node_index_schema(settings))
await self.es_client.create_index(app + "_history", self._history_index_schema(settings))
```

### 示例 2: 文档 upsert 与检索（`oxygent/databases/db_es/jes_es.py`）

```python
async def upsert(self, index_name, doc_id, body):
    return await self._run_sync(
        self.client.update, index=index_name, id=doc_id,
        body={"doc": body, "doc_as_upsert": True},
    )

async def search(self, index_name, body):
    return await self._run_sync(self.client.search, index=index_name, body=body)
```

## 6. 资源清单

| 资源名 | 类型 | 用途 | 定义位置 |
|--------|------|------|----------|
| `{app}_trace` | index | 记录每次调用的调用链路 | `oxygent/mas.py` `init_db` |
| `{app}_node` | index | 记录每个节点的日志 | `oxygent/mas.py` `init_db` |
| `{app}_history` | index | 记录读写历史 | `oxygent/mas.py` `init_db` |
| `{app}_message` | index | 消息存储（`message.is_stored` 开启时） | `oxygent/mas.py` `init_db` |
| `{app}_prompt` / `{app}_prompt_history` | index | Prompt 及其历史 | `oxygent/mas.py` `init_db` |
| `{app}_rating` / `{app}_rating_stats` | index | 评分与统计 | `oxygent/mas.py` `init_db` |

## 7. 禁止事项与注意点

- **勿在生产误删索引**：`jes_es.py` 中 `create_index` 内的 `indices.delete` 已注释（`# !!! delete table`），切勿放开。
- **索引已存在**：`create_index` 在索引已存在时返回 `None`，不会重复创建，调用方需容忍此返回值。
- **凭据管理**：ES 账号密码通过环境变量 `${PROD_ES_*}` / `${TEST_ES_*}` 注入，禁止硬编码。

## 8. 监控与告警

- 失败操作统一由 `BaseDB.try_decorator` 通过 `logger.error(..., exc_info=True)` 记录，可据此接入日志告警。