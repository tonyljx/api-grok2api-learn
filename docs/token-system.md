# Token 池与调度机制

## 1. 为什么这个模块是核心

这个项目不是单 token 直连，而是 **多池多 token 调度系统**。
你能否稳定调用，取决于：
- token 是否可用（active/cooling/expired）
- 剩余额度是否足够
- 是否被限流（429）
- 多 worker 下状态是否及时同步

---

## 2. 数据模型（`app/services/token/models.py`）

`TokenInfo` 关键字段：
- `token`：实际凭证（保存时会归一化去掉 `sso=` 前缀）
- `quota`：剩余额度
- `status`：`active/disabled/expired/cooling`
- `fail_count`：401 连续失败计数
- `use_count/last_used_at`：使用统计
- `last_sync_at`：最近刷新时间

行为函数：
- `record_fail()`：仅 401 计入；超过阈值标记 expired
- `record_success()`：清 fail，按 quota 决定 active/cooling
- `need_refresh()`：判断 cooling token 是否到刷新周期

---

## 3. TokenPool 选择策略（`app/services/token/pool.py`）

选择规则非常直观：
1. 只选 `active 且 quota>0`
2. 先找 quota 最大的一组
3. 再在同额度中随机，减少并发冲突

这意味着：
- 高额度 token 会优先被消费
- 相同额度下流量会自然打散

---

## 4. TokenManager 主流程（`app/services/token/manager.py`）

### 4.1 启动加载

- 从存储加载 token 数据；若远端空，则尝试用本地 `data/token.json` 回填。
- 为每个池构建 `TokenPool`。
- 标记 initialized + reload 时间戳。

### 4.2 一致性与保存

- `reload_if_stale()`：多 worker 下按 `token.reload_interval_sec` 周期重新拉取。
- `_schedule_save()`：写入防抖，合并高频修改，减少 I/O。
- `_save()`：通过存储锁保护 `save_tokens` 原子性。

### 4.3 状态变更

- `consume()`：按模型 effort 扣 quota，更新状态
- `mark_rate_limited()`：标记限流并延后重试
- `refresh_cooling_tokens()`：批量拉 usage，同步恢复或过期状态

---

## 5. 自动刷新调度（`app/services/token/scheduler.py`）

- 启动时根据配置启用 Scheduler。
- 循环周期 = 配置小时数。
- 为防止多实例重复刷新：
  - Redis 存储走分布式锁
  - 其他存储走 `acquire_lock("token_refresh")`

执行内容：
1. 拿锁
2. 调用 `TokenManager.refresh_cooling_tokens()`
3. 输出 checked/refreshed/recovered/expired 统计
4. 休眠下一轮

---

## 6. 你最常改的地方（扩展建议）

- 增加新的池路由策略：改 `ModelService.pool_candidates_for_model`
- 调整失败阈值：`config.defaults.toml` 的 `token.fail_threshold`
- 调整同步频率：`token.reload_interval_sec`
- 控制写盘频率：`token.save_delay_ms`

---

## 7. 排障速查

- 症状：明明有 token 但总是 429
  - 查池状态是否全是 cooling/disabled/expired
  - 查是否被 `mark_rate_limited` 长时间冻结
- 症状：多 worker 数据不一致
  - 查 `reload_if_stale` 是否关闭/过大
  - 查存储锁是否生效
- 症状：token 大量变 expired
  - 查上游 401 与 fail_threshold
