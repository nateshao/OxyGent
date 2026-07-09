# A2A（Agent-to-Agent 通信）使用指南

> 基于项目实际代码分析自动生成，可能随代码演进而过时。

## 1. 项目使用概况

- 依赖版本：`a2a-sdk==0.3.26`（见 `requirements.txt`）
- 核心模块：`oxygent/transport/a2a/`、`oxygent/oxy/agents/a2a_client_agent.py`
- 主要场景：不同 Agent（含跨进程/跨服务）之间基于 A2A 协议互相调用与协作

## 2. 配置详情

### 依赖配置

```
a2a-sdk==0.3.26
```

## 3. 工具类与封装

| 类名/模块 | 路径 | 说明 |
|-----------|------|------|
| `A2AServerGateway` | `oxygent/transport/a2a/a2a_server_gateway.py` | A2A 服务端网关，暴露本地 Agent 为 A2A 服务 |
| `A2AClientAgent` | `oxygent/oxy/agents/a2a_client_agent.py` | A2A 客户端 Agent，调用远端 A2A Agent |
| `a2a_card` | `oxygent/transport/a2a/a2a_card.py` | Agent Card（能力声明） |
| `a2a_mapper` | `oxygent/transport/a2a/a2a_mapper.py` | OxyRequest/Response 与 A2A 消息互转 |
| `a2a_protocol` | `oxygent/transport/a2a/a2a_protocol.py` | 协议定义 |
| `a2a_store` | `oxygent/transport/a2a/a2a_store.py` | 会话/任务状态存储 |

## 4. 使用规范（从代码模式推断）

- **客户端-服务端分离**：`A2AServerGateway` 负责对外暴露 Agent 能力，`A2AClientAgent` 负责发起远端调用。
- **协议映射**：跨系统消息通过 `a2a_mapper` 在 OxyGent 内部 Schema 与 A2A 标准消息间转换，避免直接耦合。
- **能力声明**：通过 Agent Card（`a2a_card`）声明可被调用的能力。

## 5. 项目实际用法示例

参见 `examples/a2a/` 目录中的示例，以及 `oxygent/oxy/agents/a2a_client_agent.py` 的客户端实现。

## 6. 禁止事项与注意点

- **协议版本一致**：客户端与服务端需使用兼容的 A2A SDK 版本（当前 `0.3.26`）。
- **消息映射**：跨系统交互务必经 `a2a_mapper` 转换，勿手工拼装协议消息。

## 7. 参考

- 示例目录：`examples/a2a/`
- 文档：`docs/docs_zh/examples/a2a/`、`docs/docs_en/examples/a2a/`