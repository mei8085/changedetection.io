# Changedetection.io 调度与抓取流程分析报告

> 本文档基于真实源码，准确描述从定时检查到入队再到 Worker 消费的完整路径，以及普通请求与浏览器抓取器的选择逻辑。

---

## 目录

1. [核心数据流总览](#核心数据流总览)
2. [调度器：ticker_thread_check_time_launch_checks](#调度器ticker_thread_check_time_launch_checks)
3. [优先级队列：RecheckPriorityQueue](#优先级队列recheckpriorityqueue)
4. [工作池与 Worker 消费](#工作池与-worker-消费)
5. [抓取器选择决策链](#抓取器选择决策链)
6. [抓取器注册与浏览器实现选择](#抓取器注册与浏览器实现选择)
7. [优先级策略详解](#优先级策略详解)
8. [异常处理与资源清理](#异常处理与资源清理)
9. [关键代码位置索引](#关键代码位置索引)
10. [配置项参考](#配置项参考)

---

## 核心数据流总览

```
[调度器 ticker_thread_check_time_launch_checks]
    ↓ (根据时间阈值、代理限制、定时调度等条件判定)
[优先级队列 RecheckPriorityQueue]
    ↓ (按优先级排序，最小堆实现)
[工作池 Worker Pool]
    ↓ (每个 Worker 有独立线程和事件循环)
[async_update_worker]
    ↓ (claim_uuid_for_processing → 防重复)
[difference_detection_processor]
    ↓ (call_browser 选择抓取器)
[抓取器 Fetcher]
    ├── html_requests    → requests 库，轻量 HTTP
    └── html_webdriver   → Playwright/Puppeteer/Selenium，浏览器渲染
```

---

## 调度器：ticker_thread_check_time_launch_checks

### 1. 真实函数名

**注意**: 真实函数名为 `ticker_thread_check_time_launch_checks`，不是 `ticker_thread_func`。

**文件**: `changedetectionio/flask_app.py:1107-1270`

**启动位置**: `flask_app.py:999`

```python
ticker_thread = threading.Thread(
    target=ticker_thread_check_time_launch_checks,
    daemon=True,
    name="TickerThread-ScheduleChecker"
).start()
```

### 2. 主循环结构

```python
def ticker_thread_check_time_launch_checks():
    proxy_last_called_time = {}
    last_health_check = 0
    recheck_time_minimum_seconds = int(os.getenv('MINIMUM_SECONDS_RECHECK_TIME', 3))
    WAIT_TIME_BETWEEN_LOOP = 1.0 if not IN_PYTEST else 0.01

    while not app.config.exit.is_set():
        # 2.1 Worker 健康检查（每 60 秒）
        # 2.2 检查是否全局暂停
        # 2.3 获取正在运行和已排队的 UUID
        # 2.4 遍历所有监控项，判定是否入队
        # 2.5 等待 WAIT_TIME_BETWEEN_LOOP 后再次循环
```

### 3. 入队判定条件（按顺序）

#### 3.1 全局暂停检查

```python
# flask_app.py:1141-1143
if datastore.data['settings']['application'].get('all_paused', False):
    app.config.exit.wait(1)
    continue
```

#### 3.2 获取运行中和已排队的 UUID

```python
# flask_app.py:1146-1149
running_uuids = worker_pool.get_running_uuids()
queued_uuids = {q_item.item['uuid'] for q_item in update_q.queue}
```

#### 3.3 遍历监控项的顺序

监控项按 `last_checked` 升序排列，最久未检查的优先被考虑：

```python
# flask_app.py:1157-1158
for k in sorted(datastore.data['watching'].items(), 
                key=lambda item: item[1].get('last_checked', 0)):
    watch_uuid_list.append(k[0])
```

#### 3.4 队列大小限制检查

```python
# flask_app.py:1171-1176
if watch_index % 100 == 0:
    current_queue_size = update_q.qsize()
    if current_queue_size >= MAX_QUEUE_SIZE:
        logger.debug(f"Queue size limit reached ({current_queue_size}/{MAX_QUEUE_SIZE}), stopping scheduler this iteration.")
        break
```

#### 3.5 监控项暂停检查

```python
# flask_app.py:1184-1186
if watch['paused']:
    continue
```

#### 3.6 定时调度限制（Time Schedule）

```python
# flask_app.py:1189-1213
# 选择监控项级或系统级的定时调度配置
if watch.get('time_between_check_use_default'):
    time_schedule_limit = datastore.data['settings']['requests'].get('time_schedule_limit', {})
else:
    time_schedule_limit = watch.get('time_schedule_limit')

if time_schedule_limit and time_schedule_limit.get('enabled'):
    result = is_within_schedule(
        time_schedule_limit=time_schedule_limit,
        default_tz=tz_name
    )
    if not result:
        continue  # 不在允许的时间段内，跳过
```

#### 3.7 时间阈值 + Jitter 检查

```python
# flask_app.py:1216-1226
threshold = recheck_time_system_seconds if watch.get('time_between_check_use_default') else watch.threshold_seconds()

jitter = datastore.data['settings']['requests'].get('jitter_seconds', 0)
if jitter > 0:
    if watch.jitter_seconds == 0:
        watch.jitter_seconds = random.uniform(-abs(jitter), jitter)

seconds_since_last_recheck = now - watch['last_checked']

if seconds_since_last_recheck >= (threshold + watch.jitter_seconds) and \
   seconds_since_last_recheck >= recheck_time_minimum_seconds:
    # 满足时间条件，继续检查其他条件
```

#### 3.8 运行中/已排队检查

```python
# flask_app.py:1227
if not uuid in running_uuids and uuid not in queued_uuids:
    # 继续检查代理限制
```

#### 3.9 代理复用时间限制

```python
# flask_app.py:1229-1246
watch_proxy = datastore.get_preferred_proxy_for_watch(uuid=uuid)
if watch_proxy and watch_proxy in list(datastore.proxy_list.keys()):
    proxy_list_reuse_time_minimum = int(
        datastore.proxy_list.get(watch_proxy, {}).get('reuse_time_minimum', 0)
    )
    if proxy_list_reuse_time_minimum:
        proxy_last_used_time = proxy_last_called_time.get(watch_proxy, 0)
        time_since_proxy_used = int(time.time() - proxy_last_used_time)
        if time_since_proxy_used < proxy_list_reuse_time_minimum:
            # 代理使用间隔不足，跳过
            continue
        else:
            # 记录本次使用时间
            proxy_last_called_time[watch_proxy] = int(time.time())
```

### 4. 最终入队

所有条件满足后，使用当前时间戳作为优先级入队：

```python
# flask_app.py:1248-1255
priority = int(time.time())

queued_successfully = worker_pool.queue_item_async_safe(
    update_q,
    queuedWatchMetaData.PrioritizedItem(
        priority=priority,
        item={'uuid': uuid}
    )
)

if queued_successfully:
    watch.jitter_seconds = 0  # 重置 jitter 供下次使用
```

### 5. 手动触发入队的入口

以下位置使用 `priority=1` 手动触发：

| 文件 | 行号 | 场景 |
|-----|-----|-----|
| `realtime/events.py` | 44 | 实时事件触发 |
| `blueprint/ui/views.py` | 41 | UI 视图操作 |
| `blueprint/ui/edit.py` | 277 | 编辑操作 |
| `blueprint/ui/__init__.py` | 66, 276, 305, 331 | 多种 UI 操作 |
| `blueprint/price_data_follower/__init__.py` | 24 | 价格数据跟踪 |
| `api/Watch.py` | 81, 554, 576 | API 调用 |
| `api/Tags.py` | 42, 50 | 标签 API |
| `__init__.py` | 439, 473, 535 | 启动/初始化 |

**克隆操作使用 `priority=5`**:

```python
# blueprint/ui/__init__.py:257
worker_pool.queue_item_async_safe(
    update_q,
    queuedWatchMetaData.PrioritizedItem(priority=5, item={'uuid': new_uuid})
)
```

---

## 优先级队列：RecheckPriorityQueue

### 1. 数据结构

**文件**: `changedetectionio/queuedWatchMetaData.py:7-10`

```python
@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any = field(compare=False)
```

- `priority` 参与比较（最小堆）
- `item` 不参与比较（`field(compare=False)`）

### 2. RecheckPriorityQueue 架构

**文件**: `changedetectionio/queue_handlers.py:15-411`

#### 2.1 设计目标

```
- 多异步 Worker，每个有独立事件循环和线程
- 混合同步/异步设计：
  - 同步接口：供 ticker 线程使用
  - 异步接口：供 Worker 使用
```

#### 2.2 内部数据结构

```python
# queue_handlers.py:41-62
def __init__(self, maxsize: int = 0):
    # 通知队列：用于信号机制
    self._notification_queue = queue.Queue(maxsize=maxsize if maxsize > 0 else 0)
    
    # 优先级存储：使用 heapq 维护最小堆
    self._priority_items = []
    
    # 线程锁：保证原子操作
    self._lock = threading.RLock()
```

### 3. 入队操作：put()

**文件**: `queue_handlers.py:64-100`

```python
def put(self, item, block: bool = True, timeout: Optional[float] = None):
    with self._lock:
        # 1. 原子性地添加到优先级堆
        heapq.heappush(self._priority_items, item)
        
        # 2. 添加通知到通知队列
        try:
            self._notification_queue.put(True, block=True, timeout=5.0)
        except Exception as notif_e:
            # 通知失败必须回滚，保持一致性
            self._priority_items.remove(item)
            heapq.heapify(self._priority_items)
            raise
    
    # 3. 发送信号（非关键路径，失败不影响入队）
    try:
        self._emit_put_signals(item)
    except Exception as signal_e:
        logger.error(f"Signal emission failed but item queued: {signal_e}")
    
    return True
```

**关键保证**: 优先级堆和通知队列的原子性一致性。

### 4. 出队操作：get()

**文件**: `queue_handlers.py:102-134`

```python
def get(self, block: bool = True, timeout: Optional[float] = None):
    # 1. 等待通知（不返回实际项目，只表示有项目可用）
    self._notification_queue.get(block=block, timeout=timeout)
    
    # 2. 获取最高优先级项目（priority 值最小）
    with self._lock:
        if not self._priority_items:
            logger.critical("Queue notification received but no priority items available")
            raise Exception("Priority queue inconsistency")
        item = heapq.heappop(self._priority_items)
    
    # 3. 发送信号
    try:
        self._emit_get_signals()
    except Exception as signal_e:
        logger.error(f"Get signal emission failed: {signal_e}")
    
    return item
```

### 5. 异步接口

#### 5.1 async_get() - Worker 使用

**文件**: `queue_handlers.py:161-202`

```python
async def async_get(self, executor=None, timeout=1.0):
    """
    使用 run_in_executor 调用同步 get()
    - 避免轮询开销
    - 超时时间由底层 queue.get() 控制，无双超时问题
    """
    loop = asyncio.get_event_loop()
    item = await loop.run_in_executor(
        executor,
        lambda: self.get(block=True, timeout=timeout)
    )
    return item
```

---

## 工作池与 Worker 消费

### 1. 工作池架构

**文件**: `changedetectionio/worker_pool.py`

#### 1.1 线程池配置

```python
# worker_pool.py:30-34
_max_executor_workers = int(os.getenv("FETCH_WORKERS", "10"))
queue_executor = ThreadPoolExecutor(
    max_workers=_max_executor_workers,
    thread_name_prefix="QueueGetter-"
)
```

#### 1.2 WorkerThread 类

```python
# worker_pool.py:37-95
class WorkerThread:
    def __init__(self, worker_id, update_q, notification_q, app, datastore):
        self.worker_id = worker_id
        self.update_q = update_q
        self.notification_q = notification_q
        self.app = app
        self.datastore = datastore
        self.thread = None
        self.loop = None
        self.running = False

    def run(self):
        # 每个 Worker 创建独立的事件循环
        self.loop = asyncio.new_event_loop()
        asyncio.set_event_loop(self.loop)
        self.running = True
        
        # 运行 worker 协程
        self.loop.run_until_complete(
            start_single_async_worker(
                self.worker_id,
                self.update_q,
                self.notification_q,
                self.app,
                self.datastore,
                queue_executor
            )
        )

    def start(self):
        self.thread = threading.Thread(
            target=self.run,
            daemon=True,
            name=f"PageFetchAsyncUpdateWorker-{self.worker_id}"
        )
        self.thread.start()
```

### 2. 单个 Worker 主循环

**文件**: `worker_pool.py:129-165`

```python
async def start_single_async_worker(worker_id, update_q, notification_q, app, datastore, executor=None):
    while not app.config.exit.is_set():
        try:
            result = await async_update_worker(
                worker_id, update_q, notification_q, app, datastore, executor
            )
            
            if result == "restart":
                continue  # Worker 请求重启
            else:
                break     # 正常退出
                
        except asyncio.CancelledError:
            break  # 任务被取消（正常关闭）
        except Exception as e:
            logger.error(f"Async worker {worker_id} crashed: {e}")
            await asyncio.sleep(5)  # 崩溃后 5 秒重启
```

### 3. async_update_worker 核心流程

**文件**: `changedetectionio/worker.py:23-708`

#### 3.1 从队列获取任务

```python
# worker.py:55-116
while not app.config.exit.is_set():
    try:
        # 1. 从队列获取任务
        queued_item_data = await q.async_get(executor=executor, timeout=1.0)
        
        # 2. 立即声明 UUID 所有权（防止竞态条件）
        uuid = queued_item_data.item.get('uuid')
        if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
            # 已被其他 Worker 处理，延迟后重新入队
            await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)
            deferred_priority = max(1000, queued_item_data.priority * 10)
            deferred_item = PrioritizedItem(
                priority=deferred_priority,
                item=queued_item_data.item
            )
            worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
            continue
            
    except asyncio.TimeoutError:
        # 队列空，检查是否需要重启
        continue
    except queue.Empty:
        # 正常超时，继续循环
        continue
```

#### 3.2 UUID 声明机制

**文件**: `worker_pool.py:218-257`

```python
def claim_uuid_for_processing(uuid, worker_id):
    """原子性地检查并声明 UUID 所有权"""
    with _uuid_processing_lock:
        if uuid in currently_processing_uuids:
            return False  # 已被其他 Worker 处理
        currently_processing_uuids[uuid] = worker_id
        logger.debug(f"Worker {worker_id} claimed UUID: {uuid}")
        return True

def release_uuid_from_processing(uuid, worker_id):
    """释放 UUID（线程安全）"""
    with _uuid_processing_lock:
        if currently_processing_uuids.get(uuid) == worker_id:
            currently_processing_uuids.pop(uuid, None)
```

#### 3.3 初始化处理器并调用抓取器

```python
# worker.py:121-175
fetch_start_time = round(time.time())

try:
    if uuid in list(datastore.data['watching'].keys()) and datastore.data['watching'][uuid].get('url'):
        watch = datastore.data['watching'].get(uuid)
        datastore.data['watching'][uuid]['last_checked'] = fetch_start_time
        
        # 1. 获取处理器模块
        processor = watch.get('processor', 'text_json_diff')
        processor_module = get_processor_module(processor)
        
        # 2. 创建处理器实例
        update_handler = processor_module.perform_site_check(
            datastore=datastore,
            watch_uuid=uuid
        )
        
        # 3. 允许插件修改处理器
        update_handler = apply_update_handler_alter(update_handler, watch, datastore)
        
        # 4. 调用抓取器（所有抓取器现在都是异步的）
        await update_handler.call_browser()
        
        # 5. 在线程池中运行变更检测（CPU 密集型）
        loop = asyncio.get_event_loop()
        changed_detected, update_obj, contents = await loop.run_in_executor(
            executor,
            lambda: update_handler.run_changedetection(watch=watch)
        )
```

---

## 抓取器选择决策链

### 1. 决策入口：call_browser()

**文件**: `changedetectionio/processors/base.py:117-260`

这是抓取器选择的核心决策点，决策链按以下顺序执行：

### 2. 完整决策链

#### 步骤 1: 初始获取 fetch_backend

```python
# base.py:133
prefer_fetch_backend = self.watch.get('fetch_backend', 'system')
```

| 值 | 含义 |
|---|-----|
| `system` | 使用系统全局设置 |
| `html_requests` | 强制使用 requests |
| `html_webdriver` | 强制使用浏览器 |
| `extra_browser_xxx` | 使用自定义浏览器连接 |
| 其他插件名 | 使用自定义插件抓取器 |

#### 步骤 2: 解析 `system` 值

```python
# base.py:140-141
if not prefer_fetch_backend or prefer_fetch_backend == 'system':
    prefer_fetch_backend = self.datastore.data['settings']['application'].get('fetch_backend')
```

#### 步骤 3: 处理自定义浏览器连接

```python
# base.py:145-152
custom_browser_connection_url = None
if prefer_fetch_backend.startswith('extra_browser_'):
    (t, key) = prefer_fetch_backend.split('extra_browser_')
    connection = list(filter(
        lambda s: s['browser_name'] == key,
        self.datastore.data['settings']['requests'].get('extra_browsers', [])
    ))
    if connection:
        prefer_fetch_backend = 'html_webdriver'
        custom_browser_connection_url = connection[0].get('browser_connection_url')
```

#### 步骤 4: PDF 强制切换到 html_requests

```python
# base.py:157-158
if self.watch.is_pdf:
    prefer_fetch_backend = "html_requests"
```

**原因**: Playwright 会将 PDF 渲染为内嵌页面，需要使用 requests + pdf2html 来正确提取文本。

#### 步骤 5: 浏览器步骤（Browser Steps）强制切换到 Playwright

```python
# base.py:160-174
from changedetectionio import content_fetchers

if hasattr(content_fetchers, prefer_fetch_backend):
    # 临时 HACK: 有浏览器步骤时强制使用 Playwright
    if prefer_fetch_backend == 'html_webdriver' and self.watch.has_browser_steps:
        logger.warning(
            "Using playwright fetcher override for possible puppeteer request in browsersteps, "
            "because puppetteer:browser steps is incomplete."
        )
        from changedetectionio.content_fetchers.playwright import fetcher as playwright_fetcher
        fetcher_obj = playwright_fetcher
    else:
        fetcher_obj = getattr(content_fetchers, prefer_fetch_backend)
else:
    # 找不到时回退到 html_requests
    fetcher_obj = getattr(content_fetchers, "html_requests")
```

### 3. Watch.get_fetch_backend 属性

**文件**: `changedetectionio/model/Watch.py:357-389`

```python
@property
def get_fetch_backend(self):
    """
    注意：这个属性只做了部分处理
    - 处理了 PDF → html_requests
    - 但没有处理 browser_steps → playwright
    - 实际的完整决策在 processors/base.py 的 call_browser() 中
    """
    if self.is_pdf:
        return 'html_requests'
    
    return self.get('fetch_backend')
```

**重要**: 这个属性不完整！实际的完整决策逻辑在 `processors/base.py:call_browser()` 中。

### 4. Watch 属性说明

#### 4.1 is_pdf

```python
# Watch.py:411-421
@property
def is_pdf(self):
    url = str(self.get("url") or "").lower()
    content_type = str(self.get("content-type") or "").lower()
    
    return (
        url.endswith(".pdf") or
        content_type.split(";")[0].strip() == "application/pdf"
    )
```

#### 4.2 has_browser_steps

```python
# Watch.py:499-504
@property
def has_browser_steps(self):
    has_browser_steps = self.get('browser_steps') and list(filter(
        lambda s: (s['operation'] and 
                   len(s['operation']) and 
                   s['operation'] != 'Choose one' and 
                   s['operation'] != 'Goto site'),
        self.get('browser_steps')))
    
    return has_browser_steps
```

### 5. 决策链总结

```
初始 fetch_backend
        │
        ▼
┌─────────────────┐
│  是 'system'?   │── Yes ──► 读取全局 settings.application.fetch_backend
└─────────────────┘
        │ No
        ▼
┌─────────────────┐
│ 以 extra_browser │── Yes ──► 解析为 html_webdriver + 自定义连接 URL
│   开头?         │
└─────────────────┘
        │ No
        ▼
┌─────────────────┐
│   is_pdf=True?  │── Yes ──► 强制切换为 html_requests
└─────────────────┘
        │ No
        ▼
┌─────────────────┐
│  fetch_backend  │
│ == html_webdriver│──┬── No ──► 直接使用配置的抓取器
│ AND             │  │
│ has_browser_steps│  └── Yes ──► 强制切换为 playwright
│ == True?        │
└─────────────────┘
        │
        ▼
   最终抓取器
```

### 6. 切换条件汇总表

| 条件 | 行为 | 优先级 |
|-----|-----|-----|
| `watch.is_pdf == True` | 强制使用 `html_requests` | **最高** |
| `watch.has_browser_steps == True` **且** `fetch_backend == html_webdriver` | 强制使用 `playwright` | **高** |
| `fetch_backend == system` | 使用全局 `settings.application.fetch_backend` | 中 |
| `fetch_backend` 不存在于 `content_fetchers` | 回退到 `html_requests` | 低 |

---

## 抓取器注册与浏览器实现选择

### 1. 抓取器注册

**文件**: `changedetectionio/content_fetchers/__init__.py:30-110`

#### 1.1 html_requests（始终可用）

```python
# __init__.py:30
from changedetectionio.content_fetchers.requests import fetcher as html_requests
```

#### 1.2 html_webdriver（动态选择）

```python
# __init__.py:93-105
use_playwright_as_chrome_fetcher = os.getenv('PLAYWRIGHT_DRIVER_URL', False)

if use_playwright_as_chrome_fetcher:
    if not strtobool(os.getenv('FAST_PUPPETEER_CHROME_FETCHER', 'False')):
        logger.debug('Using Playwright library as fetcher')
        from .playwright import fetcher as html_webdriver
    else:
        logger.debug('Using direct Python Puppeteer library as fetcher')
        from .puppeteer import fetcher as html_webdriver
else:
    logger.debug("Falling back to selenium as fetcher")
    from .webdriver_selenium import fetcher as html_webdriver
```

### 2. 浏览器实现选择逻辑

| 环境变量 | 条件 | 选择的浏览器实现 |
|---------|-----|---------------|
| `PLAYWRIGHT_DRIVER_URL` | **有值** 且 `FAST_PUPPETEER_CHROME_FETCHER=False` | Playwright |
| `PLAYWRIGHT_DRIVER_URL` | **有值** 且 `FAST_PUPPETEER_CHROME_FETCHER=True` | Puppeteer |
| `PLAYWRIGHT_DRIVER_URL` | **无值** | Selenium |

### 3. 插件抓取器

```python
# __init__.py:66-88
def get_plugin_fetchers():
    """加载所有插件抓取器"""
    from changedetectionio.pluggy_interface import plugin_manager
    
    fetchers = {}
    try:
        results = plugin_manager.hook.register_content_fetcher()
        for result in results:
            if result:
                name, fetcher_class = result
                fetchers[name] = fetcher_class
                # 注册到当前模块，使 hasattr() 检查生效
                setattr(sys.modules[__name__], name, fetcher_class)
    except Exception as e:
        logger.error(f"Error loading plugin fetchers: {e}")
    
    return fetchers

# 模块加载时初始化
_plugin_fetchers = get_plugin_fetchers()
```

### 4. 可用抓取器查询

```python
# __init__.py:39-63
def available_fetchers():
    """返回 UI 可选择的抓取器列表"""
    import inspect
    p = []
    
    # 内建抓取器（html_ 前缀）
    for name, obj in inspect.getmembers(sys.modules[__name__], inspect.isclass):
        if name.startswith('html_'):
            if name not in _plugin_fetchers:
                p.append((name, obj.fetcher_description))
    
    # 插件抓取器
    for name, fetcher_class in _plugin_fetchers.items():
        p.append((name, fetcher_class.fetcher_description))
    
    return p
```

---

## 优先级策略详解

### 1. 优先级值含义

| 优先级值 | 含义 | 触发场景 |
|---------|-----|---------|
| 1 | **最高优先级** | 手动触发、UI 操作、API 调用、实时事件 |
| 5 | **高优先级** | 克隆操作 |
| 时间戳 (>100) | **常规优先级** | 调度器自动触发（`int(time.time())`） |
| 1000+ | **延迟重试** | UUID 已被其他 Worker 处理时的重新入队 |

### 2. 最小堆排序说明

`heapq` 实现的是最小堆，所以：

- `priority=1` < `priority=5` < 时间戳 < `1000+`
- 数值越小，优先级越高
- 队列总是先出队 `priority` 最小的项目

### 3. 延迟重试

```python
# worker.py:74-76
deferred_priority = max(1000, queued_item_data.priority * 10)
deferred_item = PrioritizedItem(
    priority=deferred_priority,
    item=queued_item_data.item
)
worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
```

- 原优先级为时间戳（~17亿），乘以 10 后变为 ~170亿
- 使用 `max(1000, ...)` 确保至少为 1000
- 延迟任务会排到所有正常任务之后

---

## 异常处理与资源清理

### 1. Worker 异常处理

**文件**: `worker.py:177-386`

Worker 捕获的异常类型及处理：

| 异常类型 | 处理方式 |
|---------|---------|
| `PermissionError` | 文件权限错误，记录日志，跳过 |
| `ProcessorException` | 保存截图和 xpath 数据，更新 last_error |
| `ReplyWithContentButNoText` | 更新 last_error，可能保存截图 |
| `Non200ErrorCodeReceived` | 记录状态码错误，保存截图 |
| `FilterNotFoundInResponse` | 可能发送过滤器失败通知 |
| `BrowserStepsStepException` | 记录失败步骤位置 |
| `checksumFromPreviousCheckWasTheSame` | 无变更，不处理 |
| `BrowserConnectError` | 更新 last_error |
| `BrowserFetchTimedOut` | 更新 last_error |
| `EmptyReply` | 更新 last_error |
| `ScreenshotUnavailable` | 更新 last_error |
| `JSActionExceptions` | 保存截图，更新 last_error |
| `PageUnloadable` | 保存截图，更新 last_error |
| `BrowserStepsInUnsupportedFetcher` | 提示需要选择浏览器抓取器 |
| 其他 `Exception` | 记录完整异常栈，更新 last_error |

### 2. 资源清理

**文件**: `worker.py:605-674`

```python
finally:
    # 1. 调用抓取器 quit()
    try:
        if update_handler and hasattr(update_handler, 'fetcher') and update_handler.fetcher:
            await update_handler.fetcher.quit(watch=watch)
    except Exception as e:
        logger.error(f"Exception while cleaning/quit: {e}")
    
    # 2. 清理内存引用
    try:
        if update_handler:
            if hasattr(update_handler, 'fetcher') and update_handler.fetcher:
                update_handler.fetcher.clear_content()
            if hasattr(update_handler, 'content_processor'):
                update_handler.content_processor = None
            del update_handler
        
        if 'contents' in locals():
            del contents
        
        import gc
        gc.collect()  # 强制垃圾回收
    except Exception as cleanup_error:
        logger.error(f"Cleanup error: {cleanup_error}")
    
    # 3. 调用插件 finalize 钩子
    try:
        apply_update_finalize(...)
    except Exception as finalize_error:
        logger.error(f"Finalize hook error: {finalize_error}")
    
    # 4. 释放 UUID
    try:
        worker_pool.release_uuid_from_processing(uuid, worker_id=worker_id)
    except Exception as release_error:
        logger.error(f"Release UUID error: {release_error}")
```

### 3. Worker 重启策略

```python
# worker.py:44-45
max_jobs = int(os.getenv("WORKER_MAX_JOBS", "10"))
max_runtime_seconds = int(os.getenv("WORKER_MAX_RUNTIME", "3600"))

# worker.py:684-695
should_restart_jobs = jobs_processed >= max_jobs
should_restart_time = runtime >= max_runtime_seconds

if should_restart_jobs or should_restart_time:
    reason = f"{jobs_processed} jobs" if should_restart_jobs else f"{runtime:.0f}s runtime"
    logger.info(f"Worker {worker_id} restarting after {reason}")
    return "restart"
```

| 重启条件 | 默认值 | 说明 |
|---------|-------|-----|
| 任务数达到上限 | 10 个 | 防止内存泄漏累积 |
| 运行时间达到上限 | 3600 秒（1 小时） | 防止长时间运行 |

### 4. 工作池健康检查

**文件**: `worker_pool.py:498-553`

```python
def check_worker_health(expected_count, update_q=None, notification_q=None, app=None, datastore=None):
    alive_count = sum(1 for w in worker_threads if w.thread and w.thread.is_alive())
    
    if alive_count == expected_count:
        return {'status': 'healthy', ...}
    
    # 找出死亡的 Worker 并移除
    for i, worker in enumerate(worker_threads[:]):
        if not worker.thread or not worker.thread.is_alive():
            dead_workers.append(i)
            worker_threads.pop(i)
    
    # 重启缺失的 Worker
    if missing_workers > 0 and all([update_q, notification_q, app, datastore]):
        for i in range(missing_workers):
            add_worker(update_q, notification_q, app, datastore)
```

---

## 关键代码位置索引

| 功能 | 文件 | 行号范围 | 关键函数/类 |
|-----|-----|---------|-----------|
| 调度器主循环 | `changedetectionio/flask_app.py` | 1107-1270 | `ticker_thread_check_time_launch_checks` |
| 优先级队列 | `changedetectionio/queue_handlers.py` | 15-411 | `RecheckPriorityQueue` |
| 优先级项 | `changedetectionio/queuedWatchMetaData.py` | 7-10 | `PrioritizedItem` |
| 工作池管理 | `changedetectionio/worker_pool.py` | 完整文件 | `WorkerThread`, `claim_uuid_for_processing` |
| Worker 主逻辑 | `changedetectionio/worker.py` | 23-708 | `async_update_worker` |
| 抓取器选择 | `changedetectionio/processors/base.py` | 117-260 | `call_browser` |
| Watch 抓取后端 | `changedetectionio/model/Watch.py` | 357-504 | `get_fetch_backend`, `is_pdf`, `has_browser_steps` |
| 抓取器注册 | `changedetectionio/content_fetchers/__init__.py` | 30-110 | `available_fetchers`, `get_plugin_fetchers` |
| Requests 抓取器 | `changedetectionio/content_fetchers/requests.py` | 完整文件 | `fetcher` |
| Playwright 抓取器 | `changedetectionio/content_fetchers/playwright.py` | 完整文件 | `fetcher` |

---

## 配置项参考

### 1. 调度与队列配置

| 环境变量 | 默认值 | 说明 |
|---------|-------|-----|
| `MINIMUM_SECONDS_RECHECK_TIME` | 3 | 最小重检间隔秒数 |
| `FETCH_WORKERS` | 10 | Worker 线程数量 |
| `MAX_QUEUE_SIZE` | (代码定义) | 队列大小限制 |

### 2. Worker 配置

| 环境变量 | 默认值 | 说明 |
|---------|-------|-----|
| `WORKER_MAX_JOBS` | 10 | 每个 Worker 处理的最大任务数后重启 |
| `WORKER_MAX_RUNTIME` | 3600 | 每个 Worker 运行的最大秒数后重启 |

### 3. 请求配置

| 环境变量 | 默认值 | 说明 |
|---------|-------|-----|
| `REQUESTS_RETRY_MAX_COUNT` | 6 | Requests 抓取器最大重试次数 |
| `DEFAULT_FETCH_BACKEND` | `html_requests` | 默认抓取后端 |
| `ALLOW_IANA_RESTRICTED_ADDRESSES` | `false` | 是否允许内网地址 |
| `ALLOW_FILE_URI` | `false` | 是否允许 file:// 协议 |

### 4. 浏览器配置

| 环境变量 | 默认值 | 说明 |
|---------|-------|-----|
| `PLAYWRIGHT_DRIVER_URL` | 无 | Playwright WebSocket 连接 URL |
| `PLAYWRIGHT_BROWSER_TYPE` | `chromium` | 浏览器类型 |
| `FAST_PUPPETEER_CHROME_FETCHER` | `False` | 是否使用 Puppeteer 替代 Playwright |
| `WEBDRIVER_DELAY_BEFORE_CONTENT_READY` | 5 | 内容就绪前等待秒数 |
| `PLAYWRIGHT_SERVICE_WORKERS` | `allow` | 是否允许 Service Workers |

### 5. 截图配置

| 环境变量 | 默认值 | 说明 |
|---------|-------|-----|
| `SCREENSHOT_MAX_HEIGHT` | 20000 | 最大截图高度（像素） |
| `SCREENSHOT_CHUNK_HEIGHT` | 10000 | 分块截图高度阈值 |
| `SCREENSHOT_QUALITY` | 72 | JPEG 截图质量 |

---

## 总结

### 1. 完整路径回顾

```
ticker_thread_check_time_launch_checks()
  │
  ├─ 遍历所有 watch，按 last_checked 排序
  ├─ 检查：paused → time_schedule → threshold+jitter → running/queued → proxy_reuse
  └─ 满足条件后：priority=int(time.time()), queue_item_async_safe()
        │
        ▼
RecheckPriorityQueue
  │
  ├─ _notification_queue (信号)
  ├─ _priority_items (最小堆)
  └─ put/get 原子操作（RLock 保护）
        │
        ▼
async_update_worker()
  │
  ├─ async_get() → claim_uuid_for_processing() → 防止重复
  ├─ 创建 difference_detection_processor
  └─ call_browser() → 选择抓取器
        │
        ▼
抓取器选择决策链
  │
  ├─ fetch_backend = watch.get('fetch_backend', 'system')
  ├─ 'system' → 全局 settings.application.fetch_backend
  ├─ is_pdf → 强制 html_requests
  ├─ has_browser_steps + html_webdriver → 强制 playwright
  └─ 最终实例化 fetcher_obj
        │
        ▼
fetcher.run() → fetcher.quit()
```

### 2. 关键设计决策

1. **最小堆优先级队列**: 使用 `heapq` 实现，数值越小优先级越高
2. **UUID 声明机制**: `claim_uuid_for_processing()` 防止多个 Worker 同时处理同一任务
3. **抓取器决策在 call_browser()**: 而非 Watch 属性，确保决策的完整性
4. **PDF 强制 requests**: Playwright 对 PDF 渲染不适用
5. **Browser Steps 强制 playwright**: Puppeteer 的浏览器步骤实现不完整
6. **Worker 重启策略**: 防止内存泄漏，每 10 个任务或 1 小时重启

### 3. 常见误区修正

| 误区 | 事实 |
|-----|-----|
| 调度函数名为 `ticker_thread_func` | 实为 `ticker_thread_check_time_launch_checks` |
| `Watch.get_fetch_backend` 是完整决策 | 只处理了 PDF，browser_steps 的处理在 `call_browser()` 中 |
| `html_webdriver` 总是使用 Playwright | 取决于 `PLAYWRIGHT_DRIVER_URL`，可能是 Puppeteer 或 Selenium |
| `fetch_backend='system'` 是特殊抓取器 | 只是占位符，实际解析为全局设置的值 |
