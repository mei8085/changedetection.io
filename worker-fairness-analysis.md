# 队列 Worker 调度公平性分析报告

## 1. 概述

本文档分析了 changedetection.io 项目中队列 worker 调度系统的公平性问题，包括当前实现机制、潜在问题和改进建议。

## 2. 当前调度机制

### 2.1 队列系统

**队列类型**: `RecheckPriorityQueue`（基于 Python `heapq` 的优先级队列）

**关键文件**:
- `changedetectionio/custom_queue.py` - 队列实现
- `changedetectionio/queue_handlers.py` - 队列处理器

**队列特性**:
- 支持同步和异步接口
- 使用 `heapq` 实现最小堆排序
- 线程安全操作（通过 `threading.Lock` 保护）
- 支持按 UUID 查找位置
- 支持分页获取队列内容

### 2.2 优先级定义

**优先级等级** (`queuedWatchMetaData.py`):

| 优先级值 | 用途 | 触发场景 |
|---------|------|---------|
| 1 | 最高优先级 | 手动触发、重新检查、编辑后立即检查 |
| 5 | 克隆操作 | 复制 watch 后的首次检查 |
| 时间戳 | 定期调度 | 正常的定时检查（`int(time.time())`） |
| ≥1000 | 延迟重试 | UUID 冲突时的延迟处理 |

**优先级计算公式**:
- 定期调度: `priority = int(time.time())`
- 延迟重试: `priority = max(1000, original_priority * 10)`

### 2.3 Ticker 调度线程

**调度逻辑** (`flask_app.py:1107`):

1. **排序策略**: 按 `last_checked` 升序排列，最久未检查的优先
2. **调度周期**: 每 1 秒检查一次（测试环境 0.01 秒）
3. **队列限制**: 最大队列大小 5000
4. **并发控制**: 通过 `running_uuids` 和 `queued_uuids` 集合防止重复

**调度流程**:
```
循环:
    1. 按 last_checked 排序所有 watch
    2. 遍历每个 watch:
        a. 检查是否暂停
        b. 检查是否在调度时间窗口内
        c. 检查是否达到检查间隔（考虑 jitter）
        d. 检查是否已在运行或已在队列
        e. 检查代理使用频率限制
        f. 如果通过所有检查，加入队列（优先级 = 当前时间戳）
```

### 2.4 Worker 池机制

**Worker 管理** (`worker_pool.py`):

- 每个 worker 运行在独立线程和独立事件循环中
- 默认 worker 数量: 10（可通过 `FETCH_WORKERS` 环境变量配置）
- 支持动态调整 worker 数量
- 使用 `ThreadPoolExecutor` 进行队列操作

**UUID 认领机制** (`worker_pool.py:218-257`):

```python
def claim_uuid_for_processing(uuid, worker_id):
    with _uuid_processing_lock:
        if uuid in currently_processing_uuids:
            return False  # 已被其他 worker 认领
        currently_processing_uuids[uuid] = worker_id
        return True
```

**Worker 执行流程** (`worker.py:23-698`):

1. 从队列获取项目（通过 executor 阻塞获取）
2. 原子性认领 UUID
3. 执行页面检查和变更检测
4. 释放 UUID

## 3. 公平性问题分析

### 3.1 问题 1: Heapq 堆排序的不确定性

**现象**:
Python 的 `heapq` 模块实现的是最小堆，但当多个元素具有相同优先级时，它们的弹出顺序是不确定的。

**影响**:
- 同一秒内到期的数百个 watch 执行顺序不可预测
- 先到期的 watch 可能后执行
- 系统行为缺乏可预测性

**代码位置**: `queue_handlers.py:72` (heapq.heappush)

**风险等级**: ⚠️ 中等

### 3.2 问题 2: 时间戳优先级粒度过粗

**现象**:
使用 `int(time.time())` 作为优先级意味着：
- 同一秒内调度的所有 watch 优先级完全相同
- 在高负载系统中，每秒可能调度数百个 watch

**影响**:
- 同一秒内的 watch 执行顺序随机
- 无法保证 FIFO（先进先出）顺序
- 某些 watch 可能被意外延迟

**代码位置**: `flask_app.py:1249`

**风险等级**: ⚠️ 中等

### 3.3 问题 3: 手动触发的优先级垄断

**现象**:
手动触发的 watch 使用优先级 1，远低于定期调度的时间戳优先级（约 17 亿）。

**影响**:
- 如果有大量手动触发的 watch，定期调度的 watch 可能被无限期延迟
- 手动触发总是优先于定期调度，即使定期调度已经超时很久
- 没有手动触发的速率限制

**代码位置**: 
- `flask_app.py:1253` (定期调度使用时间戳)
- `__init__.py:441,475,537` (手动触发使用优先级 1)

**风险等级**: ⚠️ 高

### 3.4 问题 4: 延迟重试的优先级惩罚过重

**现象**:
当 UUID 认领失败（已被其他 worker 处理）时，延迟重试的优先级被设置为：
```python
deferred_priority = max(1000, queued_item_data.priority * 10)
```

**影响**:
- 一个原本优先级为 17 亿的定期调度任务，重试时优先级变为 170 亿
- 这意味着它会排在所有新调度的 watch 之后
- 可能导致某些 watch 被反复延迟，形成饥饿

**代码位置**: `worker.py:74`

**风险等级**: ⚠️ 高

### 3.5 问题 5: 缺少饥饿检测机制

**现象**:
系统没有机制检测某个 watch 是否长时间未被执行。

**影响**:
- 由于堆排序的不确定性，某些 watch 可能被意外地无限期延迟
- 管理员无法发现问题
- 没有自动恢复机制

**风险等级**: ⚠️ 中高

### 3.6 问题 6: 代理限制可能导致的不公平

**现象**:
系统对代理的使用频率有限制：
```python
if time_since_proxy_used < proxy_list_reuse_time_minimum:
    continue  # 跳过这个 watch
```

**影响**:
- 使用相同代理的 watch 可能会被不公平地延迟
- 没有排队机制来确保公平性
- 某些 watch 可能被反复跳过

**代码位置**: `flask_app.py:1230-1246`

**风险等级**: ⚠️ 中等

### 3.7 问题 7: 缺少按标签/组的调度权重

**现象**:
系统目前不支持按 watch 的标签或组设置不同的调度优先级或权重。

**影响**:
- 重要的 watch 无法获得更高的调度优先级
- 无法实现差异化服务质量（QoS）
- 所有 watch 平等竞争资源

**风险等级**: ⚠️ 低-中等

## 4. 改进建议

### 4.1 改进 1: 改进优先级计算，增加顺序保证

**建议**:
将优先级从单一的时间戳改为组合值，确保 FIFO 顺序：

```python
# 方案 A: 使用更高精度的时间戳（毫秒级）
priority = int(time.time() * 1000)  # 毫秒级

# 方案 B: 组合优先级（高32位=时间戳，低32位=计数器）
class SequencedPriority:
    _counter = 0
    @classmethod
    def next(cls, base_priority):
        cls._counter += 1
        return (base_priority << 32) | (cls._counter & 0xFFFFFFFF)
```

**预期效果**:
- 消除同一秒内的优先级冲突
- 保证先进入队列的任务先执行
- 提高系统行为的可预测性

### 4.2 改进 2: 手动触发优先级的动态调整

**建议**:

方案 A - 相对优先级：
```python
# 手动触发使用相对时间戳，而不是固定值 1
manual_priority = int(time.time()) - 3600  # 比当前时间早 1 小时
```

方案 B - 速率限制：
```python
# 限制每秒手动触发的数量
if manual_trigger_count > 10:  # 每秒最多 10 个手动触发
    manual_priority = int(time.time())  # 降级到正常优先级
```

方案 C - 混合队列：
- 维护两个独立队列：手动触发队列和定期调度队列
- worker 按比例从两个队列取任务（如 70% 手动，30% 定期）

**预期效果**:
- 防止手动触发垄断队列
- 保证定期调度任务不会被无限期延迟
- 平衡用户体验和系统公平性

### 4.3 改进 3: 延迟重试的更合理优先级调整

**建议**:
改为使用相对较小的增量而不是乘法：

```python
# 当前实现（问题）
deferred_priority = max(1000, queued_item_data.priority * 10)

# 改进方案
deferred_priority = queued_item_data.priority + 60  # 延迟 60 秒，而不是乘以 10
```

或者使用退避策略：
```python
retry_count = item.get('retry_count', 0)
deferred_priority = queued_item_data.priority + (retry_count * 30)  # 每次重试增加 30 秒
```

**预期效果**:
- 避免优先级的急剧下降
- 重试的 watch 不会被排到太后面
- 减少饥饿的可能性

### 4.4 改进 4: 实现饥饿检测和自动恢复

**建议**:

1. **监控指标**:
```python
# 跟踪每个 watch 的排队时间
class WatchQueueStats:
    def __init__(self):
        self.queue_entry_time = {}
        self.max_queue_time = {}
    
    def record_queue_entry(self, uuid):
        self.queue_entry_time[uuid] = time.time()
    
    def get_queue_time(self, uuid):
        if uuid in self.queue_entry_time:
            return time.time() - self.queue_entry_time[uuid]
        return 0
```

2. **饥饿检测**:
```python
# 在 ticker 线程中
for uuid in queued_uuids:
    queue_time = queue_stats.get_queue_time(uuid)
    if queue_time > 300:  # 超过 5 分钟
        logger.warning(f"Watch {uuid} has been in queue for {queue_time}s - possible starvation!")
        # 提升优先级
        # 或记录告警
```

3. **自动恢复**:
- 对长时间排队的 watch 临时提升优先级
- 或强制插入到队列前面

**预期效果**:
- 及时发现潜在的公平性问题
- 防止系统出现不可见的性能问题
- 提高系统可靠性

### 4.5 改进 5: 实现按标签/组的权重调度

**建议**:

1. **为 watch 添加权重配置**:
```python
# 在 Watch 模型中添加
class Watch:
    scheduling_weight: int = 1  # 1-10，10 为最高优先级
```

2. **修改优先级计算**:
```python
base_priority = int(time.time())
# 权重越高，优先级数值越小（越早执行）
weight_adjustment = (10 - watch.scheduling_weight) * 60  # 每个权重等级相差 60 秒
final_priority = base_priority - weight_adjustment
```

3. **或使用加权轮询**:
- 维护多个优先级队列
- worker 按权重比例从不同队列取任务

**预期效果**:
- 支持差异化服务质量
- 重要的 watch 可以获得更高的优先级
- 更灵活的调度策略

### 4.6 改进 6: 代理使用的公平排队

**建议**:

1. **为每个代理维护独立的队列**:
```python
proxy_queues = {
    'proxy1': [],
    'proxy2': [],
    # ...
}
```

2. **轮询调度**:
```python
# 按代理轮询分发任务
for proxy in proxy_queues:
    if can_use_proxy(proxy):
        task = proxy_queues[proxy].pop(0)
        process_task(task)
```

3. **或令牌桶算法**:
- 每个代理有一个令牌桶
- 每个任务消耗一个令牌
- 令牌按速率补充

**预期效果**:
- 使用相同代理的 watch 之间公平竞争
- 避免某些 watch 被反复跳过
- 更合理的代理资源分配

## 5. 优先级改进矩阵

| 改进项 | 实施难度 | 效果 | 风险 | 建议优先级 |
|-------|---------|------|------|-----------|
| 改进优先级计算（毫秒级） | 低 | ⭐⭐⭐⭐ | 低 | P0 - 立即实施 |
| 延迟重试优先级调整 | 低 | ⭐⭐⭐⭐ | 低 | P0 - 立即实施 |
| 手动触发优先级动态调整 | 中 | ⭐⭐⭐ | 中 | P1 - 近期实施 |
| 饥饿检测机制 | 中 | ⭐⭐⭐ | 低 | P1 - 近期实施 |
| 按标签权重调度 | 高 | ⭐⭐⭐⭐ | 中 | P2 - 规划中 |
| 代理公平排队 | 高 | ⭐⭐⭐ | 中 | P2 - 规划中 |

## 6. 监控建议

为了更好地评估调度公平性，建议添加以下监控指标：

### 6.1 队列指标
- 队列平均等待时间
- 队列等待时间分布（P50, P95, P99）
- 队列大小变化趋势
- 优先级分布统计

### 6.2 Worker 指标
- Worker 利用率
- 任务处理时间分布
- Worker 空闲时间
- UUID 认领冲突率

### 6.3 Watch 级指标
- 每个 watch 的实际检查间隔
- 检查间隔与配置间隔的偏差
- 历史排队时间
- 重试次数统计

## 7. 结论

当前调度系统在大多数情况下工作良好，但在高负载场景下可能出现公平性问题。最关键的改进是：

1. **立即修复**：延迟重试的优先级乘法问题
2. **立即修复**：优先级粒度过粗问题
3. **近期修复**：手动触发优先级垄断问题
4. **监控增强**：添加饥饿检测和相关指标

通过这些改进，系统将在高负载下表现得更加可预测和公平，减少出现 watch 饥饿的可能性。
