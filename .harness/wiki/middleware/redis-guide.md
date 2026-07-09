# Redis 使用指南

> 基于项目实际代码分析自动生成，可能随代码演进而过时。

## 1. 项目使用概况

- 依赖版本：`aioredis==2.0.1`（见 `requirements.txt`）
- 核心模块：`oxygent/databases/db_redis/`、`oxygent/mas.py`
- 主要场景：键值缓存、带过期时间的列表（list）操作，用于消息/历史缓存等场景；未配置 Redis 时自动降级为本地内存实现

## 2. 配置详情

### 依赖配置

```
aioredis==2.0.1
```

### 连接配置（`config.json`，凭据已脱敏）

```json
"redis": {
    "host": "${PROD_REDIS_HOST}",
    "port": 5360,
    "password": "***",
    "db": 0
},
"redis_param": {
    "expire_time": 86400,
    "max_size": 1024,
    "max_length": 20480
}
```

### 引擎选择逻辑（`oxygent/mas.py` `init_db`）

- 配置了 `redis.host/port/password` → 使用 `JimdbApRedis`（JD JimDB）
- 否则 → 使用 `LocalRedis`（本地内存模拟）

## 3. 工具类与封装

| 类名 | 路径 | 关键方法 | 说明 |
|------|------|----------|------|
| `BaseRedis` | `oxygent/databases/db_redis/base_redis.py` | `set` / `get` / `mset` / `mget` / `delete` / `lpush` / `brpop` / `lrange` / `ltrim` / `llen` / `expire` | 抽象接口，继承 `BaseDB` |
| `JimdbApRedis` | `oxygent/databases/db_redis/jimdb_ap_redis.py` | 同上 + `rpop` / `lrem` / `lindex` | 基于 `aioredis`，含连接池与重试 |
| `LocalRedis` | `oxygent/databases/db_redis/local_redis.py` | `lpush` / `rpop` / `close` 等 | 用 `deque` 内存模拟，开发/测试用 |

## 4. 使用规范（从代码模式推断）

- **连接池**：`JimdbApRedis` 使用 `Redis.from_url(...)`，`max_connections=5`、`health_check_interval=30`。
- **重试策略**：`retry_decorator` 捕获 `ConnectionError`/`TimeoutError`，先 `close` 再重建连接池重试一次；其它异常记录日志并返回 `None`。
- **默认过期**：`set` 默认 `ex=86400`（1 天）；`lpush` 默认过期取 `redis_param.expire_time`。
- **列表约束**：`lpush` 使用 pipeline 原子执行 `lpush + ltrim + expire`，`max_size` 默认取 `redis_param.max_size`，`max_length` 为 `redis_param.max_length * 1024`（约 20MB）。
- **值类型**：`lpush` 支持 `str/bytes/int/float/dict`，`dict` 会 `json.dumps` 序列化并按 `max_length` 截断，其它类型抛 `ValueError`。
- **brpop 模拟**：JimDB 不支持 `brpop`，`JimdbApRedis.brpop` 用 `rpop` + `asyncio.sleep(timeout)` 模拟阻塞。

## 5. 项目实际用法示例

### 示例 1: 初始化 Redis（`oxygent/mas.py`）

```python
redis_config = Config.get_redis_config()
if redis_config:
    self.redis_client = JimdbApRedis(
        host=redis_config["host"], port=redis_config["port"],
        password=redis_config["password"], db=redis_config.get("db", 0),
    )
else:
    self.redis_client = LocalRedis()
```

### 示例 2: 带限长/过期的列表写入（`oxygent/databases/db_redis/jimdb_ap_redis.py`）

```python
async with self.redis_pool.pipeline(transaction=False) as pipe:
    pipe.lpush(key, *new_values)
    pipe.ltrim(key, 0, max_size - 1)
    pipe.expire(key, ex)
    results = await pipe.execute()
    return results[0]
```

## 6. 资源清单

> 项目未集中定义 key 常量，key 由调用方在运行时动态生成。命名规范以业务上下文拼接为主，参见 `oxygent/mas.py` 及各 Oxy 组件调用处。

| 资源名 | 类型 | 用途 | 定义位置 |
|--------|------|------|----------|
| 动态 key | key/list | 运行时缓存与列表队列 | 调用方运行时生成 |

## 7. 禁止事项与注意点

- **值类型限制**：向 `lpush` 传入非 `str/bytes/int/float/dict` 会抛 `ValueError`，需先序列化。
- **返回 None 语义**：非连接类异常会被 `retry_decorator` 吞并返回 `None`，调用方需判空。
- **凭据管理**：Redis 密码通过环境变量 `${PROD_REDIS_PASSWORD}` / `${TEST_REDIS_PASSWORD}` 注入，禁止硬编码。
- **LIFO 语义**：因使用 `lpush`，`lrange` 取出的元素后进先出，注意顺序。

## 8. 监控与告警

- 连接重连与操作失败通过 `logger.error(..., exc_info=True)` 记录（见 `retry_decorator`），可据此接入告警。