# 浏览器抓取任务失败处理机制深度分析

## 一、架构分层与职责边界

### 1.1 四层架构体系

| 层级 | 模块 | 职责定位 | 失败处理职责 |
|------|------|----------|--------------|
| **调度层** | `flask_app.py` (ticker_thread) | 定期检查watch，决定是否入队 | 无重试，仅按固定间隔调度 |
| **Worker层** | `worker.py` | 执行抓取任务，捕获异常 | 记录错误，触发通知，不重试 |
| **Processor层** | `processors/base.py` | 选择Fetcher，准备参数 | 透传异常，不处理重试 |
| **Fetcher层** | `content_fetchers/` | 实际HTTP请求/浏览器操作 | **仅requests有底层重试** |

### 1.2 各层核心职责边界

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        调度层 (Scheduler)                                   │
│  ticker_thread_check_time_launch_checks()                                  │
│  ├── 遍历所有watch                                                        │
│  ├── 判断条件: now - last_checked >= threshold + jitter                    │
│  ├── 入队: 优先级 = 当前时间戳                                            │
│  └── 无失败重试逻辑                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Worker层 (Executor)                                  │
│  async_update_worker()                                                    │
│  ├── 从队列获取任务                                                        │
│  ├── claim_uuid_for_processing()                                          │
│  ├── 调用processor.perform_site_check()                                    │
│  ├── 捕获所有异常 → 记录last_error                                         │
│  ├── 判断阈值 → 触发通知                                                   │
│  └── 不重试，等待下次调度周期                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Processor层 (Orchestrator)                           │
│  difference_detection_processor.call_browser()                             │
│  ├── 选择Fetcher类型 (requests/playwright)                                │
│  ├── 准备请求参数 (headers, timeout, proxy)                               │
│  ├── await self.fetcher.run()                                             │
│  └── 透传异常至Worker层                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Fetcher层 (Connector)                                │
│  ┌─────────────────┐     ┌─────────────────┐                              │
│  │  requests.py    │     │  playwright.py  │                              │
│  │  ├── urllib3    │     │  ├── 无重试     │                              │
│  │  │  Retry()     │     │  │ 机制          │                              │
│  │  │  内置重试    │     │  └── 直接抛出   │                              │
│  │  └── 底层网络   │     │      异常        │                              │
│  │      重试       │     └─────────────────┘                              │
│  └─────────────────┘                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、三类失败场景的真实传播路径

### 2.1 失败场景分类与处理矩阵

| 失败场景 | 触发条件 | Fetcher层处理 | Worker层处理 | 恢复机制 |
|----------|----------|--------------|-------------|----------|
| **网络抖动** | TCP连接重置、DNS解析失败 | requests重试(6次) | 无重试 | requests层自动恢复 |
| **超时** | connect/read timeout | requests重试(6次) | 记录`last_error` | requests层自动恢复 |
| **反爬拦截** | HTTP 403/429响应 | 不重试(status=0) | 记录`last_error` | 等待下次调度 |
| **页面不存在** | HTTP 404响应 | 不重试 | 记录`last_error` | 等待下次调度 |
| **服务器错误** | HTTP 5xx响应 | 不重试 | 记录`last_error` | 等待下次调度 |
| **浏览器连接失败** | Playwright连接超时 | 抛出`BrowserConnectError` | 记录`last_error` | 等待下次调度 |
| **浏览器步骤失败** | 元素定位失败 | 抛出`BrowserStepsStepException` | 记录+通知 | 等待下次调度 |

### 2.2 requests层重试配置详解

**配置位置**: `content_fetchers/requests.py:68-80`

```python
max_retries = int(os.getenv("REQUESTS_RETRY_MAX_COUNT", "6"))
retry_strategy = Retry(
    total=6,                    # 总重试次数
    connect=6,                  # 连接超时重试
    read=6,                     # 读取超时重试
    status=0,                   # HTTP状态码不重试
    backoff_factor=0.5,         # 指数退避因子
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],
    raise_on_status=False
)
```

**重试时间计算** (`backoff_factor=0.5`):

| 重试次数 | 等待时间 | 累计等待 |
|----------|----------|----------|
| 第1次 | 0.5 × 2^0 = 0.5s | 0.5s |
| 第2次 | 0.5 × 2^1 = 1.0s | 1.5s |
| 第3次 | 0.5 × 2^2 = 2.0s | 3.5s |
| 第4次 | 0.5 × 2^3 = 4.0s | 7.5s |
| 第5次 | 0.5 × 2^4 = 8.0s | 15.5s |
| 第6次 | 0.5 × 2^5 = 16.0s | 31.5s |

**重试覆盖范围**:

```
┌────────────────────────────────────────────────────────────────────┐
│                    requests层重试覆盖范围                          │
├────────────────────────────────────────────────────────────────────┤
│  ✓ TCP连接超时 (connect timeout)                                  │
│  ✓ 读取超时 (read timeout)                                        │
│  ✓ TCP连接重置 (connection reset by peer)                         │
│  ✓ DNS解析失败                                                    │
│  ✓ 连接被拒绝 (connection refused)                                │
│  ✗ HTTP 4xx错误 (反爬、认证失败等)                                │
│  ✗ HTTP 5xx错误 (服务器内部错误)                                  │
│  ✗ Playwright浏览器错误                                          │
└────────────────────────────────────────────────────────────────────┘
```

---

## 三、瞬时重试 vs 调度重试

### 3.1 职责划分

| 重试类型 | 负责层级 | 触发时机 | 重试策略 |
|----------|----------|----------|----------|
| **瞬时重试** | Fetcher层 (仅requests) | 抓取执行期间 | 指数退避，最多6次 |
| **调度重试** | 调度层 | 失败后下次周期 | 固定间隔(time_between_check) |

### 3.2 瞬时重试流程图（仅requests）

```
requests.fetcher.run()
        │
        ↓
   发起请求
        │
        ↓ (网络抖动/超时)
   ┌────┴────┐
   ↓         ↓
 失败      成功
   │         │
   ↓         ↓
urllib3     返回
 Retry()    内容
   │
   ↓ (重试计数 < 6)
 等待 backoff_factor × 2^attempt
   │
   ↓
 重试请求
   │
   ↓ (重试计数 >= 6)
 抛出异常 → Processor → Worker → 记录last_error
```

### 3.3 调度重试流程图

```
调度线程 ticker_thread_check_time_launch_checks()
        │
        ↓
   遍历所有watch
        │
        ↓
┌───────────────────────┐
│ now - last_checked   │
│ >= threshold + jitter │
└──────────┬──────────┘
           │ 是
           ↓
┌───────────────────────┐
│ uuid not in          │
│ running_uuids        │
│ and not in queued_uuids│
└──────────┬──────────┘
           │ 是
           ↓
   入队 (优先级=当前时间戳)
        │
        ↓
   Worker处理
        │
        ↓ (失败)
   记录last_error
        │
        ↓ (等待)
   下次调度周期(通常几分钟~几小时)
```

---

## 四、恢复与最终放弃的可判定条件

### 4.1 恢复判定条件

| 恢复类型 | 触发条件 | 恢复机制 |
|----------|----------|----------|
| **瞬时恢复** | requests重试成功 | 自动继续处理流程 |
| **调度恢复** | 下次调度周期到达 | 自动重新入队 |
| **手动恢复** | 用户手动触发重新检查 | 立即入队(优先级=1) |

### 4.2 错误状态清除条件

**位置**: `worker.py:388-398`

```python
else:
    # 抓取成功分支
    update_obj['last_error'] = False
    cleanup_error_artifacts(uuid, datastore)  # 清理错误截图等
```

**清除条件**:
1. `watch.get('ignore_status_codes')` 为 False 时，成功抓取会重置 `consecutive_filter_failures = 0`
2. 成功抓取会清除 `last_error` 并调用 `cleanup_error_artifacts()`

### 4.3 通知触发条件（降级机制）

**位置**: `worker.py:259-274` (过滤器失败) / `worker.py:324-335` (步骤失败)

```python
if watch.get('filter_failure_notification_send', False):
    c = watch.get('consecutive_filter_failures', 0)
    c += 1
    threshold = datastore.data['settings']['application'].get('filter_failure_notification_threshold_attempts', 0)
    if c >= threshold:
        if not watch.get('notification_muted'):
            await send_filter_failure_notification(uuid, notification_q, datastore)
        c = 0  # 发送后重置
    datastore.update_watch(uuid=uuid, update_obj={'consecutive_filter_failures': c})
```

**触发条件**:
1. `filter_failure_notification_send = True`
2. `consecutive_filter_failures >= filter_failure_notification_threshold_attempts`
3. `notification_muted = False`

### 4.4 当前架构的问题：缺乏"最终放弃"机制

```
当前行为:
失败 → 记录last_error → 下次调度重试 → 再次失败 → 无限循环

缺失的机制:
1. 失败次数上限
2. 失败率熔断
3. 动态退避延迟
```

---

## 五、失败信号传播路径详解

### 5.1 完整传播路径图

```
用户配置watch → 调度层入队 → Worker获取任务 → Processor选择Fetcher
                                                    │
                    ┌───────────────────────────────┴───────────────────────────────┐
                    ↓                                                             ↓
            requests fetcher                                               playwright fetcher
                    │                                                             │
                    ↓                                                             ↓
            ┌──────────────┐                                                直接执行
            │ urllib3重试  │                                                      │
            └──────┬───────┘                                                      ↓
                   │                                                          失败?
                   ↓                                                             │
            重试成功?                                                             ↓
                   │                                                          抛出异常
            ┌──────┴──────┐                                                          │
            ↓             ↓                                                          │
           成功         失败                                                          │
            │             │                                                          │
            │             ↓                                                          │
            │      抛出Exception                                                     │
            │             │                                                          │
            └──────┬──────┴───────────────────────────────┬───────────────────────────┘
                   │                                     │
                   ↓                                     ↓
            Processor透传                           Processor透传
                   │                                     │
                   └───────────────┬──────────────────────┘
                                   ↓
                          Worker捕获异常
                                   │
                                   ↓
                          记录last_error
                                   │
                                   ↓
                          判断通知阈值
                                   │
                          ┌────────┴────────┐
                          ↓                 ↓
                    达到阈值            未达阈值
                          │                 │
                          ↓                 ↓
                    发送通知            任务完成
                          │                 │
                          └────────┬────────┘
                                   ↓
                          等待下次调度周期
```

### 5.2 异常类型与传播路径

| 异常类型 | 触发层 | 传播路径 | Worker处理 |
|----------|--------|----------|------------|
| `Non200ErrorCodeReceived` | Fetcher | Fetcher→Processor→Worker | 记录状态码，保存截图 |
| `EmptyReply` | Fetcher | Fetcher→Processor→Worker | 记录错误，建议增加延迟 |
| `BrowserConnectError` | Fetcher | Fetcher→Processor→Worker | 记录连接错误 |
| `BrowserFetchTimedOut` | Fetcher | Fetcher→Processor→Worker | 记录超时错误 |
| `BrowserStepsStepException` | Fetcher | Fetcher→Processor→Worker | 记录步骤号，触发通知 |
| `FilterNotFoundInResponse` | Processor | Processor→Worker | 记录+累计失败计数+通知 |
| `PageUnloadable` | Fetcher | Fetcher→Processor→Worker | 记录错误，保存截图 |

---

## 六、监控记录机制

### 6.1 记录项汇总

| 记录项 | 存储位置 | 更新时机 | 用途 |
|--------|----------|----------|------|
| `last_error` | watch['last_error'] | 每次失败时 | 显示错误信息 |
| `last_check_status` | watch['last_check_status'] | 失败时 | 记录HTTP状态码 |
| `check_count` | watch['check_count'] | 每次检查时 | 统计检查次数 |
| `fetch_time` | watch['fetch_time'] | 每次检查时 | 记录耗时 |
| `consecutive_filter_failures` | watch['consecutive_filter_failures'] | 过滤器/步骤失败时 | 通知阈值判断 |
| `browser_steps_last_error_step` | watch['browser_steps_last_error_step'] | 步骤失败时 | 定位失败步骤 |

### 6.2 错误清除机制

```python
# worker.py:711-718
def cleanup_error_artifacts(uuid, datastore):
    """清除错误相关文件"""
    cleanup_files = ["last-error-screenshot.png", "last-error.txt"]
    for f in cleanup_files:
        full_path = os.path.join(datastore.datastore_path, uuid, f)
        if os.path.isfile(full_path):
            os.unlink(full_path)
```

---

## 七、优化建议

### 7.1 当前架构缺陷

| 问题 | 影响 | 严重程度 |
|------|------|----------|
| Playwright无重试 | 网络抖动导致的失败无法自动恢复 | 高 |
| 无失败次数上限 | 持续失败的watch无限重试 | 中 |
| 无熔断机制 | 大量失败任务占用资源 | 中 |
| 无动态退避 | 失败后按固定间隔重试，易触发反爬 | 中 |

### 7.2 建议的重试策略增强

```python
# 建议的失败重试策略
class EnhancedRetryPolicy:
    def __init__(self, max_retries=3, backoff_factor=2.0, max_delay=300):
        self.max_retries = max_retries
        self.backoff_factor = backoff_factor
        self.max_delay = max_delay
    
    def get_delay(self, attempt, error_type):
        """根据错误类型和重试次数计算延迟"""
        # 网络错误使用指数退避
        network_errors = (BrowserConnectError, BrowserFetchTimedOut)
        if isinstance(error_type, network_errors):
            delay = min(self.backoff_factor ** attempt, self.max_delay)
            return delay
        
        # 反爬错误使用更长延迟
        anti_crawl_errors = (Non200ErrorCodeReceived,)
        if isinstance(error_type, anti_crawl_errors):
            if error_type.status_code in [403, 429]:
                return min(self.backoff_factor ** (attempt + 2), self.max_delay)
        
        # 默认使用固定间隔
        return self.backoff_factor ** attempt
    
    def should_retry(self, error_type, attempt, consecutive_failures):
        """判断是否应该重试"""
        # 达到最大重试次数
        if attempt >= self.max_retries:
            return False
        
        # 达到连续失败上限
        if consecutive_failures >= 10:
            return False
        
        # 不可恢复错误
        unrecoverable = (BrowserStepsInUnsupportedFetcher,)
        if isinstance(error_type, unrecoverable):
            return False
        
        return True
```

---

## 八、总结

### 8.1 重试机制现状

```
┌─────────────────────────────────────────────────────────────────┐
│                     重试机制总结                                │
├─────────────────────────────────────────────────────────────────┤
│  Fetcher层 (requests):                                          │
│    ✓ 内置urllib3重试: 连接超时、读取超时、网络抖动                 │
│    ✗ 不重试HTTP状态码错误 (4xx/5xx)                             │
│                                                                │
│  Fetcher层 (playwright):                                        │
│    ✗ 无任何重试机制                                             │
│                                                                │
│  Worker层:                                                      │
│    ✗ 不做重试，仅记录错误                                       │
│                                                                │
│  调度层:                                                        │
│    ✓ 按固定间隔(time_between_check)重新调度                      │
│    ✗ 无动态退避、无熔断                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 三类失败场景的真实处理

| 场景 | 瞬时重试 | 调度重试 | 通知触发 | 恢复条件 |
|------|----------|----------|----------|----------|
| **网络抖动** | requests重试6次 | 是 | 否 | requests成功或下次调度 |
| **超时** | requests重试6次 | 是 | 否 | requests成功或下次调度 |
| **反爬拦截** | 否 | 是 | 是(达到阈值) | 用户介入或网站解除限制 |
| **服务器错误** | 否 | 是 | 是(达到阈值) | 服务器恢复 |
| **浏览器连接失败** | 否 | 是 | 是(达到阈值) | 浏览器服务恢复 |

### 8.3 关键结论

1. **瞬时重试责任边界**: 仅 `requests` fetcher 承担，覆盖底层网络错误
2. **调度重试责任边界**: 调度层承担，按固定间隔重新入队
3. **恢复判定**: 成功抓取自动清除错误状态，无需显式恢复操作
4. **最终放弃**: 当前架构**缺失**，需通过通知阈值间接实现告警

该系统采用"快速失败+延迟通知+定时重检"的策略，适合资源有限的场景，但在面对频繁网络抖动或严格反爬时可能需要增强重试和熔断机制。
