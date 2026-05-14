# Worker 公平性与退避策略分析报告

## 1. 系统架构概览

### 1.1 核心组件

changedetection.io 采用多线程异步 Worker 架构来处理网页监控任务，主要组件包括：

- **Ticker 线程** (`flask_app.py:ticker_thread_check_time_launch_checks`): 定时扫描所有监控项，根据调度规则将满足条件的任务添加到队列
- **优先级队列** (`queue_handlers.py:RecheckPriorityQueue`): 基于堆的优先级队列，支持同步和异步接口
- **Worker 线程池** (`worker_pool.py`): 多个独立的 Worker 线程，每个线程运行独立的事件循环
- **UUID 跟踪机制** (`worker_pool.py:currently_processing_uuids`): 防止同一监控项被重复处理

### 1.2 架构特点

```
Ticker Thread (调度器)
      ↓
RecheckPriorityQueue (优先级队列)
      ↓
Worker Thread 1  Worker Thread 2  Worker Thread N
(独立事件循环)    (独立事件循环)    (独立事件循环)
```

## 2. 优先级调度机制

### 2.1 优先级分类

系统采用多层优先级机制，通过 `PrioritizedItem` 数据结构实现：

| 优先级值 | 类型 | 来源 | 说明 |
|---------|------|------|------|
| 1 | 立即执行 | 手动触发/API调用 | 用户操作的最高优先级 |
| 5 | 克隆操作 | 监控项克隆 | 次高优先级 |
| Unix时间戳 > 100 | 定时调度 | Ticker自动调度 | 按时间先后顺序执行 |

### 2.2 优先级队列实现

**位置**: `queue_handlers.py:RecheckPriorityQueue`

核心实现特点：
- 使用 `heapq` 实现最小堆，保证 O(log n) 的插入和删除复杂度
- 混合同步/异步设计，支持多事件循环架构
- 线程安全，使用 `threading.RLock` 保护关键操作

```python
# 关键实现代码片段 (queue_handlers.py:65-85)
def put(self, item, block=True, timeout=None):
    with self._lock:
        heapq.heappush(self._priority_items, item)
        self._notification_queue.put(True, block=True, timeout=5.0)
    # 发送信号用于UI更新
    self._emit_put_signals(item)
```

## 3. 不同来源的公平性保障

### 3.1 基于优先级的公平性

系统通过优先级机制保障不同来源任务的公平性：

#### 3.1.1 任务来源与优先级映射

**立即执行 (Priority=1)** 的触发场景：
- **手动刷新**: 用户通过UI点击"重新检查"按钮 (`blueprint/ui/__init__.py:66`)
- **API触发**: `/api/v1/watch/<uuid>?recheck=true` (`api/Watch.py:81`)
- **新建监控**: 添加新监控项后立即检查 (`blueprint/ui/views.py:41`)
- **Socket.IO实时事件**: 实时触发的检查请求 (`realtime/events.py:44`)
- **价格数据跟随**: 价格相关的主动检查 (`blueprint/price_data_follower/__init__.py:24`)
- **编辑后重检**: 监控项配置修改后 (`blueprint/ui/edit.py:277`)
- **标签级操作**: 对标签下所有监控项的批量操作 (`api/Tags.py:42,50`)

**克隆操作 (Priority=5)** 的触发场景：
- **监控项克隆**: 用户复制现有监控项时 (`blueprint/ui/__init__.py:257`)

**定时调度 (Priority=Unix时间戳)** 的触发场景：
- **Ticker 自动调度**: 基于 `time_between_check` 配置的周期性检查 (`flask_app.py:1249`)

#### 3.1.2 优先级队列的公平性特性

1. **FIFO 特性**: 相同优先级的任务遵循先进先出原则
2. **优先级抢占**: 高优先级任务会被优先处理
3. **时间顺序**: 定时调度任务按最早到期的时间戳排序

### 3.2 基于扫描顺序的公平性

**位置**: `flask_app.py:1157`

Ticker 线程在每次调度循环中，按照 `last_checked` 时间排序扫描所有监控项：

```python
# 按 last_checked 升序排列，确保最久未检查的先被考虑
for k in sorted(datastore.data['watching'].items(), 
                key=lambda item: item[1].get('last_checked', 0)):
    watch_uuid_list.append(k[0])
```

这种排序策略确保：
- 最久未被检查的监控项优先获得调度机会
- 防止某些监控项被持续"饿死"
- 在高负载下仍保持基本的时间公平性

### 3.3 代理级别的公平性

**位置**: `flask_app.py:1229-1246`

系统支持为每个代理配置 `reuse_time_minimum` 限制，防止对同一代理的过度使用：

```python
watch_proxy = datastore.get_preferred_proxy_for_watch(uuid=uuid)
if watch_proxy:
    proxy_list_reuse_time_minimum = int(
        datastore.proxy_list.get(watch_proxy, {}).get('reuse_time_minimum', 0)
    )
    if proxy_list_reuse_time_minimum:
        proxy_last_used_time = proxy_last_called_time.get(watch_proxy, 0)
        time_since_proxy_used = int(time.time() - proxy_last_used_time)
        if time_since_proxy_used < proxy_list_reuse_time_minimum:
            # 跳过此监控项，等待下次循环
            continue
        else:
            proxy_last_called_time[watch_proxy] = int(time.time())
```

**设计意图**:
- 防止对同一目标服务器的请求频率过高
- 避免被目标网站封禁
- 在多代理场景下，分散请求压力

## 4. 繁忙队列下的退避策略

### 4.1 队列大小限制

**位置**: `flask_app.py:60, 1171-1176`

系统定义了最大队列大小限制：

```python
MAX_QUEUE_SIZE = 5000

# 在调度循环中检查队列大小
if watch_index % 100 == 0:  # 每100个监控项检查一次
    current_queue_size = update_q.qsize()
    if current_queue_size >= MAX_QUEUE_SIZE:
        logger.debug(f"Queue size limit reached ({current_queue_size}/{MAX_QUEUE_SIZE}), stopping scheduler this iteration.")
        break
```

**退避策略**:
- 当队列达到 5000 项时，停止本轮调度
- 等待下一个调度周期 (1秒后) 再尝试
- 防止队列无限增长导致内存溢出

### 4.2 重复处理保护机制

**位置**: `worker.py:70-77`

当 Worker 从队列获取任务后，会立即尝试"认领"该 UUID：

```python
# 从队列获取任务后立即尝试认领
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # 已被其他 Worker 认领，延迟后重新入队
    logger.trace(f"Worker {worker_id} detected UUID {uuid} already processing during claim - deferring")
    await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)  # 生产环境 10 秒，测试环境 0.3 秒
    deferred_priority = max(1000, queued_item_data.priority * 10)  # 优先级降低
    deferred_item = PrioritizedItem(priority=deferred_priority, item=queued_item_data.item)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

**退避策略特点**:
- **指数退避**: 延迟时间随优先级降低而增加
- **优先级降低**: 重新入队时优先级变为 `max(1000, original_priority * 10)`
- **防止竞态**: 使用线程安全的 `claim_uuid_for_processing` 机制

### 4.3 UUID 跟踪机制

**位置**: `worker_pool.py:18-240`

```python
# 全局状态追踪
currently_processing_uuids = {}  # {uuid: worker_id}
_uuid_processing_lock = threading.Lock()

# 原子认领操作
def claim_uuid_for_processing(uuid, worker_id):
    with _uuid_processing_lock:
        if uuid in currently_processing_uuids:
            return False  # 已被认领
        currently_processing_uuids[uuid] = worker_id
        return True

# 释放操作
def release_uuid_from_processing(uuid, worker_id):
    with _uuid_processing_lock:
        if currently_processing_uuids.get(uuid) == worker_id:
            currently_processing_uuids.pop(uuid, None)
```

**公平性保障**:
- 防止同一监控项被多个 Worker 同时处理
- 确保每个监控项在同一时间只有一个执行实例
- 避免资源浪费和重复请求

### 4.4 Worker 重启机制

**位置**: `worker.py:44-45, 688-695`

系统实现了 Worker 的自动重启策略，防止内存泄漏和资源耗尽：

```python
# 环境变量配置
max_jobs = int(os.getenv("WORKER_MAX_JOBS", "10"))           # 每个 Worker 最多处理 10 个任务
max_runtime_seconds = int(os.getenv("WORKER_MAX_RUNTIME", "3600"))  # 最长运行 1 小时

# 检查重启条件
should_restart_jobs = jobs_processed >= max_jobs
should_restart_time = runtime >= max_runtime_seconds

if should_restart_jobs or should_restart_time:
    logger.info(f"Worker {worker_id} restarting after {reason}")
    return "restart"
```

**退避策略**:
- 任务数限制: 防止内存泄漏累积
- 运行时间限制: 防止长时间运行的任务阻塞
- 优雅重启: 在任务间隙重启，不中断正在处理的任务

### 4.5 Worker 健康检查

**位置**: `worker_pool.py:498-553`

系统定期检查 Worker 健康状态并自动恢复：

```python
def check_worker_health(expected_count, update_q=None, notification_q=None, app=None, datastore=None):
    # 检查哪些 Worker 实际在运行
    alive_count = sum(1 for w in worker_threads if w.thread and w.thread.is_alive())
    
    if alive_count == expected_count:
        return {'status': 'healthy', ...}
    
    # 找出死亡的 Worker
    dead_workers = []
    for i, worker in enumerate(worker_threads[:]):
        if not worker.thread or not worker.thread.is_alive():
            dead_workers.append(i)
    
    # 移除并重启死亡的 Worker
    for i in reversed(dead_workers):
        worker_threads.pop(i)
    
    missing_workers = expected_count - alive_count
    if missing_workers > 0:
        for i in range(missing_workers):
            add_worker(update_q, notification_q, app, datastore)
    
    return {'status': 'repaired', ...}
```

**位置**: `flask_app.py:1123-1138`

Ticker 线程每 60 秒执行一次健康检查：

```python
if now - last_health_check > 60:
    expected_workers = int(os.getenv("FETCH_WORKERS", datastore.data['settings']['requests']['workers']))
    health_result = worker_pool.check_worker_health(
        expected_count=expected_workers,
        update_q=update_q,
        notification_q=notification_q,
        app=app,
        datastore=datastore
    )
    if health_result['status'] != 'healthy':
        logger.warning(f"Worker health check: {health_result['message']}")
    last_health_check = now
```

## 5. 阻塞避免机制

### 5.1 混合同步/异步架构

**位置**: `queue_handlers.py:15-39`

系统采用混合设计避免阻塞：

```
设计目标:
- 同步调用方 (Ticker线程、Flask路由) 使用 threading.Queue
- 异步 Worker 使用 asyncio.Event 等待
- 避免 run_in_executor 导致的线程耗尽问题

关键特点:
- 每个 Worker 有独立的事件循环
- 等待时使用纯协程，不占用线程
- 可扩展到 100-200+ Worker
```

### 5.2 超时机制

**位置**: `queue_handlers.py:161-202`

所有队列操作都有超时保护：

```python
async def async_get(self, executor=None, timeout=1.0):
    try:
        item = await loop.run_in_executor(
            executor,
            lambda: self.get(block=True, timeout=timeout)
        )
        return item
    except queue.Empty:
        # 超时是正常行为，继续等待
        raise
```

**防止阻塞的策略**:
- 队列获取设置 1 秒超时
- Worker 定期检查退出标志
- 避免无限阻塞等待

## 6. 调度流程详解

### 6.1 Ticker 调度流程

```
每 1 秒执行一次调度循环:
│
├─→ 检查 Worker 健康状态 (每 60 秒)
│
├─→ 检查是否全局暂停 (all_paused)
│
├─→ 获取正在运行和已排队的 UUIDs
│
├─→ 按 last_checked 排序所有监控项
│
└─→ 遍历每个监控项:
    │
    ├─→ 每 100 项检查队列是否已满
    │   └─→ 满则停止本轮调度
    │
    ├─→ 检查监控项是否暂停
    │   └─→ 暂停则跳过
    │
    ├─→ 检查时间调度窗口
    │   └─→ 不在窗口内则跳过
    │
    ├─→ 计算时间阈值 (考虑 jitter)
    │
    ├─→ 检查是否满足重检条件
    │   └─→ 不满足则跳过
    │
    ├─→ 检查代理使用频率限制
    │   └─→ 超限则跳过
    │
    └─→ 加入队列 (优先级 = 当前时间戳)
```

### 6.2 Worker 处理流程

```
Worker 循环:
│
├─→ 从队列获取任务 (带 1 秒超时)
│
├─→ 尝试认领 UUID
│   ├─→ 成功 → 继续处理
│   └─→ 失败 → 延迟 10 秒后重新入队 (优先级降低)
│
├─→ 执行页面抓取和变更检测
│
├─→ 处理变更检测结果
│   ├─→ 有变更 → 发送通知
│   └─→ 无变更 → 记录检查时间
│
├─→ 释放 UUID
│
└─→ 检查是否需要重启 Worker
    ├─→ 处理任务数 > 10 → 重启
    └─→ 运行时间 > 1 小时 → 重启
```

## 7. 配置参数汇总

### 7.1 环境变量配置

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| `FETCH_WORKERS` | 5 | Worker 线程数量 |
| `WORKER_MAX_JOBS` | 10 | 每个 Worker 处理多少任务后重启 |
| `WORKER_MAX_RUNTIME` | 3600 | Worker 最长运行时间 (秒) |
| `NOTIFICATION_WORKERS` | 1 | 通知 Worker 数量 |
| `MINIMUM_SECONDS_RECHECK_TIME` | 3 | 最小重检间隔 (秒) |

### 7.2 硬编码限制

| 参数 | 值 | 位置 | 说明 |
|-----|---|------|------|
| `MAX_QUEUE_SIZE` | 5000 | `flask_app.py:60` | 最大队列大小 |
| `DEFER_SLEEP_TIME_ALREADY_QUEUED` | 10 秒 | `worker.py:21` | 重复处理时的延迟 |
| `WAIT_TIME_BETWEEN_LOOP` | 1 秒 | `flask_app.py:1116` | Ticker 循环间隔 |

## 8. 公平性分析总结

### 8.1 已实现的公平性机制

1. **优先级分层**: 用户操作 > 克隆操作 > 定时调度
2. **时间排序**: 定时任务按到期时间先后执行
3. **代理限流**: 防止对同一代理的过度使用
4. **重复保护**: 同一 UUID 不会被重复处理
5. **队列限制**: 防止队列无限增长
6. **Worker 隔离**: 每个 Worker 独立事件循环

### 8.2 潜在的公平性问题

1. **优先级反转**: 高优先级任务可能被低优先级任务"阻塞" (正在执行的任务不可抢占)
2. **域名级公平性**: 未实现按域名的请求频率限制
3. **用户级公平性**: 多用户场景下未实现用户间的公平调度
4. **长期饥饿**: 低优先级任务在持续高负载下可能长时间得不到执行

### 8.3 改进建议

1. **引入域名限流**: 类似代理限流，为每个目标域名设置访问频率限制
2. **实现加权轮询**: 在相同优先级内，按监控项或用户实现轮询调度
3. **老化机制**: 低优先级任务随时间推移逐渐提升优先级
4. **可配置的队列大小**: 将 `MAX_QUEUE_SIZE` 改为可配置参数
5. **监控指标**: 添加队列延迟、等待时间等监控指标

## 9. 参考文件

- `changedetectionio/queue_handlers.py`: 优先级队列实现
- `changedetectionio/worker_pool.py`: Worker 池管理
- `changedetectionio/worker.py`: Worker 主循环
- `changedetectionio/flask_app.py`: Ticker 调度器
- `changedetectionio/queuedWatchMetaData.py`: 优先级项定义
