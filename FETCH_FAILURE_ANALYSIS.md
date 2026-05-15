# 浏览器抓取任务失败处理机制分析报告

## 一、调用链架构概览

浏览器抓取任务的调用链分为三层，各层职责明确：

| 层级 | 模块 | 职责 |
|------|------|------|
| **Worker层** | `worker.py` | 任务调度、异常捕获、资源清理、通知触发 |
| **Processor层** | `processors/base.py` | 变更检测逻辑、Fetcher选择、参数准备 |
| **Fetcher层** | `content_fetchers/playwright.py` / `requests.py` | 实际HTTP请求、浏览器自动化、异常抛出 |

---

## 二、失败信号传播路径

### 2.1 调用链完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Worker层 (worker.py)                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ async_update_worker()                                              │    │
│  │   ├── 从队列获取任务                                                │    │
│  │   ├── claim_uuid_for_processing()                                  │    │
│  │   └── processor_module.perform_site_check() ───────────────────────┼────┼───→ 异常捕获
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                   ↓                                        │
│                           Processor层 (processors/base.py)                  │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ difference_detection_processor.call_browser()                       │    │
│  │   ├── 选择Fetcher类型                                              │    │
│  │   ├── 准备请求参数                                                 │    │
│  │   └── await self.fetcher.run() ────────────────────────────────────┼────┼───→ 异常捕获
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                   ↓                                        │
│                           Fetcher层 (playwright.py / requests.py)            │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ fetcher.run()                                                      │    │
│  │   ├── 建立浏览器连接                                                │    │
│  │   ├── 执行页面请求                                                 │    │
│  │   ├── 处理响应                                                     │    │
│  │   └── 抛出异常 ────────────────────────────────────────────────────┼────┘
│  └─────────────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 异常类型与触发场景

| 异常类型 | 触发场景 | 所属模块 |
|----------|----------|----------|
| `Non200ErrorCodeReceived` | HTTP响应码非200 | content_fetchers |
| `EmptyReply` | 响应内容为空 | content_fetchers |
| `PageUnloadable` | 页面无法正确加载 | content_fetchers |
| `BrowserConnectError` | 浏览器连接失败 | content_fetchers |
| `BrowserFetchTimedOut` | 浏览器抓取超时 | content_fetchers |
| `BrowserStepsStepException` | 浏览器步骤执行失败 | content_fetchers |
| `ScreenshotUnavailable` | 截图获取失败 | content_fetchers |
| `ReplyWithContentButNoText` | 获取到HTML但无可用文本 | content_fetchers |
| `FilterNotFoundInResponse` | CSS/XPath过滤器未找到 | processors |

---

## 三、各层职责划分

### 3.1 Worker层职责

**位置**: `changedetectionio/worker.py`

**核心职责**:
1. **任务调度**: 从优先级队列获取任务，管理UUID处理状态
2. **异常捕获**: 捕获所有来自Processor/Fetcher层的异常
3. **错误记录**: 将错误信息存储到watch对象的`last_error`字段
4. **通知触发**: 根据异常类型和阈值条件触发通知
5. **资源清理**: 确保浏览器连接、页面等资源正确释放
6. **指标记录**: 更新`check_count`、`fetch_time`等统计信息

**关键代码逻辑** (`worker.py:177-386`):
```python
# 异常捕获与处理示例
except content_fetchers_exceptions.Non200ErrorCodeReceived as e:
    err_text = f"Error - Request returned a HTTP error code {e.status_code}"
    if e.screenshot:
        watch.save_screenshot(screenshot=e.screenshot, as_error=True)
    datastore.update_watch(uuid=uuid, update_obj={'last_error': err_text})
    process_changedetection_results = False
```

### 3.2 Processor层职责

**位置**: `changedetectionio/processors/base.py`

**核心职责**:
1. **Fetcher选择**: 根据配置选择合适的内容获取器
2. **参数准备**: 准备请求头、超时时间、代理等参数
3. **请求执行**: 调用Fetcher的`run()`方法执行抓取
4. **SSRF防护**: 验证目标URL是否为私有IP地址
5. **内容后处理**: 清理Unicode代理字符等

**关键代码逻辑** (`processors/base.py:117-261`):
```python
async def call_browser(self, preferred_proxy_id=None):
    # 选择Fetcher类型
    prefer_fetch_backend = self.watch.get('fetch_backend', 'system')
    
    # 创建Fetcher实例
    self.fetcher = fetcher_obj(proxy_override=proxy_url,
                               custom_browser_connection_url=custom_browser_connection_url,
                               screenshot_format=self.screenshot_format)
    
    # 执行抓取
    await self.fetcher.run(...)
```

### 3.3 Fetcher层职责

**位置**: `changedetectionio/content_fetchers/playwright.py` / `requests.py`

**核心职责**:
1. **底层请求**: 执行实际的HTTP请求或浏览器操作
2. **异常抛出**: 根据不同失败场景抛出对应异常
3. **内容提取**: 获取页面内容、截图、favicon等
4. **连接管理**: 管理浏览器连接的建立和关闭
5. **重试处理**: requests fetcher内置底层重试机制

**requests层重试配置** (`requests.py:68-80`):
```python
max_retries = int(os.getenv("REQUESTS_RETRY_MAX_COUNT", "6"))
retry_strategy = Retry(
    total=max_retries,
    connect=max_retries,    # 连接超时重试
    read=max_retries,       # 读取超时重试
    status=0,               # 不重试HTTP状态码
    backoff_factor=0.5,     # 指数退避: 0.3s, 0.6s, 1.2s...
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],
    raise_on_status=False
)
```

---

## 四、状态转移图

```
                    ┌──────────────────┐
                    │   任务入队        │
                    └────────┬─────────┘
                             │
                             ↓
                    ┌──────────────────┐
                    │   Worker获取任务  │
                    └────────┬─────────┘
                             │
                             ↓
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐          ┌────────▼────────┐
     │  Processor层    │          │  Processor层    │
     │  call_browser() │          │  call_browser() │
     └────────┬────────┘          └────────┬────────┘
              │                             │
              ↓                             ↓
     ┌────────▼────────┐          ┌────────▼────────┐
     │  Fetcher层      │          │  Fetcher层      │
     │  run() 成功     │          │  run() 失败     │
     └────────┬────────┘          └────────┬────────┘
              │                             │
              ↓                             ↓
     ┌────────▼────────┐          ┌────────▼────────┐
     │ 变更检测        │          │ 异常抛出        │
     │ run_changedetection()      │ Exception      │
     └────────┬────────┘          └────────┬────────┘
              │                             │
              ↓                             ↓
     ┌────────▼────────┐          ┌────────▼────────┐
     │ 检测到变更?     │          │ Worker捕获异常   │
     └────────┬────────┘          │ 记录last_error  │
              │                   └────────┬────────┘
         ┌────┴────┐                       │
         │         │                       ↓
    ┌────▼────┐ ┌──▼──────┐      ┌────────▼────────┐
    │ 是      │ │ 否      │      │ 通知触发判断    │
    └────┬────┘ └────┬─────┘      └────────┬────────┘
         │           │                      │
         ↓           ↓                      ↓
    ┌────▼────┐ ┌────▼────┐      ┌────────▼────────┐
    │发送通知  │ │任务完成 │      │ 达到阈值?       │
    └─────────┘ └─────────┘      └────────┬────────┘
                                          │
                                    ┌──────┴──────┐
                                    │             │
                               ┌────▼────┐  ┌────▼────┐
                               │ 是      │  │ 否      │
                               └────┬────┘  └────┬────┘
                                    │             │
                                    ↓             ↓
                               ┌────▼────┐  ┌────▼────┐
                               │发送通知  │  │任务完成 │
                               └─────────┘  └─────────┘
```

---

## 五、重试机制分析

### 5.1 当前重试策略

| 层级 | 是否有重试 | 重试类型 | 配置方式 |
|------|-----------|----------|----------|
| **Worker层** | 否 | - | - |
| **Processor层** | 否 | - | - |
| **Fetcher层 (requests)** | 是 | 底层网络重试 | `REQUESTS_RETRY_MAX_COUNT`环境变量 |
| **Fetcher层 (playwright)** | 否 | - | - |

### 5.2 requests层重试配置详情

```python
# 配置参数
max_retries = int(os.getenv("REQUESTS_RETRY_MAX_COUNT", "6"))

# 重试策略
retry_strategy = Retry(
    total=6,                    # 总重试次数
    connect=6,                  # 连接超时重试次数
    read=6,                     # 读取超时重试次数
    status=0,                   # HTTP状态码不重试
    backoff_factor=0.5,         # 退避因子
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"]
)

# 退避时间计算 (backoff_factor=0.5):
# 第1次重试: 0.5 * 2^0 = 0.5s
# 第2次重试: 0.5 * 2^1 = 1.0s
# 第3次重试: 0.5 * 2^2 = 2.0s
# ...
```

### 5.3 重试覆盖场景

| 场景 | requests层是否重试 | playwright层是否重试 |
|------|------------------|---------------------|
| 连接超时 | 是 | 否 |
| 读取超时 | 是 | 否 |
| TCP连接重置 | 是 | 否 |
| DNS解析失败 | 是 | 否 |
| HTTP 5xx错误 | 否 | 否 |
| HTTP 4xx错误 | 否 | 否 |
| 反爬拦截 | 否 | 否 |

---

## 六、通知降级机制

### 6.1 过滤器失败通知

**触发条件** (`worker.py:259-274`):
```python
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

### 6.2 浏览器步骤失败通知

**触发条件** (`worker.py:324-335`):
```python
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

### 6.3 通知层级优先级

```
Individual watch settings > Tag settings > Global settings

notification_urls
notification_title  
notification_body
notification_format
```

---

## 七、监控记录机制

### 7.1 记录内容

| 记录项 | 存储位置 | 更新时机 |
|--------|----------|----------|
| `last_error` | watch['last_error'] | 每次失败时 |
| `last_check_status` | watch['last_check_status'] | 每次检查时 |
| `check_count` | watch['check_count'] | 每次检查时 |
| `fetch_time` | watch['fetch_time'] | 每次检查时 |
| `consecutive_filter_failures` | watch['consecutive_filter_failures'] | 过滤器失败时 |
| `browser_steps_last_error_step` | watch['browser_steps_last_error_step'] | 步骤失败时 |

### 7.2 日志记录

```python
# Worker层日志
logger.info(f"Worker {worker_id} processing watch UUID {uuid}")
logger.error(f"Worker {worker_id} exception processing watch UUID: {uuid}")

# Fetcher层日志
logger.error(f"Browser connection error {msg}")
logger.error(f"Browser processing took too long - {msg}")

# Processor层日志
logger.debug(f"Using proxy '{proxy_url}' for {self.watch['uuid']}")
```

---

## 八、关键判断条件汇总

### 8.1 异常处理决策点

| 判断条件 | 位置 | 决策行为 |
|----------|------|----------|
| `e.status_code == 403` | worker.py:217 | 记录"Access denied"错误 |
| `e.status_code == 404` | worker.py:219 | 记录"Page not found"错误 |
| `watch.get('filter_failure_notification_send')` | worker.py:259 | 是否发送过滤器失败通知 |
| `c >= threshold` | worker.py:265 | 是否达到通知阈值 |
| `watch.get('notification_muted')` | worker.py:266 | 是否静音通知 |
| `empty_pages_are_a_change` | playwright.py:359 | 空页面是否视为变更 |
| `ignore_status_codes` | playwright.py:354 | 是否忽略非200状态码 |

### 8.2 恢复/放弃决策

```
失败 → 记录错误 → 增加失败计数 → 判断阈值 → 发送通知(达到阈值) → 继续下次检查

无恢复机制: 当前实现没有自动重试失败任务的机制，失败后仅记录状态，等待下一个调度周期
```

---

## 九、问题与优化建议

### 9.1 当前架构存在的问题

1. **缺乏上层重试机制**: 仅requests层有底层重试，playwright和业务层无重试
2. **无指数退避**: 失败任务按固定周期重试，可能触发反爬
3. **无熔断机制**: 持续失败的任务仍会被频繁调度
4. **失败任务无优先级调整**: 失败后重试优先级未调整

### 9.2 优化建议

```python
# 建议的重试机制伪代码
class RetryPolicy:
    def __init__(self, max_retries=3, backoff_factor=2.0):
        self.max_retries = max_retries
        self.backoff_factor = backoff_factor
    
    def get_delay(self, attempt):
        """计算第N次重试的延迟时间"""
        return self.backoff_factor ** attempt
    
    def should_retry(self, error_type, attempt):
        """判断是否应该重试"""
        retryable_errors = [
            BrowserConnectError,
            BrowserFetchTimedOut,
            EmptyReply
        ]
        return attempt < self.max_retries and isinstance(error_type, tuple(retryable_errors))
```

---

## 十、总结

| 维度 | 当前状态 |
|------|----------|
| **失败信号传播** | 三层链式传播: Fetcher→Processor→Worker |
| **重试机制** | 仅requests层有底层网络重试，无业务层重试 |
| **通知降级** | 支持连续失败阈值触发，支持静音设置 |
| **监控记录** | 记录last_error、check_count、fetch_time等指标 |
| **恢复策略** | 无自动恢复，依赖下次调度周期 |

该系统的失败处理机制侧重于**快速失败、记录状态、延迟通知**，通过阈值机制避免频繁通知，但缺乏主动重试和熔断机制。
