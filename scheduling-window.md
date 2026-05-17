# 时间处理与调度窗口在抓取任务编排中的角色分析

## 1. 概述

在 changedetection.io 系统中，时间处理与调度窗口是抓取任务编排的核心机制，负责控制何时执行网页监控任务。该机制通过多层时间检查、时区感知、队列优先级和并发控制，实现了灵活且高效的任务调度。

## 2. 核心架构

### 2.1 模块组成

| 模块 | 职责 | 核心文件 |
|------|------|----------|
| 时间处理器 | 时间窗口判断、时区转换 | `changedetectionio/time_handler.py` |
| 调度器主循环 | 扫描watch、入队决策 | `changedetectionio/flask_app.py` (ticker_thread) |
| 优先级队列 | 任务排序、公平性保证 | `changedetectionio/queue_handlers.py` |
| 工作池管理 | 并发控制、worker生命周期 | `changedetectionio/worker_pool.py` |
| 任务执行器 | 实际抓取任务处理 | `changedetectionio/worker.py` |

### 2.2 数据流

```
用户配置时间窗口 → 存储到watch/全局配置
        ↓
Ticker线程 (每秒循环)
        ↓
遍历所有watch → 检查时间窗口 → 检查检查间隔 → 检查队列状态
        ↓
符合条件 → 加入优先级队列 (PriorityQueue)
        ↓
Worker池竞争获取任务 → 执行抓取 → 结果处理
```

## 3. 时间窗口配置与解释

### 3.1 配置结构

用户可在两个层级配置时间调度：

1. **全局系统级**：在设置中配置，作为所有watch的默认值
2. **Watch级**：每个监控项可单独配置，覆盖全局设置

配置结构示例：
```python
time_schedule_limit = {
    'enabled': True,
    'timezone': 'Europe/Berlin',
    'monday': {
        'enabled': True,
        'start_time': '09:00',
        'duration': {'hours': 8, 'minutes': 0}
    },
    'tuesday': { ... },
    # ... 其他星期
}
```

### 3.2 核心检查函数

#### `is_within_schedule(time_schedule_limit, default_tz="UTC")`

**位置**：`time_handler.py:83-114`

**功能**：判断当前时间是否在允许的调度窗口内。

**执行流程**：
1. 检查调度是否启用 (`enabled` 字段)
2. 确定时区：优先使用配置中的 `timezone`，否则使用默认值
3. 获取目标时区的当前星期几
4. 检查当天是否启用调度
5. 计算持续时间（小时×60 + 分钟）
6. 调用 `am_i_inside_time()` 进行精确时间判断

#### `am_i_inside_time(day_of_week, time_str, timezone_str, duration=15)`

**位置**：`time_handler.py:17-80`

**功能**：精确判断当前时间是否落在指定的时间窗口内。

**关键逻辑**：
```python
# 处理跨天情况（如 23:30 开始，持续2小时）
# 1. 检查前一天的重叠
if target_weekday == (current_weekday - 1) % 7:
    start_datetime_tz = start_datetime_tz.shift(days=-1)
    end_datetime_tz = start_datetime_tz.shift(minutes=duration)
    if start_datetime_tz <= now_tz <= end_datetime_tz:
        return True

# 2. 检查当天范围
if target_weekday == current_weekday:
    end_datetime_tz = start_datetime_tz.shift(minutes=duration)
    if start_datetime_tz <= now_tz <= end_datetime_tz:
        return True

# 3. 检查次日重叠
if target_weekday == (current_weekday + 1) % 7:
    end_datetime_tz = start_datetime_tz.shift(minutes=duration)
    if now_tz < start_datetime_tz and now_tz.shift(days=1) <= end_datetime_tz:
        return True
```

### 3.3 调度器主循环中的应用

**位置**：`flask_app.py:1107-1270` (`ticker_thread_check_time_launch_checks`)

在每秒执行的调度循环中，时间窗口检查是第一道关卡：

```python
# 确定使用全局还是watch级配置
if watch.get('time_between_check_use_default'):
    time_schedule_limit = datastore.data['settings']['requests'].get('time_schedule_limit', {})
else:
    time_schedule_limit = watch.get('time_schedule_limit')

# 执行时间窗口检查
if time_schedule_limit and time_schedule_limit.get('enabled'):
    result = is_within_schedule(time_schedule_limit=time_schedule_limit,
                                default_tz=tz_name)
    if not result:
        # 不在时间窗口内，跳过此watch
        continue
```

## 4. 跨时区与夏令时处理

### 4.1 时区处理机制

系统使用 **Arrow** 库进行时间处理，该库原生支持IANA时区数据库，能够正确处理时区转换和夏令时。

**时区确定优先级**：
1. Watch配置中的 `time_schedule_limit.timezone`
2. 全局设置 `scheduler_timezone_default`
3. 系统环境变量 `TZ`
4. 默认值 `UTC`

**位置**：`flask_app.py:1199`
```python
tz_name = datastore.data['settings']['application'].get(
    'scheduler_timezone_default', 
    os.getenv('TZ', 'UTC').strip()
)
```

### 4.2 夏令时自动处理

由于使用 Arrow 库和 IANA 时区（如 `America/New_York` 而非 `EST`），系统能够自动处理夏令时切换：

- **时间转换**：`arrow.now(timezone_str)` 自动考虑夏令时
- **时间计算**：`shift()` 方法在加减时间时正确处理夏令时跳跃
- **跨天判断**：基于实际时间点而非固定偏移量进行比较

**示例**：
```python
# 在美国东部时区，从3月到11月是EDT（UTC-4），其他时间是EST（UTC-5）
now_tz = arrow.now('America/New_York')
# Arrow会自动选择正确的偏移量
```

### 4.3 午夜跨天处理

时间窗口支持跨午夜的情况，例如：
- 开始时间：23:30（周一）
- 持续时间：120分钟
- 实际覆盖：周一23:30 - 周二01:30

`am_i_inside_time()` 函数通过三次检查确保正确处理：
1. 检查是否在前一天窗口的溢出部分
2. 检查是否在当天窗口内
3. 检查是否在当天窗口溢出到次日的部分

## 5. 窗口外任务推迟机制

### 5.1 推迟策略

当watch不在时间窗口内时，采用**静默跳过**策略：

1. **不立即入队**：不在时间窗口内的watch不会被加入队列
2. **等待下一轮**：ticker线程每秒循环一次，下次循环时重新检查
3. **无惩罚机制**：跳过不会影响下次调度的优先级

**位置**：`flask_app.py:1207-1209`
```python
if not result:
    logger.trace(f"{uuid} Time scheduler - not within schedule skipping.")
    continue
```

### 5.2 多重准入检查

在进入队列前，watch需要通过多层检查：

| 检查层级 | 描述 | 失败处理 |
|---------|------|----------|
| 全局暂停 | `all_paused` 标志 | 跳过整个循环 |
| Watch暂停 | `watch['paused']` | 跳过此watch |
| 时间窗口 | `is_within_schedule()` | 跳过此watch |
| 检查间隔 | `time_between_check` + jitter | 跳过此watch |
| 代理限制 | 代理重用间隔 | 跳过此watch |
| 运行状态 | 是否已在运行/队列中 | 跳过此watch |
| 队列上限 | 队列大小 ≤ 5000 | 停止本轮调度 |

### 5.3 检查间隔与抖动

除了时间窗口，系统还使用检查间隔控制频率：

```python
# 计算实际阈值
threshold = recheck_time_system_seconds if watch.get('time_between_check_use_default') else watch.threshold_seconds()

# 应用抖动（±jitter_seconds）
jitter = datastore.data['settings']['requests'].get('jitter_seconds', 0)
if jitter > 0:
    if watch.jitter_seconds == 0:
        watch.jitter_seconds = random.uniform(-abs(jitter), jitter)

# 最终判断
if seconds_since_last_recheck >= (threshold + watch.jitter_seconds):
    # 可以入队
```

抖动机制避免了大量watch在同一时间集中请求，分散了负载。

## 6. 队列公平性与优先级

### 6.1 优先级队列实现

系统使用 `RecheckPriorityQueue`（基于堆的优先级队列），优先级规则：

```python
# 使用当前Unix时间戳作为优先级
# 这样：
# 1. 最早应该执行的任务（时间戳最小）最先被处理
# 2. 可以通过插入优先级1来插队（立即执行）
priority = int(time.time())
queuedWatchMetaData.PrioritizedItem(priority=priority, item={'uuid': uuid})
```

### 6.2 优先级分类

| 优先级值 | 类型 | 场景 |
|---------|------|------|
| 1 | 立即执行 | 用户手动触发、API调用 |
| 5 | 克隆任务 | 批量操作 |
| >100 | 定时调度 | 正常ticker入队（Unix时间戳） |

### 6.3 公平性保证

1. **排序策略**：按 `last_checked` 升序遍历watch，最久未检查的优先被考虑
   ```python
   # flask_app.py:1157
   for k in sorted(datastore.data['watching'].items(), 
                   key=lambda item: item[1].get('last_checked',0)):
   ```

2. **防重复入队**：检查是否已在运行或队列中
   ```python
   if not uuid in running_uuids and uuid not in queued_uuids:
       # 入队
   ```

3. **队列大小限制**：最大5000项，防止内存溢出
   ```python
   MAX_QUEUE_SIZE = 5000
   if current_queue_size >= MAX_QUEUE_SIZE:
       break
   ```

### 6.4 并发冲突处理

当多个worker尝试处理同一任务时：

```python
# worker.py:93-100
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # 已被其他worker认领，推迟并重新入队
    await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)
    deferred_priority = max(1000, queued_item_data.priority * 10)
    deferred_item = PrioritizedItem(priority=deferred_priority, 
                                    item=queued_item_data.item)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

**原子性保证**：使用 `threading.Lock` 保护UUID认领操作，确保同一时间只有一个worker处理某个watch。

## 7. 抓取并发上限控制

### 7.1 Worker池配置

并发抓取能力由Worker数量控制：

```python
# 配置来源优先级：
# 1. 环境变量 FETCH_WORKERS
# 2. 全局设置 settings.requests.workers
# 3. 默认值 5
n_workers = int(os.getenv("FETCH_WORKERS", 
              datastore.data['settings']['requests']['workers']))
worker_pool.start_workers(n_workers, update_q, notification_q, app, datastore)
```

### 7.2 动态伸缩

Worker池支持动态调整：

```python
# worker_pool.py:377-436
def adjust_async_worker_count(new_count, update_q=None, notification_q=None, app=None, datastore=None):
    # 增加worker
    for i in range(workers_to_add):
        add_worker(update_q, notification_q, app, datastore)
    # 减少worker
    for _ in range(workers_to_remove):
        remove_worker()
```

### 7.3 健康检查与自愈

每60秒执行worker健康检查：

```python
# flask_app.py:1123-1138
if now - last_health_check > 60:
    health_result = worker_pool.check_worker_health(
        expected_count=expected_workers,
        update_q=update_q,
        notification_q=notification_q,
        app=app,
        datastore=datastore
    )
    # 自动重启崩溃的worker
```

### 7.4 资源隔离

每个Worker在独立线程中运行，拥有自己的asyncio事件循环：

```python
# worker_pool.py:49-67
def run(self):
    self.loop = asyncio.new_event_loop()
    asyncio.set_event_loop(self.loop)
    self.loop.run_until_complete(
        start_single_async_worker(...)
    )
```

这种设计提供了更好的隔离性，单个worker崩溃不会影响其他worker。

## 8. 关键交互流程

### 8.1 完整调度流程

```
┌─────────────────────────────────────────────────────────────┐
│                     Ticker 线程 (每秒)                       │
├─────────────────────────────────────────────────────────────┤
│  1. 检查全局暂停标志                                        │
│  2. 获取正在运行的UUID集合                                  │
│  3. 构建队列中UUID集合（O(1)查找）                          │
│  4. 按 last_checked 升序遍历所有watch                       │
│     ├─ 检查watch是否暂停                                    │
│     ├─ 确定时间窗口配置（全局/watch级）                      │
│     ├─ 调用 is_within_schedule() 检查时间窗口               │
│     ├─ 不在窗口 → continue（跳过）                          │
│     ├─ 计算检查间隔 + 抖动                                  │
│     ├─ 未到间隔 → continue                                  │
│     ├─ 检查代理限制                                        │
│     ├─ 已在运行/队列 → continue                            │
│     └─ 全部通过 → 以 time.time() 为优先级入队               │
│  5. 等待 WAIT_TIME_BETWEEN_LOOP (1秒)                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   RecheckPriorityQueue                      │
├─────────────────────────────────────────────────────────────┤
│  • 基于 heapq 的最小堆                                      │
│  • 优先级 = Unix时间戳（越小越先）                          │
│  • 支持跨线程/跨事件循环访问                                │
│  • 最大容量 5000                                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Worker 池 (N个线程)                       │
├─────────────────────────────────────────────────────────────┤
│  每个Worker独立事件循环                                     │
│  1. 阻塞等待队列任务                                        │
│  2. 获取任务 → 原子认领UUID                                 │
│  3. 认领失败 → 延迟后重入队（优先级×10）                     │
│  4. 认领成功 → 执行抓取                                     │
│  5. 释放UUID → 等待下一个任务                               │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 时间窗口与其他调度因素的协作

时间窗口检查是**准入控制**的第一步，与其他因素的关系：

| 因素 | 关系 | 说明 |
|------|------|------|
| 检查间隔 | 逻辑与 | 必须同时满足 |
| 代理限制 | 逻辑与 | 必须同时满足 |
| 队列上限 | 熔断 | 队列满时停止入队 |
| 全局暂停 | 熔断 | 暂停时停止所有调度 |
| Watch暂停 | 熔断 | 单个watch跳过 |

## 9. 设计权衡与考量

### 9.1 优点

1. **时区正确性**：基于IANA时区数据库，自动处理夏令时
2. **灵活性**：支持全局和watch级配置，覆盖各种使用场景
3. **可预测性**：时间窗口是硬约束，窗口外绝不会执行
4. **负载均衡**：抖动机制分散请求峰值
5. **自愈能力**：Worker健康检查和自动重启

### 9.2 限制

1. **轮询开销**：每秒遍历所有watch，watch数量大时CPU开销增加
2. **精度限制**：秒级调度循环，无法支持亚秒级精度
3. **无窗口记忆**：窗口关闭时正在运行的任务不会被中断
4. **推迟无补偿**：窗口内错过的任务不会在窗口开启时补偿执行

### 9.3 性能优化

1. **批量检查**：每100个watch才检查队列大小
2. **集合查找**：使用set进行O(1)的运行/排队状态检查
3. **排序遍历**：按last_checked排序，优先处理最久未检查的
4. **无锁设计**：使用copy-on-write避免遍历时的并发修改问题

## 10. 代码索引

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 时间窗口核心判断 | `time_handler.py` | 17-114 |
| 调度器主循环 | `flask_app.py` | 1107-1270 |
| 优先级队列实现 | `queue_handlers.py` | 15-200 |
| Worker池管理 | `worker_pool.py` | 1-553 |
| Worker任务处理 | `worker.py` | 46-400 |
| 时间窗口单元测试 | `tests/unit/test_time_handler.py` | 1-600 |
