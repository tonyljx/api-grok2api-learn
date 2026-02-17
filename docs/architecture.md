# 架构总览：模块分层与职责边界

## 1. 总体分层

项目可以看成 5 层：

1. **入口层**（`main.py`）
   - 初始化配置/日志/生命周期
   - 注册 API 路由、中间件、异常处理
   - 启动 Token 自动刷新调度

2. **协议层**（`app/api/**`）
   - 对外提供 OpenAI 兼容接口 + 管理接口 + Public 接口
   - 负责请求参数验证与响应格式转换

3. **业务编排层**（`app/services/grok/services/**`）
   - Chat / Image / ImageEdit / Video / Voice 的主要业务逻辑
   - 统一调用 TokenManager、模型映射、流式包装

4. **上游通信层**（`app/services/reverse/**`）
   - 直接处理与 Grok Web 的 HTTP/WS/GraphQL 交互
   - 承载 Retry、Headers、Statsig 等细节

5. **基础设施层**（`app/core/**` + `app/services/token/**`）
   - 配置、认证、异常、日志、存储抽象
   - Token 池管理、配额刷新与并发协调

---

## 2. 关键目录速查

- `app/api/v1/`：OpenAI 兼容 API（chat/images/models/files）
- `app/api/v1/admin_api/`：后台管理 API（token/config/cache）
- `app/api/v1/public_api/`：公共玩法 API（imagine/voice/video）
- `app/services/grok/services/`：业务服务核心
- `app/services/reverse/`：Reverse 调用实现
- `app/services/token/`：Token 选择、状态与调度
- `app/core/`：通用底座（config/storage/auth/exceptions）
- `app/static/`：前端静态资源（admin/public）

---

## 3. 一次请求的跨层路径（以 chat 为例）

`/v1/chat/completions` 请求大致是：

1. 路由层校验参数、判定模型类型（文本/图片/视频）
2. 从 TokenManager 中按模型候选池获取可用 token
3. 进入对应 Service（ChatService/ImageService/VideoService）
4. Service 组装上游入参，调用 `reverse` 层发起请求
5. 如果是流式：按 SSE chunk 透传/重写；结束时记录 usage
6. 如果失败：按 429/401 等策略进行 token 切换与状态更新
7. 返回 OpenAI 风格响应

---

## 4. 设计亮点（对快速定位很有帮助）

- **模型中心化**：`ModelService` 统一维护 model_id -> 能力/计费/池候选。
- **流式 usage 延迟记账**：`wrap_stream_with_usage` 在流结束后扣减额度，避免半程失败错误记账。
- **存储可替换**：`StorageFactory` 抽象 local/redis/mysql/pgsql，一套代码支持多部署环境。
- **多 worker 一致性策略**：`reload_if_stale + 分布式锁` 避免 token 状态漂移和刷新重复。

---

## 5. 你最该先看的“十个文件”

1. `main.py`
2. `app/api/v1/chat.py`
3. `app/services/grok/services/chat.py`
4. `app/services/grok/services/image.py`
5. `app/services/grok/services/video.py`
6. `app/services/grok/services/model.py`
7. `app/services/token/manager.py`
8. `app/services/token/models.py`
9. `app/core/config.py`
10. `app/core/storage.py`
