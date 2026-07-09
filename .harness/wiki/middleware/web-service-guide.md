# Web 服务（FastAPI / WebSocket）使用指南

> 基于项目实际代码分析自动生成，可能随代码演进而过时。

## 1. 项目使用概况

- 依赖版本：`fastapi==0.115.12`、`uvicorn==0.34.2`、`websockets==15.0.1`、`python-multipart==0.0.20`（见 `requirements.txt`）
- 核心模块：`oxygent/routes.py`、`oxygent/web/`、`oxygent/mas.py`
- 主要场景：对外提供 HTTP API 与 WebSocket 实时消息通道（流式输出、消息推送），承载 OxyGent 的 Web 交互界面

## 2. 配置详情

### 依赖配置

```
fastapi==0.115.12
uvicorn==0.34.2
websockets==15.0.1
```

### 服务配置（`config.json`）

```json
"server": {
    "host": "127.0.0.1",
    "port": 8080,
    "auto_open_webpage": true,
    "log_level": "INFO",
    "workers": 1,
    "allow_origins": ["*"]
}
```

> 生产环境（`prod`）覆盖为 `host: 0.0.0.0`、`auto_open_webpage: false`。

## 3. 工具类与封装

| 模块 | 路径 | 说明 |
|------|------|------|
| 路由定义 | `oxygent/routes.py` | FastAPI 路由与接口，含 ES 客户端接入 |
| Web 资源 | `oxygent/web/` | 前端页面/静态资源 |
| MAS 启动 | `oxygent/mas.py` | 组装并通过 uvicorn 启动服务 |

## 4. 使用规范（从代码模式推断）

- **端口**：默认 `8080`；`workers=1`（单进程）。
- **跨域**：`allow_origins: ["*"]`，生产环境如有安全要求应收敛白名单。
- **实时通道**：通过 WebSocket 推送 `stream` / `stream_end` 等消息（配合 LLM 流式输出）。
- **消息开关**：`message.is_send_tool_call` / `is_send_observation` 等控制推送内容，生产环境默认关闭工具调用与观察推送。

## 5. 禁止事项与注意点

- **CORS 收敛**：生产环境避免 `allow_origins: ["*"]`。
- **绑定地址**：`prod` 绑定 `0.0.0.0` 对外暴露，注意网络与鉴权防护。

## 6. 参考

- 后端示例：`examples/backend/`