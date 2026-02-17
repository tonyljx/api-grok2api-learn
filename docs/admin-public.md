# 管理后台与 Public 功能实现

## 1. 页面路由

- 后台页面：`app/api/pages/admin.py`
  - `/admin/login` `/admin/config` `/admin/cache` `/admin/token`
- Public 页面：`app/api/pages/public.py`
  - `/login` `/imagine` `/voice` `/video`
  - 受 `app.public_enabled` 控制

这些路由都直接返回 `app/static/**` 下的 HTML。

---

## 2. 管理 API（`app/api/v1/admin_api/*`）

### 2.1 Token 管理（`token.py`）

能力包括：
- 查询全部 token
- 批量更新 token（会做字段归一化和白名单过滤）
- 同步刷新 usage 状态
- 异步批量刷新（SSE 进度）
- 批量 NSFW 开关

实现亮点：
- 通过 `storage.acquire_lock("tokens_save")` 保证并发写安全
- 更新后会触发 `TokenManager.reload()`，让运行态立刻生效

### 2.2 配置管理（`config.py`）

- 读取/保存运行时配置
- 按段管理，避免覆盖默认配置基线

### 2.3 缓存管理（`cache.py`）

- 统计并清理临时文件缓存（图片/视频）
- 支撑后台“缓存页”管理功能

---

## 3. Public API（`app/api/v1/public_api/*`）

- `imagine.py`：偏“持续生成+前端实时展示”的交互，SSE 推送状态/图像事件
- `voice.py`：调用 `VoiceService` 获取 LiveKit token
- `video.py`：视频生成入口（与主 API 路由能力对齐）

Public 入口的核心价值：
- 提供轻量“玩法端”体验
- 把复杂参数封装在后端，前端只关心 prompt 与展示

---

## 4. 静态资源组织

- `app/static/admin/**`：后台 JS/CSS/页面
- `app/static/public/**`：Public JS/CSS/页面
- `app/static/common/**`：公共头尾、toast、鉴权脚本等

当你要改前端行为时，通常流程是：
1. 先改 `static/**/js/*.js`
2. 按需改 `api/v1/admin_api` 或 `api/v1/public_api`
3. 若涉及鉴权，再看 `core/auth.py`

---

## 5. 快速定位清单

- “后台 token 页面点刷新没反应” → `admin/static/admin/js/token.js` + `admin_api/token.py`
- “Public 页面 404” → `app.public_enabled` + `api/pages/public.py`
- “配置保存后不生效” → `admin_api/config.py` + `core/config.py` + 生命周期加载
