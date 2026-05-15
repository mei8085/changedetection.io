# 失败处置链路深度复核报告

## 一、失败类型与阈值通知触发条件

### 1.1 失败分类汇总

| 失败类型 | 异常类 | 累计计数 | 触发阈值通知 | 代码位置 |
|----------|--------|----------|--------------|----------|
| **过滤器未找到** | `FilterNotFoundInResponse` | ✓ | ✓ | worker.py:241-278 |
| **浏览器步骤失败** | `BrowserStepsStepException` | ✓ | ✓ | worker.py:301-337 |
| **HTTP非200响应** | `Non200ErrorCodeReceived` | ✗ | ✗ | worker.py:216-239 |
| **空响应** | `EmptyReply` | ✗ | ✗ | worker.py:339-344 |
| **浏览器连接失败** | `BrowserConnectError` | ✗ | ✗ | worker.py:291-294 |
| **浏览器抓取超时** | `BrowserFetchTimedOut` | ✗ | ✗ | worker.py:296-299 |
| **页面无法加载** | `PageUnloadable` | ✗ | ✗ | worker.py:361-373 |
| **截图不可用** | `ScreenshotUnavailable` | ✗ | ✗ | worker.py:346-350 |
| **JS执行失败** | `JSActionExceptions` | ✗ | ✗ | worker.py:352-359 |
| **浏览器步骤不支持** | `BrowserStepsInUnsupportedFetcher` | ✗ | ✗ | worker.py:375-379 |
| **处理器异常** | `ProcessorException` | ✗ | ✗ | worker.py:182-190 |
| **内容无文本** | `ReplyWithContentButNoText` | ✗ | ✗ | worker.py:192-214 |
| **权限错误** | `PermissionError` | ✗ | ✗ | worker.py:177-180 |
| **其他异常** | `Exception` | ✗ | ✗ | worker.py:381-386 |

### 1.2 触发阈值通知的失败类型详解

**(1) FilterNotFoundInResponse**

```python
# worker.py:259-274
if watch.get('filter_failure_notification_send', False):
    c = watch.get('consecutive_filter_failures', 0)
    c += 1
    threshold = datastore.data['settings']['application'].get('filter_failure_notification_threshold_attempts', 0)
    if c >= threshold:
        if not watch.get('notification_muted'):
            await send_filter_failure_notification(uuid, notification_q, datastore)
        c = 0  # 发送后重置计数器
    datastore.update_watch(uuid=uuid, update_obj={'consecutive_filter_failures': c})
```

**触发条件**：
1. `filter_failure_notification_send = True`
2. `consecutive_filter_failures >= filter_failure_notification_threshold_attempts`
3. `notification_muted = False`

**(2) BrowserStepsStepException**

```python
# worker.py:324-335
if watch.get('filter_failure_notification_send', False):
    c = watch.get('consecutive_filter_failures', 0)
    c += 1
    threshold = datastore.data['settings']['application'].get('filter_failure_notification_threshold_attempts', 0)
    if threshold > 0 and c >= threshold:
        if not watch.get('notification_muted'):
            await send_step_failure_notification(watch_uuid=uuid, step_n=e.step_n, ...)
        c = 0
    datastore.update_watch(uuid=uuid, update_obj={'consecutive_filter_failures': c})
```

**触发条件**：
1. `filter_failure_notification_send = True`
2. `filter_failure_notification_threshold_attempts > 0`
3. `consecutive_filter_failures >= threshold`
4. `notification_muted = False`

### 1.3 计数重置条件

**位置**: `worker.py:394-395`

```python
if not watch.get('ignore_status_codes'):
    update_obj['consecutive_filter_failures'] = 0
```

**重置时机**：抓取成功且 `ignore_status_codes = False`

---

## 二、手动recheck与定时调度重试的入队条件差异

### 2.1 入队条件对比

| 条件项 | 手动recheck (priority=1) | 定时调度 (priority=timestamp) |
|--------|--------------------------|-------------------------------|
| **触发方式** | API调用 `/api/v1/watch?recheck_all=1` | ticker_thread定时扫描 |
| **优先级** | `priority=1`（最高） | `priority=int(time.time())`（按时间排序） |
| **时间间隔检查** | 无 | `now - last_checked >= threshold + jitter` |
| **最小间隔检查** | 无 | `now - last_checked >= MINIMUM_SECONDS_RECHECK_TIME` |
| **运行状态检查** | ✓ (不在running_uuids) | ✓ (不在running_uuids) |
| **队列状态检查** | ✓ (不在queued_uuids) | ✓ (不在queued_uuids) |
| **暂停状态检查** | 无 | ✓ (watch['paused'] == False) |
| **时间调度限制** | 无 | ✓ (is_within_schedule) |
| **代理复用限制** | 无 | ✓ (proxy reuse_time_minimum) |

### 2.2 手动recheck入队逻辑

**位置**: `api/Watch.py:536-589`

```python
# 同步入队（<20个watch）
for uuid in watches_to_queue_filtered:
    worker_pool.queue_item_async_safe(self.update_q, 
        queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))

# 异步入队（>=20个watch）
def queue_all_watches_background():
    for uuid in watches_to_queue:
        if uuid not in queued_uuids and uuid not in running_uuids:
            worker_pool.queue_item_async_safe(self.update_q, 
                queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
```

**入队条件**：
1. `uuid not in queued_uuids`
2. `uuid not in running_uuids`

### 2.3 定时调度入队逻辑

**位置**: `flask_app.py:1107-1268`

```python
# 完整入队条件链
if seconds_since_last_recheck >= (threshold + watch.jitter_seconds) \
   and seconds_since_last_recheck >= recheck_time_minimum_seconds:
    if not uuid in running_uuids and uuid not in queued_uuids:
        # 代理复用时间检查
        if watch_proxy:
            if time_since_proxy_used >= proxy_list_reuse_time_minimum:
                # 入队
                priority = int(time.time())
                queuedWatchMetaData.PrioritizedItem(priority=priority, item={'uuid': uuid})
```

**入队条件**：
1. `seconds_since_last_recheck >= threshold + jitter`
2. `seconds_since_last_recheck >= MINIMUM_SECONDS_RECHECK_TIME`
3. `uuid not in running_uuids`
4. `uuid not in queued_uuids`
5. `watch['paused'] == False`
6. `is_within_schedule()` 返回 True（如配置了时间调度限制）
7. `time_since_proxy_used >= proxy_list_reuse_time_minimum`（如配置了代理）

### 2.4 优先级机制

```python
# 手动recheck - 最高优先级
priority = 1

# 定时调度 - 时间戳优先级（自然排序）
priority = int(time.time())

# 延迟队列 - 提升优先级
deferred_priority = max(1000, queued_item_data.priority * 10)
```

**优先级规则**：
- `priority=1`：立即执行（手动触发）
- `priority=timestamp`：按时间顺序执行（定时调度）
- `priority=1000+`：延迟重试（内部调度）

---

## 三、最终放弃机制分析

### 3.1 当前实现状态

**结论**：当前架构**无硬终止阈值**，持续失败的watch会无限循环重试。

### 3.2 持续失败但继续周期调度的判定依据

```
调度线程 ticker_thread_check_time_launch_checks()
        │
        ↓
   遍历所有watch
        │
        ↓
┌───────────────────────────────────────────┐
│              入队条件检查链                │
├───────────────────────────────────────────┤
│ 1. watch['paused'] == False              │ ← 未暂停
│ 2. is_within_schedule() == True          │ ← 在时间窗口内
│ 3. seconds_since_last_recheck >=         │ ← 达到检查间隔
│    threshold + jitter                    │
│ 4. seconds_since_last_recheck >=         │ ← 满足最小间隔
│    MINIMUM_SECONDS_RECHECK_TIME          │
│ 5. uuid not in running_uuids             │ ← 未在运行
│ 6. uuid not in queued_uuids              │ ← 未在队列
│ 7. time_since_proxy_used >=              │ ← 代理复用时间满足
│    proxy_list_reuse_time_minimum         │
└──────────────┬────────────────────────────┘
               │ 全部满足
               ↓
          入队调度
```

### 3.3 无终止阈值的影响

```
当前行为：
失败 → 记录last_error → 下次调度周期 → 再次失败 → 无限循环

影响：
1. 持续失败的watch消耗系统资源
2. 可能触发目标站点反爬机制
3. 用户无法感知任务已"放弃"
4. 无熔断机制保护系统稳定性
```

### 3.4 当前架构的"放弃"等价机制

| 机制 | 实现方式 | 是否自动 |
|------|----------|----------|
| **用户暂停** | 手动设置 `watch['paused'] = True` | 否 |
| **通知告警** | 达到 `consecutive_filter_failures` 阈值 | 是 |
| **静音通知** | 设置 `watch['notification_muted'] = True` | 否 |

### 3.5 代码层面验证

**调度线程检查逻辑** (`flask_app.py:1185-1264`)：

```python
# 仅检查暂停状态，不检查失败次数
if watch['paused']:
    continue

# 不检查 last_error 或 consecutive_filter_failures
# 不检查失败历史

# 仅基于时间间隔判断是否入队
if seconds_since_last_recheck >= (threshold + watch.jitter_seconds) \
   and seconds_since_last_recheck >= recheck_time_minimum_seconds:
    # 入队...
```

**Worker层异常处理逻辑** (`worker.py:177-386`)：

```python
# 所有异常处理都只是记录错误，不修改暂停状态
except SomeException as e:
    datastore.update_watch(uuid=uuid, update_obj={'last_error': err_text})
    process_changedetection_results = False
    # 不设置暂停标志，不阻止下次调度
```

---

## 四、总结

### 4.1 失败分类与通知触发

| 类别 | 失败类型 | 计数行为 | 通知行为 |
|------|----------|----------|----------|
| **阈值通知类** | 过滤器未找到、浏览器步骤失败 | 累计 `consecutive_filter_failures` | 达到阈值时发送 |
| **仅记录类** | HTTP错误、连接失败、超时等 | 不计数 | 不发送 |

### 4.2 入队条件差异

| 维度 | 手动recheck | 定时调度 |
|------|-------------|----------|
| 优先级 | 1（最高） | timestamp（时序） |
| 时间检查 | 无 | 检查间隔+抖动 |
| 状态检查 | 仅运行/队列状态 | 暂停+时间窗口+代理限制 |
| 触发方式 | API调用 | 定时扫描 |

### 4.3 最终放弃机制

**当前状态**：无硬终止阈值，持续失败会无限循环重试

**判定依据**：
- 仅检查暂停状态 `watch['paused']`
- 不检查 `last_error` 或失败次数
- 无熔断、无最大重试次数限制

**改进建议**：
1. 添加 `max_consecutive_failures` 配置项
2. 达到阈值时自动暂停watch或延长检查间隔
3. 添加失败率熔断机制
