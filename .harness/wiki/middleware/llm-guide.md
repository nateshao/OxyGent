# LLM（大模型调用）使用指南

> 基于项目实际代码分析自动生成，可能随代码演进而过时。

## 1. 项目使用概况

- 依赖版本：`openai==1.77.0`、`httpx==0.28.1`（见 `requirements.txt`）
- 核心模块：`oxygent/oxy/llms/`
- 主要场景：作为 Agent 的推理引擎，支持 OpenAI 兼容 API、纯 HTTP 接口、本地模型、LiteLLM 等多种后端；支持流式输出与 Token 用量统计

## 2. 配置详情

### 依赖配置

```
openai==1.77.0
httpx==0.28.1
```

### LLM 运行参数（`config.json`）

```json
"llm": {
    "temperature": 0.1,
    "max_tokens": 4096,
    "top_p": 1,
    "semaphore": 16,
    "timeout": 300
},
"token_tracking": {
    "enabled": true,
    "default_encoding": "o200k_base"
}
```

> API Key / base_url / model_name 由各 LLM Oxy 实例通过参数或环境变量注入（脱敏）。

## 3. 工具类与封装

| 类名 | 路径 | 关键方法 | 说明 |
|------|------|----------|------|
| `BaseLLM` | `oxygent/oxy/llms/base_llm.py` | `_get_messages` / `_build_payload` / `_build_token_usage` | LLM 抽象基类 |
| `RemoteLLM` | `oxygent/oxy/llms/remote_llm.py` | — | 远程 LLM 基类（含 api_key/base_url/model_name）|
| `OpenAILLM` | `oxygent/oxy/llms/openai_llm.py` | `_execute` / `_get_client` | 基于官方 `AsyncOpenAI`，支持流式 |
| `HttpLLM` | `oxygent/oxy/llms/http_llm.py` | `_execute` | 纯 HTTP（httpx）调用 |
| `LiteLLM` / `LocalLLM` / `ActorLLM` / `MockLLM` | `oxygent/oxy/llms/*.py` | — | 其它后端与测试用模拟 |

## 4. 使用规范（从代码模式推断）

- **懒加载客户端**：`OpenAILLM._get_client` 首次调用才创建 `AsyncOpenAI` 并复用。
- **默认流式**：`OpenAILLM` 默认 `stream=True` 并携带 `stream_options.include_usage`，通过 `oxy_request.send_message` 逐块推送 `stream` / `stream_end`。
- **思考链处理**：识别 `reasoning_content`，用 `<think>` / `</think>` 包裹推理内容后再输出正式回答。
- **Token 统计**：通过 `_build_token_usage` 结合返回 usage 与本地编码（默认 `o200k_base`）统计用量。
- **并发/超时**：由 `llm.semaphore=16`、`llm.timeout=300` 控制。

## 5. 项目实际用法示例

### 示例 1: OpenAI 客户端懒加载与调用（`oxygent/oxy/llms/openai_llm.py`）

```python
def _get_client(self) -> AsyncOpenAI:
    if self._client is None:
        self._client = AsyncOpenAI(api_key=self.api_key, base_url=self.base_url)
    return self._client

completion = await self._get_client().chat.completions.create(**payload)
```

## 6. 禁止事项与注意点

- **凭据管理**：`api_key`/`base_url` 禁止硬编码，通过配置或环境变量注入。
- **流式判空**：usage-only chunk 的 `choices` 为空，需先判断 `if not chunk.choices: continue`。

## 7. 监控与告警

- Token 用量随 `OxyResponse.extra["usage"]` 返回，可用于成本监控。