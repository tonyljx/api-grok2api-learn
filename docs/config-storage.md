# 配置、存储与一致性机制

## 1. 配置系统（`app/core/config.py`）

配置来源分两层：
1. `config.defaults.toml`：代码仓默认基线
2. `data/config.toml`：运行时覆盖（可由后台修改）

加载策略是“**深度合并**”：运行时配置覆盖默认值，未配置项自动继承默认值。

另外实现里还有：
- 废弃配置节迁移（兼容旧版结构）
- 未知字段过滤与日志提醒
- 全局 `get_config(path, default)` 读取接口

---

## 2. 存储抽象（`app/core/storage.py`）

`BaseStorage` 统一定义：
- `load/save_config`
- `load/save_tokens`
- `acquire_lock`
- `check_connection`

实现包括：
- `LocalStorage`
- `RedisStorage`
- `MySQLStorage`
- `PostgresStorage`

### 2.1 Local 特性

- 配置文件 TOML 写入
- token 文件 JSON 原子写（临时文件 + `os.replace`）
- 锁策略：`asyncio.Lock` +（可用时）`fcntl.flock`

### 2.2 Redis/DB 特性

- Redis 用 hash/set 建模，支持分布式锁
- MySQL/Postgres 基于 SQLAlchemy 异步连接
- 存储工厂按环境变量选择实现

---

## 3. 生命周期中的配置/存储时机

`main.py` 启动流程：
1. 注册 grok 默认配置
2. `config.load()` 合并配置
3. 启动 token refresh scheduler（可开关）

关闭流程：
- 关闭存储连接
- 停止 scheduler

---

## 4. 认证与权限控制（`app/core/auth.py`）

- `/v1/*` 通过 `verify_api_key` 校验 Bearer（若 `app.api_key` 非空）
- `/v1/admin/*` 通过 `verify_app_key` 校验后台密码
- `/v1/public/*` 通过 `public_enabled/public_key` 控制开放与鉴权

注意：如果对应 key 为空，认证默认可关闭（用于本地开发）。

---

## 5. 异常处理（`app/core/exceptions.py`）

统一映射为 OpenAI 风格：
- `AppException`：业务可控错误
- `HTTPException`：HTTP 级错误映射默认 code
- `RequestValidationError`：参数校验错误优化展示
- `Exception`：兜底 500

这保证了上层路由可以“抛语义异常”，而不是到处手写响应结构。

---

## 6. 建议的配置修改流程

1. 先改 `config.defaults.toml`（定义新默认）
2. 再改 `app/core/config.py`（如涉及迁移/兼容）
3. 最后补充管理接口展示（若需要在线编辑）

这样能确保：本地、Docker、远端存储都行为一致。
