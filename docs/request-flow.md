# 请求生命周期与核心链路

## 1. 入口与通用中间件

- 应用入口在 `main.py`：
  - 注册 CORS、响应日志中间件、异常处理器
  - 挂载 `/v1`、`/v1/admin`、`/v1/public`、页面路由
- `ResponseLoggerMiddleware`：
  - 为请求生成 trace_id
  - 跳过静态页与部分前端路由日志
  - 记录请求/响应耗时

---

## 2. Chat Completions 主链路

### 2.1 路由层（`app/api/v1/chat.py`）

职责：
- Pydantic 定义请求模型（`ChatCompletionRequest`）
- 校验角色、内容块类型、媒体 URL/Data URI
- 根据 model 决策调用文本 / 图片 / 视频服务
- 组装 OpenAI 兼容 JSON 或 `StreamingResponse`

关键点：
- `reasoning_effort=none` 会关闭思维输出
- 图片/视频有单独配置对象（`image_config` / `video_config`）
- 非法 base64 裸串会被拦截，要求 data URI

### 2.2 服务层（`app/services/grok/services/chat.py`）

职责：
- `MessageExtractor` 把 OpenAI 消息转成上游 message + 附件列表
- `GrokChatService.chat_openai` 做模型映射、token 选择、重试
- 使用并发信号量限制 chat reverse 并发
- 调用 `wrap_stream_with_usage` 在成功后计费

关键点：
- `ModelService` 提供 `model_id -> grok_model + mode`
- 429 时会尝试切换 token（并标记限流状态）
- 失败会抛 `AppException/UpstreamException` 统一给异常处理器

### 2.3 Reverse 层

- 由 `app/services/reverse/app_chat.py` 等模块与 Grok 上游通信。
- 这里处理 headers、代理、重试、流读取与上游特定字段。

---

## 3. 图片链路

### 3.1 图片生成（`/v1/images/generations`）

路由：`app/api/v1/image.py`
- 校验 `model=grok-imagine-1.0`
- 校验 `n(1~10)`、`size`、`response_format`
- 流式模式限制 `n in [1,2]`

服务：`app/services/grok/services/image.py`
- 支持 WebSocket 图片流解析（中间图 + 最终图）
- `response_format=url` 时会落盘并转本地文件 URL
- `response_format=b64_json` 时直接返回 base64

### 3.2 图片编辑（`/v1/images/edits`）

- `ImageEditService` 先上传输入图，再调用上游编辑能力。
- 支持多图输入；内部会尽量找 parent_post_id 关联上下文。

---

## 4. 视频链路

服务：`app/services/grok/services/video.py`

- 提供文生视频和图生视频两条路径：
  - `generate`（text-to-video）
  - `generate_from_image`（image-to-video）
- 本质通过 post + `tool_overrides.videoGen` 启动上游生成
- 使用视频并发信号量，避免过载
- Token 选择时考虑分辨率/时长等需求

---

## 5. 错误处理与响应格式统一

- 所有业务异常统一继承 `AppException`
- `app/core/exceptions.py` 把异常映射为 OpenAI 风格 error JSON
- `RequestValidationError` 被转为更可读 message/param

---

## 6. 常见问题定位路径

- **429 太多**：看 `TokenManager.get_token*` 与 `mark_rate_limited`
- **流式中断**：看 `chat/image/video` 的 stream timeout 与 reverse 读取
- **图片无结果**：看 `ImageWS*Processor` 的 final/medium 图筛选阈值
- **401 激增**：看 token `record_fail`、fail threshold、状态变更
