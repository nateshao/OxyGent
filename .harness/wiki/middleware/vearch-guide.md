# Vearch 向量数据库使用指南

> 基于项目实际代码分析自动生成，可能随代码演进而过时。

## 1. 项目使用概况

- 通信方式：基于 `httpx` 的 HTTP 调用（Vearch master/router 节点），未固定依赖 SDK
- 核心模块：`oxygent/databases/db_vector/vearch_db.py`、`oxygent/embedding_cache.py`
- 主要场景：向量存储、相似度检索、工具（tool）语义检索与召回

## 2. 配置详情

### 连接配置（`config.json`，`test`/`prod` 环境，凭据已脱敏）

```json
"vearch": {
    "router_url": "${VEARCH_ROUTER_URL}",
    "master_url": "${VEARCH_MASTER_URL}",
    "db_name": "${VEARCH_DB_NAME}",
    "tool_space_name": "${VEARCH_TOOL_SPACE_NAME}",
    "embedding_model_url": "${EMBEDDING_MODEL_URL}"
}
```

> `default` 环境下 `vearch` 为空对象，即默认不启用向量库。

## 3. 工具类与封装

| 类名 | 路径 | 关键方法 | 说明 |
|------|------|----------|------|
| `BaseVectorDB` | `oxygent/databases/db_vector/base_vector_db.py` | `create_space` / `query_search` | 抽象接口，继承 `BaseDB` |
| `VearchDB` | `oxygent/databases/db_vector/vearch_db.py` | `create_space` / `query_search` 等 | Vearch 完整客户端，支持嵌入与过滤 |
| `VectorToolAsync` | `oxygent/databases/db_vector/vearch_db.py` | `create_db` / `create_space` 等静态方法 | 低层异步 HTTP 操作（`httpx.AsyncClient`）|
| `EmbeddingCache` | `oxygent/embedding_cache.py` | — | 嵌入向量缓存 |

## 4. 使用规范（从代码模式推断）

- **HTTP 通信**：所有 Vearch 操作通过 `httpx.AsyncClient` 异步发起，区分 master（库/空间管理）与 router（数据读写）节点。
- **重试与容错**：`VearchDB` 继承 `BaseDB`，公开方法自动套 `try_decorator` 重试。
- **嵌入解耦**：向量嵌入通过独立的 `embedding_model_url` 服务生成，并有 `EmbeddingCache` 缓存以降低重复计算。
- **按需启用**：默认配置为空，仅在 `test`/`prod` 配置向量库地址后启用。

## 5. 项目实际用法示例

### 示例 1: 低层建库/建空间（`oxygent/databases/db_vector/vearch_db.py`）

```python
class VectorToolAsync(object):
    @staticmethod
    async def create_db(master_url: str, db_name: str) -> dict[str, Any]:
        url = f"{master_url}/db/_create"
        data = {"name": db_name}
        async with httpx.AsyncClient() as client:
            response = await client.put(url, json=data)
            return response.json()
```

## 6. 资源清单

| 资源名 | 类型 | 用途 | 定义位置 |
|--------|------|------|----------|
| `${VEARCH_DB_NAME}` | database | 向量库名称 | `config.json` |
| `${VEARCH_TOOL_SPACE_NAME}` | space | 工具向量空间（tool 检索） | `config.json` |
| `${EMBEDDING_MODEL_URL}` | service | 生成嵌入向量的模型服务地址 | `config.json` |

## 7. 禁止事项与注意点

- **凭据/地址管理**：Vearch 与嵌入服务地址均通过环境变量注入，禁止硬编码。
- **master/router 分工**：库与空间管理走 master，数据读写走 router，勿混用。
- **默认不启用**：`default` 环境为空配置，切换到需要向量检索的功能前需配置好向量库。

## 8. 监控与告警

- 失败操作由 `BaseDB.try_decorator` 记录 `logger.error`，可据此接入告警。