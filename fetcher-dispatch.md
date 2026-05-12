# Changedetection.io 调度与抓取流程分析报告

## 目录

1. [概述](#概述)
2. [调度系统架构](#调度系统架构)
3. [任务入队机制](#任务入队机制)
4. [队列管理系统](#队列管理系统)
5. [工作池与消费者模型](#工作池与消费者模型)
6. [抓取器选择逻辑](#抓取器选择逻辑)
7. [抓取执行流程](#抓取执行流程)
8. [优先级策略详解](#优先级策略详解)
9. [异常处理与恢复机制](#异常处理与恢复机制)
10. [关键代码位置索引](#关键代码位置索引)

---

## 概述

Changedetection.io 的监控页面从调度到抓取的完整流程是一个典型的生产者-消费者模型。整个系统由以下核心组件组成：

- **调度器(Ticker)**：定期检查监控任务，决定哪些任务需要入队
- **优先级队列(PriorityQueue)**：管理待执行任务的排队顺序
- **工作池(Worker Pool)**：管理多个异步工作线程
- **工作线程(Worker)**：从队列获取任务并执行抓取
- **抓取器(Fetcher)**：根据任务配置选择不同的抓取策略

整个流程的数据流如下：

```
[调度器/Ticker] 
    ↓ (根据时间阈值决定)
[优先级队列]
    ↓ (按优先级排序)
[工作池/Worker Pool]
    ↓ (分发到空闲Worker)
[工作线程/Worker]
    ↓ (调用处理器)
[处理器/Processor]
    ↓ (选择抓取器)
[抓取器/Fetcher] → [普通HTTP请求] 或 [浏览器渲染]
```

---

## 调度系统架构

### 1. 调度器实现位置

调度器的核心实现在 `flask_app.py` 中，通过一个独立的 ticker 线程运行。

### 2. 调度器主循环

**文件**: `flask_app.py` (约 1150-1270 行)

调度器在一个无限循环中执行以下操作：

```python
def ticker_thread_func(app, datastore, update_q, exit_flag):
    while not exit_flag.is_set():
        # 1. 遍历所有监控项
        for uuid, watch in datastore.data['watching'].items():
            # 2. 检查是否需要入队
            if should_enqueue(watch):
                # 3. 加入优先级队列
                queue_item(watch)
        
        # 4. 等待一段时间后再次检查
        exit_flag.wait(WAIT_TIME_BETWEEN_LOOP)
```

### 3. 入队判定条件

调度器决定是否将监控任务入队需要满足以下条件：

#### 3.1 时间阈值检查

```python
# 阈值计算逻辑 (flask_app.py:1216-1226)
threshold = recheck_time_system_seconds if watch.get('time_between_check_use_default') else watch.threshold_seconds()
jitter = datastore.data['settings']['requests'].get('jitter_seconds', 0)
seconds_since_last_recheck = now - watch['last_checked']

if seconds_since_last_recheck >= (threshold + watch.jitter_seconds) and seconds_since_last_recheck >= recheck_time_minimum_seconds:
    # 满足时间条件，继续检查其他条件
```

#### 3.2 状态检查

- 监控项不能已经在运行中 (`uuid not in running_uuids`)
- 监控项不能已经在队列中 (`uuid not in queued_uuids`)

#### 3.3 代理限制检查

```python
# 代理复用时间限制检查 (flask_app.py:1229-1246)
watch_proxy = datastore.get_preferred_proxy_for_watch(uuid=uuid)
if watch_proxy and watch_proxy in list(datastore.proxy_list.keys()):
    proxy_list_reuse_time_minimum = int(datastore.proxy_list.get(watch_proxy, {}).get('reuse_time_minimum', 0))
    if proxy_list_reuse_time_minimum:
        proxy_last_used_time = proxy_last_called_time.get(watch_proxy, 0)
        time_since_proxy_used = int(time.time() - proxy_last_used_time)
        if time_since_proxy_used < proxy_list_reuse_time_minimum:
            # 代理使用间隔不足，跳过
            continue
```

#### 3.4 定时调度检查

```python
# 时间调度检查 (flask_app.py:1200-1213)
if time_schedule_limit and time_schedule_limit.get('enabled'):
    result = is_within_schedule(time_schedule_limit=time_schedule_limit,
                                default_tz=tz_name)
    if not result:
        # 不在允许的时间段内，跳过
        continue
```

---

## 任务入队机制

### 1. 入队入口点

任务可以通过多个入口点加入队列：

#### 1.1 调度器自动入队 (Ticker)

```python
# flask_app.py:1248-1254
priority = int(time.time())  # 使用当前时间戳作为优先级
queued_successfully = worker_pool.queue_item_async_safe(update_q,
    queuedWatchMetaData.PrioritizedItem(priority=priority, item={'uuid': uuid})
)
```

#### 1.2 手动触发入队

- **UI 操作**: `blueprint/ui/__init__.py`, `blueprint/ui/edit.py`
- **API 操作**: `api/Watch.py`, `api/Tags.py`
- **实时事件**: `realtime/events.py`

```python
# 手动触发通常使用最高优先级 (priority=1)
worker_pool.queue_item_async_safe(update_q,
    queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid})
)
```

#### 1.3 克隆操作入队

```python
# blueprint/ui/__init__.py:257
# 克隆操作使用优先级 5
worker_pool.queue_item_async_safe(update_q,
    queuedWatchMetaData.PrioritizedItem(priority=5, item={'uuid': new_uuid})
)
```

### 2. 优先级数据结构

**文件**: `queuedWatchMetaData.py`

```python
@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any=field(compare=False)
```

使用 `dataclass` 配合 `order=True` 实现优先级比较，`item` 字段不参与比较。

### 3. 线程安全入队

**文件**: `worker_pool.py:290-339`

```python
def queue_item_async_safe(update_q, item, silent=False):
    """Bulletproof queue operation with comprehensive error handling"""
    
    # 1. 提取 UUID 用于日志
    item_uuid = extract_uuid(item)
    
    # 2. 验证输入
    if not update_q or not item:
        logger.critical("Queue or item is None/invalid")
        return False
    
    # 3. 执行入队操作
    try:
        success = update_q.put(item, block=True, timeout=5.0)
        if success is False:
            logger.critical("Queue.put() returned False")
            return False
        return True
    except Exception as e:
        # 4. 错误处理和健康检查
        logger.critical(f"Queue operation failed: {e}")
        log_queue_health(update_q)
        return False
```

---

## 队列管理系统

### 1. 队列实现演进

系统使用了改进的 `RecheckPriorityQueue` 实现，替代了原来的简单 `PriorityQueue`。

### 2. RecheckPriorityQueue 架构

**文件**: `queue_handlers.py:15-411`

#### 2.1 核心设计理念

```python
class RecheckPriorityQueue:
    """
    Thread-safe priority queue supporting multiple async event loops.
    
    ARCHITECTURE:
    - Multiple async workers, each with its own event loop in its own thread
    - Hybrid sync/async design for maximum scalability
    - Sync interface for ticker thread (threading.Queue)
    - Async interface for workers (asyncio.Event)
    """
```

#### 2.2 内部数据结构

```python
def __init__(self, maxsize: int = 0):
    # 1. 通知队列：用于信号机制
    self._notification_queue = queue.Queue(maxsize=maxsize if maxsize > 0 else 0)
    
    # 2. 优先级存储：使用 heapq 维护最小堆
    self._priority_items = []
    
    # 3. 线程锁：保证原子操作
    self._lock = threading.RLock()
```

### 3. 入队操作 (put)

**文件**: `queue_handlers.py:64-100`

```python
def put(self, item, block: bool = True, timeout: Optional[float] = None):
    """Thread-safe sync put with priority ordering"""
    
    with self._lock:
        # 1. 原子性地添加到优先级堆
        heapq.heappush(self._priority_items, item)
        
        # 2. 添加通知到通知队列
        try:
            self._notification_queue.put(True, block=True, timeout=5.0)
        except Exception as notif_e:
            # 通知失败必须回滚
            self._priority_items.remove(item)
            heapq.heapify(self._priority_items)
            raise
    
    # 3. 发送信号（非关键路径，失败不影响入队）
    try:
        self._emit_put_signals(item)
    except Exception as signal_e:
        logger.error(f"Signal emission failed but item queued: {signal_e}")
```

### 4. 出队操作 (get)

**文件**: `queue_handlers.py:102-134`

```python
def get(self, block: bool = True, timeout: Optional[float] = None):
    """Thread-safe sync get with priority ordering"""
    
    # 1. 等待通知（不返回实际项目，只表示有项目可用）
    self._notification_queue.get(block=block, timeout=timeout)
    
    # 2. 获取最高优先级项目
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

#### 5.1 async_put

**文件**: `queue_handlers.py:136-159`

```python
async def async_put(self, item, executor=None):
    """Async put with priority ordering - uses thread pool to avoid blocking"""
    
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(
        executor,
        lambda: self.put(item, block=True, timeout=5.0)
    )
    return result
```

#### 5.2 async_get

**文件**: `queue_handlers.py:161-202`

```python
async def async_get(self, executor=None, timeout=1.0):
    """
    Efficient async get using executor for blocking call.
    
    HYBRID APPROACH: Best of both worlds
    - Uses run_in_executor for efficient blocking
    - Single timeout (no double-timeout wrapper)
    - Scales well: executor sized to match worker count
    """
    
    loop = asyncio.get_event_loop()
    item = await loop.run_in_executor(
        executor,
        lambda: self.get(block=True, timeout=timeout)
    )
    return item
```

### 6. 队列查询功能

#### 6.1 获取队列状态

```python
def qsize(self) -> int:
    """Get current queue size"""
    with self._lock:
        return len(self._priority_items)

def empty(self) -> bool:
    """Check if queue is empty"""
    return self.qsize() == 0
```

#### 6.2 查找特定 UUID 位置

**文件**: `queue_handlers.py:271-299`

```python
def get_uuid_position(self, target_uuid: str) -> Dict[str, Any]:
    """Find position of UUID in queue"""
    
    with self._lock:
        queue_list = list(self._priority_items)
        
        # 查找目标项目
        for item in queue_list:
            if item.item.get('uuid') == target_uuid:
                # 计算位置：统计优先级更高的项目数
                position = sum(1 for other in queue_list if other.priority < item.priority)
                return {
                    'position': position,
                    'total_items': len(queue_list),
                    'priority': item.priority,
                    'found': True
                }
        
        return {'position': None, 'total_items': len(queue_list), 'found': False}
```

#### 6.3 获取队列摘要

**文件**: `queue_handlers.py:336-376`

```python
def get_queue_summary(self) -> Dict[str, Any]:
    """Get queue summary statistics"""
    
    with self._lock:
        queue_list = list(self._priority_items)
        
        immediate_items = clone_items = scheduled_items = 0
        priority_counts = {}
        
        for item in queue_list:
            priority = item.priority
            priority_counts[priority] = priority_counts.get(priority, 0) + 1
            
            if priority == 1:
                immediate_items += 1      # 立即执行
            elif priority == 5:
                clone_items += 1          # 克隆操作
            elif priority > 100:
                scheduled_items += 1      # 定时调度（时间戳）
        
        return {
            'total_items': len(queue_list),
            'priority_breakdown': priority_counts,
            'immediate_items': immediate_items,
            'clone_items': clone_items,
            'scheduled_items': scheduled_items
        }
```

---

## 工作池与消费者模型

### 1. 工作池架构

**文件**: `worker_pool.py`

#### 1.1 线程池配置

```python
# 全局配置
_max_executor_workers = int(os.getenv("FETCH_WORKERS", "10"))
queue_executor = ThreadPoolExecutor(
    max_workers=_max_executor_workers,
    thread_name_prefix="QueueGetter-"
)

# Worker 线程列表
worker_threads = []  # List of WorkerThread objects

# 当前处理中的 UUID 映射
currently_processing_uuids = {}
_uuid_processing_lock = threading.Lock()
```

#### 1.2 WorkerThread 类

**文件**: `worker_pool.py:37-107`

```python
class WorkerThread:
    """Container for a worker thread with its own event loop"""
    
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
        """Run the worker in its own event loop"""
        # 创建独立的事件循环
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
```

### 2. 工作线程启动

**文件**: `worker_pool.py:109-126`

```python
def start_async_workers(n_workers, update_q, notification_q, app, datastore):
    """Start async workers, each with its own thread and event loop"""
    
    logger.info(f"Starting {n_workers} async workers (isolated threads)")
    for i in range(n_workers):
        try:
            worker = WorkerThread(i, update_q, notification_q, app, datastore)
            worker.start()
            worker_threads.append(worker)
        except Exception as e:
            logger.error(f"Failed to start async worker {i}: {e}")
            continue
```

### 3. 单个 Worker 主循环

**文件**: `worker_pool.py:129-164`

```python
async def start_single_async_worker(worker_id, update_q, notification_q, app, datastore, executor=None):
    """Start a single async worker with auto-restart capability"""
    
    while not app.config.exit.is_set():
        try:
            result = await async_update_worker(worker_id, update_q, notification_q, app, datastore, executor)
            
            if result == "restart":
                # Worker 请求重启
                continue
            else:
                # 正常退出
                break
                
        except asyncio.CancelledError:
            # 任务被取消（正常关闭）
            break
        except Exception as e:
            # 崩溃后 5 秒重启
            logger.error(f"Async worker {worker_id} crashed: {e}")
            await asyncio.sleep(5)
```

### 4. 任务处理主流程

**文件**: `worker.py:23-708`

#### 4.1 从队列获取任务

```python
async def async_update_worker(worker_id, q, notification_q, app, datastore, executor=None):
    while not app.config.exit.is_set():
        try:
            # 1. 从队列获取任务（阻塞等待）
            queued_item_data = await q.async_get(executor=executor, timeout=1.0)
            
            # 2. 立即声明 UUID 所有权（防止竞态条件）
            uuid = queued_item_data.item.get('uuid')
            if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
                # 已被其他 worker 处理，延迟后重新入队
                await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)
                deferred_priority = max(1000, queued_item_data.priority * 10)
                deferred_item = PrioritizedItem(priority=deferred_priority, item=queued_item_data.item)
                worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
                continue
                
        except asyncio.TimeoutError:
            # 队列空，检查是否需要重启
            continue
        except Exception as e:
            # 异常处理
            continue
```

#### 4.2 UUID 声明机制

**文件**: `worker_pool.py:218-257`

```python
def claim_uuid_for_processing(uuid, worker_id):
    """
    Atomically check if UUID is available and claim it for processing.
    
    Returns:
        True if successfully claimed
        False if already being processed by another worker
    """
    with _uuid_processing_lock:
        if uuid in currently_processing_uuids:
            return False
        currently_processing_uuids[uuid] = worker_id
        return True

def release_uuid_from_processing(uuid, worker_id):
    """Release a UUID from processing (thread-safe)"""
    with _uuid_processing_lock:
        if currently_processing_uuids.get(uuid) == worker_id:
            currently_processing_uuids.pop(uuid, None)
```

---

## 抓取器选择逻辑

### 1. 抓取器选择入口

**文件**: `processors/base.py:117-260`

```python
async def call_browser(self, preferred_proxy_id=None):
    """选择并调用合适的抓取器"""
    
    # 1. 获取首选抓取后端
    prefer_fetch_backend = self.watch.get('fetch_backend', 'system')
    
    # 2. 解析 'system' 值
    if not prefer_fetch_backend or prefer_fetch_backend == 'system':
        prefer_fetch_backend = self.datastore.data['settings']['application'].get('fetch_backend')
```

### 2. 特殊情况处理

#### 2.1 自定义浏览器连接

```python
# processors/base.py:145-152
custom_browser_connection_url = None
if prefer_fetch_backend.startswith('extra_browser_'):
    (t, key) = prefer_fetch_backend.split('extra_browser_')
    connection = list(
        filter(lambda s: (s['browser_name'] == key), 
               self.datastore.data['settings']['requests'].get('extra_browsers', [])))
    if connection:
        prefer_fetch_backend = 'html_webdriver'
        custom_browser_connection_url = connection[0].get('browser_connection_url')
```

#### 2.2 PDF 文件强制使用 Requests

```python
# processors/base.py:157-158
if self.watch.is_pdf:
    prefer_fetch_backend = "html_requests"
```

### 3. 抓取器获取逻辑

```python
# processors/base.py:160-174
from changedetectionio import content_fetchers

if hasattr(content_fetchers, prefer_fetch_backend):
    # 特殊处理：浏览器步骤强制使用 Playwright
    if prefer_fetch_backend == 'html_webdriver' and self.watch.has_browser_steps:
        logger.warning("Using playwright fetcher override for browsersteps")
        from changedetectionio.content_fetchers.playwright import fetcher as playwright_fetcher
        fetcher_obj = playwright_fetcher
    else:
        fetcher_obj = getattr(content_fetchers, prefer_fetch_backend)
else:
    # 找不到时回退到 requests
    fetcher_obj = getattr(content_fetchers, "html_requests")
```

### 4. Watch 模型中的抓取后端解析

**文件**: `model/Watch.py:356-390`

```python
@property
def get_fetch_backend(self):
    """
    Get the fetch backend for this watch with special case handling.
    
    CHAIN RESOLUTION:
    - Watch override → Tag override → Global settings (future Pydantic implementation)
    """
    # PDF 强制使用 requests
    if self.is_pdf:
        return 'html_requests'
    
    return self.get('fetch_backend')
```

### 5. 抓取器注册机制

**文件**: `content_fetchers/__init__.py:39-110`

#### 5.1 可用抓取器查询

```python
def available_fetchers():
    """Returns list of available fetchers for UI selection"""
    import inspect
    p = []
    
    # 1. 内建抓取器（html_ 前缀）
    for name, obj in inspect.getmembers(sys.modules[__name__], inspect.isclass):
        if name.startswith('html_'):
            if name not in _plugin_fetchers:
                p.append((name, obj.fetcher_description))
    
    # 2. 插件抓取器
    for name, fetcher_class in _plugin_fetchers.items():
        p.append((name, fetcher_class.fetcher_description))
    
    return p
```

#### 5.2 浏览器抓取器选择

```python
# content_fetchers/__init__.py:91-106
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

---

## 抓取执行流程

### 1. 抓取器基类

**文件**: `content_fetchers/base.py:41-215`

```python
class Fetcher():
    """Base class for all content fetchers"""
    
    # 能力标志
    supports_browser_steps = False      # 是否支持浏览器步骤
    supports_screenshots = False        # 是否支持截图
    supports_xpath_element_data = False # 是否支持 XPath 元素数据
    
    @abstractmethod
    async def run(self, url=None, timeout=None, ...):
        """执行抓取，设置 self.error, self.status_code, self.content"""
        pass
    
    @abstractmethod
    async def quit(self, watch=None):
        """清理资源"""
        return
```

### 2. Requests 抓取器

**文件**: `content_fetchers/requests.py:16-268`

#### 2.1 特点

- 轻量级 HTTP 客户端
- 不执行 JavaScript
- 速度快，资源消耗低
- 不支持浏览器步骤

#### 2.2 执行流程

```python
async def run(self, ...):
    """Async wrapper that runs the synchronous requests code in a thread pool"""
    
    loop = asyncio.get_event_loop()
    
    # 在线程池中运行同步代码
    await loop.run_in_executor(
        None,
        lambda: self._run_sync(...)
    )

def _run_sync(self, url, timeout, ...):
    """Synchronous requests implementation"""
    
    # 1. 检查是否配置了浏览器步骤（不支持）
    if self.browser_steps:
        raise BrowserStepsInUnsupportedFetcher(url=url)
    
    # 2. 配置代理
    proxies = {}
    if self.proxy_override:
        proxies = {'http': self.proxy_override, 'https': self.proxy_override}
    
    # 3. 创建会话并配置重试策略
    session = requests.Session()
    max_retries = int(os.getenv("REQUESTS_RETRY_MAX_COUNT", "6"))
    retry_strategy = Retry(
        total=max_retries,
        connect=max_retries,
        read=max_retries,
        backoff_factor=0.5,
        allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],
    )
    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    
    # 4. 执行请求（手动处理重定向以进行 SSRF 防护）
    r = session.request(method=request_method, ..., allow_redirects=False)
    
    # 5. 手动跟随重定向（验证每一跳）
    for _ in range(10):
        if not r.is_redirect:
            break
        location = r.headers.get('Location', '')
        redirect_url = urljoin(current_url, location)
        # 验证重定向 URL（防止 SSRF）
        if not allow_iana_restricted and is_private_hostname(parsed_redirect.hostname):
            raise Exception("Redirect blocked")
        r = session.request('GET', redirect_url, ..., allow_redirects=False)
    
    # 6. 处理编码检测
    if not is_binary:
        # 优先检测 XML 声明
        if 'xml' in content_type:
            xml_encoding_match = re.search(rb'<\?xml[^>]+encoding=["\']([^"\']+)["\']', r.content[:200])
            if xml_encoding_match:
                r.encoding = xml_encoding_match.group(1).decode('ascii')
        # 然后检测 BOM
        boms = [(b'\xef\xbb\xbf', 'utf-8-sig'), ...]
        bom_encoding = next((enc for bom, enc in boms if r.content.startswith(bom)), None)
        if bom_encoding:
            r.encoding = bom_encoding
        # 然后检测 meta charset
        meta_charset_match = re.search(rb'<meta[^>]+charset\s*=\s*["\']?\s*([^"\'\s;>]+)', r.content[:2000])
        if meta_charset_match:
            r.encoding = meta_charset_match.group(1).decode('ascii', errors='ignore')
        # 最后使用 chardet 猜测
        else:
            encoding = chardet.detect(r.content)['encoding']
            r.encoding = encoding
    
    # 7. 检查状态码
    if r.status_code != 200 and not ignore_status_codes:
        raise Non200ErrorCodeReceived(url=url, status_code=r.status_code, page_html=r.text)
    
    # 8. 设置结果
    self.status_code = r.status_code
    if is_binary:
        self.content = hashlib.md5(r.content).hexdigest()
    else:
        self.content = r.text
    self.raw_content = r.content
```

### 3. Playwright 抓取器

**文件**: `content_fetchers/playwright.py:153-471`

#### 3.1 特点

- 完整浏览器环境
- 支持 JavaScript 执行
- 支持浏览器自动化步骤
- 支持截图功能
- 资源消耗较高

#### 3.2 执行流程

```python
async def run(self, ...):
    """Playwright browser fetcher implementation"""
    
    async with async_playwright() as p:
        browser_type = getattr(p, self.browser_type)
        
        # 1. 连接浏览器
        browser = await browser_type.connect_over_cdp(self.browser_connection_url, timeout=60000)
        
        # 2. 创建上下文
        context = await browser.new_context(
            accept_downloads=False,
            bypass_csp=True,  # 允许在 GitHub 等站点执行 JavaScript
            extra_http_headers=request_headers,
            ignore_https_errors=True,
            proxy=self.proxy,
            service_workers=os.getenv('PLAYWRIGHT_SERVICE_WORKERS', 'allow'),
            user_agent=manage_user_agent(headers=request_headers),
        )
        
        # 3. 创建页面
        self.page = await context.new_page()
        
        # 4. 导航到 URL
        response = await browsersteps_interface.action_goto_url(value=url)
        
        if response is None:
            raise EmptyReply(url=url, status_code=None)
        
        # 5. 获取响应头
        self.headers = await response.all_headers()
        
        # 6. 执行自定义 JS 代码
        if self.webdriver_js_execute_code:
            await browsersteps_interface.action_execute_js(value=self.webdriver_js_execute_code)
        
        # 7. 等待内容稳定
        extra_wait = int(os.getenv("WEBDRIVER_DELAY_BEFORE_CONTENT_READY", 5)) + self.render_extract_delay
        await self.page.wait_for_timeout(extra_wait * 1000)
        
        # 8. 检查状态码
        self.status_code = response.status
        if self.status_code != 200 and not ignore_status_codes:
            screenshot = await capture_full_page_async(self.page, ...)
            raise Non200ErrorCodeReceived(url=url, status_code=self.status_code, screenshot=screenshot)
        
        # 9. 执行浏览器步骤
        if self.browser_steps:
            await self.iterate_browser_steps(start_url=url)
            await self.page.wait_for_timeout(extra_wait * 1000)
        
        # 10. 提取数据
        # 10.1 提取 XPath 元素数据（用于可视化选择器）
        self.xpath_data = await self.page.evaluate(XPATH_ELEMENT_JS, {
            "visualselector_xpath_selectors": visualselector_xpath_selectors,
            "max_height": MAX_TOTAL_HEIGHT
        })
        
        # 10.2 提取库存数据（用于补货监控）
        self.instock_data = await self.page.evaluate(INSTOCK_DATA_JS)
        
        # 10.3 提取页面内容
        self.content = await self.page.content()
        
        # 10.4 截图
        self.screenshot = await capture_full_page_async(page=self.page, ...)
        
        # 11. 清理资源（finally 块确保执行）
        try:
            await asyncio.wait_for(self.page.close(), timeout=5.0)
        except Exception:
            pass
        finally:
            self.page = None
        
        try:
            await asyncio.wait_for(context.close(), timeout=5.0)
        except Exception:
            pass
        
        try:
            await asyncio.wait_for(browser.close(), timeout=5.0)
        except Exception:
            pass
```

#### 3.3 全页截图实现

**文件**: `content_fetchers/playwright.py:16-151`

```python
async def capture_full_page_async(page, screenshot_format='JPEG', watch_uuid=None, lock_viewport_elements=False):
    """
    Capture full page screenshot with intelligent chunking.
    
    ARCHITECTURE:
    - For large pages, scroll and capture in chunks
    - Use subprocess for stitching to prevent memory leaks
    """
    
    # 1. 获取页面尺寸
    page_height = await page.evaluate("document.documentElement.scrollHeight")
    page_width = await page.evaluate("document.documentElement.scrollWidth")
    
    # 2. 锁定视口元素（防止布局偏移）
    if lock_viewport_elements and page_height > page.viewport_size['height']:
        lock_elements_js = read_file('lock-elements-sizing.js')
        await page.evaluate(lock_elements_js)
    
    # 3. 分块截图
    step_size = SCREENSHOT_SIZE_STITCH_THRESHOLD  # 默认 10000px
    screenshot_chunks = []
    y = 0
    
    while y < min(page_height, SCREENSHOT_MAX_TOTAL_HEIGHT):
        if y > 0:
            await page.evaluate(f"window.scrollTo(0, {y})")
        
        await page.request_gc()
        
        screenshot_kwargs = {
            'type': screenshot_format.lower(),
            'full_page': False
        }
        if screenshot_format.lower() == 'jpeg':
            screenshot_kwargs['quality'] = int(os.getenv("SCREENSHOT_QUALITY", 72))
        
        screenshot_chunks.append(await page.screenshot(**screenshot_kwargs))
        y += step_size
    
    # 4. 恢复原始视口
    await page.set_viewport_size({'width': original_viewport['width'], 'height': original_viewport['height']})
    
    # 5. 拼接截图（使用子进程防止内存泄漏）
    if len(screenshot_chunks) > 1:
        # 使用 spawn 子进程
        ctx = multiprocessing.get_context('spawn')
        parent_conn, child_conn = ctx.Pipe()
        p = ctx.Process(target=stitch_images_worker_raw_bytes, args=(child_conn, page_height, SCREENSHOT_MAX_TOTAL_HEIGHT))
        p.start()
        
        # 通过原始字节发送（不使用 pickle）
        parent_conn.send_bytes(struct.pack('I', len(screenshot_chunks)))
        for chunk in screenshot_chunks:
            parent_conn.send_bytes(chunk)
        
        screenshot = parent_conn.recv_bytes()
        p.join()
        
        return screenshot
    else:
        return screenshot_chunks[0]
```

### 4. 抓取器调用时机

**文件**: `worker.py:156-167`

```python
# Worker 中调用处理器
processor = watch.get('processor', 'text_json_diff')
processor_module = get_processor_module(processor)
update_handler = processor_module.perform_site_check(datastore=datastore, watch_uuid=uuid)

# 允许插件修改处理器
update_handler = apply_update_handler_alter(update_handler, watch, datastore)

# 调用抓取器（所有抓取器现在都是异步的）
await update_handler.call_browser()

# 在执行器中运行变更检测（CPU 密集型操作）
loop = asyncio.get_event_loop()
changed_detected, update_obj, contents = await loop.run_in_executor(
    executor,
    lambda: update_handler.run_changedetection(watch=watch)
)
```

---

## 优先级策略详解

### 1. 优先级值含义

| 优先级值 | 含义 | 触发场景 |
|---------|------|---------|
| 1 | 最高优先级（立即执行） | 手动触发、API 调用、实时事件 |
| 5 | 高优先级 | 克隆操作 |
| 时间戳 (>100) | 常规优先级 | 调度器自动触发 |
| 1000+ | 延迟重试 | UUID 已被其他 Worker 处理时的重新入队 |

### 2. 优先级计算

#### 2.1 调度器触发

```python
# flask_app.py:1248
priority = int(time.time())  # 使用当前时间戳
```

使用时间戳的优势：
- 较早入队的任务具有较低的优先级值（更高优先级）
- 自然实现 FIFO（先入先出）
- 可以被手动触发的任务（priority=1）抢占

#### 2.2 手动触发

```python
# 多处使用 priority=1
worker_pool.queue_item_async_safe(update_q,
    PrioritizedItem(priority=1, item={'uuid': uuid})
)
```

#### 2.3 延迟重试

```python
# worker.py:74-76
deferred_priority = max(1000, queued_item_data.priority * 10)
deferred_item = PrioritizedItem(priority=deferred_priority, item=queued_item_data.item)
```

---

## 异常处理与恢复机制

### 1. Worker 级别异常处理

**文件**: `worker.py:157-387`

Worker 捕获的异常类型：

```python
try:
    await update_handler.call_browser()
except PermissionError as e:
    # 文件权限错误
    logger.critical(f"File permission error: {e}")
except ProcessorException as e:
    # 处理器异常（可能包含截图）
    if e.screenshot:
        watch.save_screenshot(screenshot=e.screenshot)
    datastore.update_watch(uuid=uuid, update_obj={'last_error': e.message})
except content_fetchers_exceptions.ReplyWithContentButNoText as e:
    # 有内容但无文本（可能是过滤问题）
    datastore.update_watch(uuid=uuid, update_obj={
        'last_error': f"Got HTML content but no text found (With {e.status_code} reply code)"
    })
except content_fetchers_exceptions.Non200ErrorCodeReceived as e:
    # 非 200 状态码
    err_text = f"Error - Request returned a HTTP error code {e.status_code}"
    if e.screenshot:
        watch.save_screenshot(screenshot=e.screenshot, as_error=True)
    datastore.update_watch(uuid=uuid, update_obj={'last_error': err_text})
except FilterNotFoundInResponse as e:
    # 过滤器未找到
    datastore.update_watch(uuid=uuid, update_obj={
        'last_error': "Warning, no filters were found..."
    })
    # 可能发送过滤器失败通知
except content_fetchers_exceptions.BrowserStepsStepException as e:
    # 浏览器步骤执行失败
    error_step = e.step_n + 1
    datastore.update_watch(uuid=uuid, update_obj={
        'last_error': f"Browser step at position {error_step} could not run...",
        'browser_steps_last_error_step': error_step
    })
except Exception as e:
    # 通用异常
    logger.exception(f"Worker {worker_id} full exception details:")
    datastore.update_watch(uuid=uuid, update_obj={'last_error': "Exception: " + str(e)})
```

### 2. 资源清理

**文件**: `worker.py:605-674`

```python
finally:
    # 1. 关闭抓取器
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
        
        # 强制垃圾回收
        import gc
        gc.collect()
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

**文件**: `worker.py:43-48, 684-695`

```python
# 配置
max_jobs = int(os.getenv("WORKER_MAX_JOBS", "10"))
max_runtime_seconds = int(os.getenv("WORKER_MAX_RUNTIME", "3600"))  # 1 小时

# 重启条件检查
should_restart_jobs = jobs_processed >= max_jobs
should_restart_time = runtime >= max_runtime_seconds

if should_restart_jobs or should_restart_time:
    reason = f"{jobs_processed} jobs" if should_restart_jobs else f"{runtime:.0f}s runtime"
    logger.info(f"Worker {worker_id} restarting after {reason}")
    return "restart"
```

### 4. 工作池健康检查

**文件**: `worker_pool.py:498-553`

```python
def check_worker_health(expected_count, update_q=None, notification_q=None, app=None, datastore=None):
    """
    Check if the expected number of async workers are running and restart any missing ones.
    """
    
    alive_count = sum(1 for w in worker_threads if w.thread and w.thread.is_alive())
    
    if alive_count == expected_count:
        return {
            'status': 'healthy',
            'message': f'All {expected_count} async workers running'
        }
    
    # 找出死亡的 Worker
    dead_workers = []
    for i, worker in enumerate(worker_threads[:]):
        if not worker.thread or not worker.thread.is_alive():
            dead_workers.append(i)
            worker_threads.pop(i)  # 从列表移除
    
    # 重启缺失的 Worker
    missing_workers = expected_count - alive_count
    if missing_workers > 0 and all([update_q, notification_q, app, datastore]):
        logger.info(f"Restarting {missing_workers} crashed async workers")
        for i in range(missing_workers):
            add_worker(update_q, notification_q, app, datastore)
```

---

## 关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 调度器主循环 | `flask_app.py` | ~1150-1270 |
| 优先级队列实现 | `queue_handlers.py` | 15-411 |
| 优先级项数据结构 | `queuedWatchMetaData.py` | 7-9 |
| 工作池管理 | `worker_pool.py` | 完整文件 |
| Worker 主逻辑 | `worker.py` | 23-708 |
| 抓取器选择 | `processors/base.py` | 117-260 |
| Requests 抓取器 | `content_fetchers/requests.py` | 16-268 |
| Playwright 抓取器 | `content_fetchers/playwright.py` | 153-471 |
| 抓取器注册 | `content_fetchers/__init__.py` | 39-110 |
| Watch 抓取后端 | `model/Watch.py` | 356-390 |

---

## 配置项参考

### 1. 队列与 Worker 配置

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `FETCH_WORKERS` | 10 | 工作线程数量 |
| `WORKER_MAX_JOBS` | 10 | 每个 Worker 处理的最大任务数后重启 |
| `WORKER_MAX_RUNTIME` | 3600 | 每个 Worker 运行的最大秒数后重启 |
| `MINIMUM_SECONDS_RECHECK_TIME` | 3 | 最小重检间隔秒数 |

### 2. 请求配置

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `REQUESTS_RETRY_MAX_COUNT` | 6 | Requests 抓取器最大重试次数 |
| `DEFAULT_SETTINGS_REQUESTS_TIMEOUT` | 45 | 默认请求超时秒数 |
| `DEFAULT_SETTINGS_REQUESTS_WORKERS` | 5 | 默认工作线程数 |
| `DEFAULT_FETCH_BACKEND` | html_requests | 默认抓取后端 |

### 3. 浏览器配置

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `PLAYWRIGHT_DRIVER_URL` | ws://playwright-chrome:3000 | Playwright 连接 URL |
| `PLAYWRIGHT_BROWSER_TYPE` | chromium | 浏览器类型 |
| `FAST_PUPPETEER_CHROME_FETCHER` | False | 是否使用 Puppeteer |
| `WEBDRIVER_DELAY_BEFORE_CONTENT_READY` | 5 | 内容就绪前等待秒数 |
| `PLAYWRIGHT_SERVICE_WORKERS` | allow | 是否允许 Service Workers |

### 4. 截图配置

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `SCREENSHOT_MAX_HEIGHT` | 20000 | 最大截图高度（像素） |
| `SCREENSHOT_CHUNK_HEIGHT` | 10000 | 分块截图高度阈值 |
| `SCREENSHOT_QUALITY` | 72 | JPEG 截图质量 |

---

## 总结

Changedetection.io 的调度与抓取系统采用了成熟的生产者-消费者模型，具有以下特点：

1. **高可扩展性**：基于异步事件循环的多 Worker 架构，支持动态调整 Worker 数量

2. **优先级调度**：使用优先级队列实现任务分级，手动触发优先于自动调度

3. **灵活的抓取器选择**：根据任务配置自动选择 Requests 或浏览器抓取器

4. **完善的异常处理**：多层级异常捕获和自动恢复机制

5. **资源管理**：内存清理、Worker 重启、子进程隔离等策略防止资源泄漏

6. **线程安全**：UUID 声明机制、锁保护、原子操作确保并发安全

这个架构能够高效处理大量监控任务，同时保持系统的稳定性和可靠性。
