# 队列 Worker 调度公平性分析报告（证据化校正 v2.0）

## 文档信息

| 项目 | 内容 |
|------|------|
| 分析日期 | 2026-05-15 |
| 版本 | v2.0（证据化校正版） |
| 分析范围 | 队列实现、优先级设置、调度逻辑、重试机制、所有入口点去重 |
| 验证方式 | 代码静态分析、逐行证据核对、边界条件计算、数学推导 |

---

## 1. 所有入口点去重核对表（v2.0 校正版）

### 1.1 入口点分类总览（共 16 个入口点）

| 分类 | 数量 | 占比 |
|------|------|------|
| ✅ **有去重**（queued + running 检查） | 6 | 37.5% |
| ❌ **无去重**（直接排队，无检查） | 8 | 50.0% |
| 🔄 **新UUID天然不重复** | 2 | 12.5% |

**关键校正**: 
- API/Watch 的 `recheck_all` 批量路径 **确实有去重**（之前版本遗漏）
- 新增 "新UUID天然不重复" 分类，避免过度统计

---

### 1.2 ✅ 有去重的入口点（queued + running 双重检查）

| 序号 | 入口点类型 | 文件位置 | 行号 | 去重代码证据 |
|------|-----------|---------|------|-------------|
| 1 | 定时调度线程 | `flask_app.py` | 1227 | `if not uuid in running_uuids and uuid not in queued_uuids:` |
| 2 | UI单个手动触发 | `blueprint/ui/__init__.py` | 273 | `if worker_pool.is_watch_running(uuid) or uuid in update_q.get_queued_uuids():` |
| 3 | UI批量手动触发 (<20个) | `blueprint/ui/__init__.py` | 299-301 | `if watch_uuid not in queued_uuids and watch_uuid not in running_uuids:` |
| 4 | UI后台线程批量触发 (>=20个) | `blueprint/ui/__init__.py` | 330 | 同上 |
| 5 | API批量 recheck_all (<20个) | `api/Watch.py` | 542-550 | `queued_uuids = set(self.update_q.get_queued_uuids())` <br> `running_uuids = set(worker_pool.get_running_uuids())` <br> 然后过滤 |
| 6 | API批量 recheck_all (>=20个) | `api/Watch.py` | 565-575 | 同样在启动后台线程前捕获 queued_uuids 和 running_uuids |

---

### 1.3 ❌ 无去重的入口点（直接排队，无检查）

| 序号 | 入口点类型 | 文件位置 | 行号 | 风险等级 | 证据 |
|------|-----------|---------|------|---------|------|
| 1 | UI批量操作(recheck) | `blueprint/ui/__init__.py` | 66 | 🟡 中 | 循环内直接排队，无检查：`worker_pool.queue_item_async_safe(update_q, PrioritizedItem(priority=1, item={'uuid': uuid}))` |
| 2 | 编辑后保存触发 | `blueprint/ui/edit.py` | 277 | 🟡 中 | 直接排队，无检查：同上 |
| 3 | Socket.IO实时触发 | `realtime/events.py` | 44 | 🔴 高 | 直接排队，无检查：同上 |
| 4 | 价格数据跟踪触发 | `blueprint/price_data_follower/__init__.py` | 24 | 🟡 中 | 直接排队，无检查：同上 |
| 5 | API单个watch触发 | `api/Watch.py` | 81 | 🔴 高 | 直接排队，无检查：同上 |
| 6 | API标签recheck（所有路径） | `api/Tags.py` | 41-42, 49-50 | 🔴 高 | 直接排队，无检查：同上 |
| 7 | 启动时批量排队 | `__init__.py` | 437-442, 469-480 | 🟢 低 | 启动时理论上无重复，但代码中无检查 |
| 8 | Worker重试机制 | `worker.py` | 74 | 🔴 高 | 重试时直接排队，无检查 |

---

### 1.4 🔄 新UUID天然不重复的入口点

这些入口点处理的是**新创建的 watch**，UUID 是全新生成的，理论上不可能出现重复排队，因此去重检查的必要性较低。

| 序号 | 入口点类型 | 文件位置 | 行号 | 说明 |
|------|-----------|---------|------|------|
| 1 | Clone后触发 | `blueprint/ui/__init__.py` | 257 | 新 UUID 刚生成，不可能在队列中 |
| 2 | 添加新watch触发 | `blueprint/ui/views.py` | 41 | 新 UUID 刚生成，不可能在队列中 |

---

### 1.5 去重覆盖率修正统计

| 统计项 | 修正前(v1.0) | 修正后(v2.0) |
|--------|-------------|-------------|
| 总入口点数量 | 14 | 16 |
| 有去重的入口点 | 4 (28.6%) | 6 (37.5%) |
| 无去重的入口点 | 10 (71.4%) | 8 (50.0%) |
| 新UUID天然不重复 | - | 2 (12.5%) |

**关键发现**:
- API 批量 recheck_all 确实有去重，修正了 v1.0 的错误
- 新增分类后，无去重的比例从 71.4% 下降到 50%
- 但仍有 8 个高风险入口点完全无去重保护

---

## 2. 永久饥饿结论的成立前提复核（v2.0 校正版）

### 2.1 前提 1：优先级差距导致的绝对优先级分层

**数学依据**:
- 手动触发优先级: `P_manual = 1`
- 定时任务优先级: `P_scheduled = T`，其中 T 是当前时间戳（约 1,700,000,000）
- 优先级差距: `P_scheduled / P_manual = 1.7 × 10^9`

**最小堆排序性质定理**:
> 在最小堆优先级队列中，对于任意两个元素 A 和 B，如果 `priority(A) < priority(B)`，则 A 一定在 B 之前被处理。

**推论**:
- 任何 `priority = 1` 的任务一定在任何 `priority > 1` 的任务之前被处理
- 只要队列中存在手动触发任务，定时任务就不会被处理

**结论**: ✅ 前提成立，证据充分

---

### 2.2 前提 2：手动触发速率可能超过系统处理速率

**系统处理能力估算**:
- 假设每个任务平均处理时间: 2 秒
- 10 个 worker: 处理速率 = 5 任务/秒
- 20 个 worker: 处理速率 = 10 任务/秒

**高风险无去重入口点可能的触发速率**:
1. **API单个watch触发** (`api/Watch.py:81`) - 无速率限制，自动化脚本可能达到 10+ 次/秒
2. **API标签recheck** (`api/Tags.py:41-42`) - 一次可能触发成百上千个任务
3. **Socket.IO实时触发** (`realtime/events.py:44`) - 无速率限制
4. **UI批量操作(recheck)** (`blueprint/ui/__init__.py:66`) - 一次可能触发所有watch

**临界条件证明**:
```
当 手动触发速率 > 系统处理速率 时
队列积压 = ∫(手动触发速率 - 处理速率) dt
=> 队列无限增长，定时任务永远饥饿
```

**结论**: ✅ 前提成立，系统存在达到临界条件的可能（特别是无去重入口点可能被滥用）

---

### 2.3 前提 3：重试机制的优先级乘法放大

**当前重试逻辑** (`worker.py:74`):
```python
deferred_priority = max(1000, queued_item_data.priority * 10)
```

**优先级演变数学模型**:
设原始优先级为 P₀，第 k 次重试后的优先级为 P_k:

```
P_k = P₀ × 10^k
```

**重试时无去重检查**:
- 重试时直接排队，不检查是否已在队列中
- 可能导致同一UUID在队列中出现多次，且优先级越来越低

**结论**: ✅ 前提成立，乘法放大确实存在且重试无去重

---

### 2.4 前提 4：无优先级老化机制

**检查结果**:
- 队列中没有任何机制提升长时间等待任务的优先级
- 任务一旦进入队列，其优先级值永远不变
- 没有 "等待时间加权" 或 "优先级老化" 算法

**结论**: ✅ 前提成立，系统完全没有优先级老化机制

---

### 2.5 前提 5：部分入口点无去重保护，可能加剧队列膨胀

**v2.0 校正后证据**:
- 8 个入口点无去重保护（占 50%）
- 特别是 API 单个 recheck 和标签 recheck 完全无保护
- 重试机制也无去重保护，可能导致重复排队
- 队列膨胀会进一步加剧优先级饥饿问题

**结论**: ✅ 前提成立，去重保护不足确实可能加剧饥饿

---

## 3. 相对优先级边界：定时任务 vs 手动重试（v2.0 保持不变）

### 3.1 定时任务的重试优先级演变

假设当前时间戳 T = 1,700,000,000

| 重试次数 k | 优先级值 P_k | 相对值 (P_k / T) | 排在未来多少秒的定时任务之后 | 影响 |
|-----------|-------------|-----------------|--------------------------|------|
| 0 (原始) | 1,700,000,000 | 1.0x | 基准（当前时间） | 正常定时任务 |
| 1 | 17,000,000,000 | 10x | 约 153 亿秒 = 485 年 | 实际上等于永久丢弃 |
| 2 | 170,000,000,000 | 100x | 约 1683 亿秒 = 5337 年 | 永久丢弃 |
| 3 | 1,700,000,000,000 | 1000x | 约 1.68 万亿秒 = 53370 年 | 永久丢弃 |
| 4 | 17,000,000,000,000 | 10000x | 天文数字 | 永久丢弃 |

**数学证明定时任务重试后的饥饿**:

设重试 k 次后任务优先级为 `P_retry = T × 10^k`

新产生的定时任务优先级为 `P_new = T + Δt`，其中 `Δt` 是从重试到新任务产生的时间差

由于 `10^k >> Δt/T` 对于任何合理的 Δt（如 1 年）都成立:
```
P_retry = T × 10^k >> T + Δt = P_new
```

因此重试的定时任务会排在所有未来产生的定时任务之后。

**临界分析**:
- 即使只重试 1 次，任务也已实际上被永久丢弃
- 重试机制的设计缺陷导致了永久饥饿，而不仅仅是延迟

---

### 3.2 手动触发任务的重试优先级演变

| 重试次数 k | 优先级值 P_k | 相对定时任务 (P_k / T) | 与新定时任务的关系 | 备注 |
|-----------|-------------|----------------------|-------------------|------|
| 0 (原始) | 1 | ~0.0000000006x | **优先于所有定时任务** | 正常手动触发 |
| 1 | max(1000, 1×10) = 1000 | ~0.0000006x | **仍然优先于所有定时任务** | 优先级仍然极低 |
| 2 | max(1000, 1000×10) = 10000 | ~0.000006x | **仍然优先于所有定时任务** | 还是极低 |
| 3 | 100000 | ~0.00006x | **仍然优先于所有定时任务** | |
| 4 | 1000000 | ~0.0006x | **仍然优先于所有定时任务** | |
| 5 | 10000000 | ~0.006x | **仍然优先于所有定时任务** | |
| 6 | 100000000 | ~0.06x | **仍然优先于所有定时任务** | |
| 7 | 1000000000 | ~0.6x | **仍然优先于所有定时任务** | |
| 8 | 10000000000 | ~6x | 开始落后于新的定时任务 | 需要 8 次重试才会落后 |

**关键发现**:
- 手动触发任务需要重试 **8 次** 才会开始落后于新的定时任务
- 在前 7 次重试中，它仍然优先于所有定时任务
- 这意味着频繁冲突的手动触发任务会长期占用队列头部位置

---

### 3.3 优先级边界交叉点分析

| 任务类型 | 优先级范围 | 与定时任务的关系 |
|---------|-----------|-----------------|
| 原始手动触发 | 1 | 完全优先 |
| 重试 1-7 次的手动触发 | 10 - 10^7 | 仍然完全优先 |
| 重试 8+ 次的手动触发 | ≥ 10^8 | 开始与定时任务交叉 |
| 原始定时任务 | ~1.7×10^9 | 基准 |
| 重试 1 次的定时任务 | ~1.7×10^10 | 永久落后 |
| 重试 2+ 次的定时任务 | ~1.7×10^11+ | 永久丢弃 |

**边界结论**:
1. **手动触发任务享有近乎绝对的优先级特权**，即使多次重试也不会轻易失去优先地位
2. **定时任务一旦重试就几乎等于被丢弃**，优先级被放大到天文数字
3. **两种任务的优先级范围几乎完全不重叠**，导致了绝对的不公平

---

## 4. 修复优先级重排序（v2.0 基于修正证据）

### 4.1 严重程度评估矩阵

| 问题 | 数据丢失风险 | 系统可用性风险 | 公平性影响 | 修复紧迫性 | 综合优先级 |
|------|-------------|---------------|-----------|----------|-----------|
| **重试优先级乘法放大** | 🔴 极高 | 🔴 高 | 🔴 极高 | 立即 | **P0** |
| **手动触发固定优先级 1** | 🟢 低 | 🔴 极高 | 🔴 极高 | 立即 | **P0** |
| **API单个recheck无去重** | 🟡 中 | 🔴 高 | 🟡 中 | 立即 | **P0** |
| **API标签recheck无去重** | 🟡 中 | 🔴 高 | 🟡 中 | 立即 | **P0** |
| **重试机制无去重** | 🟡 中 | 🔴 高 | 🟡 中 | 立即 | **P0** |
| **Socket.IO无去重** | 🟡 中 | 🔴 高 | 🟡 中 | 近期 | **P1** |
| **优先级粒度太粗** | 🟢 低 | 🟡 中 | 🟡 中 | 近期 | **P1** |
| **缺少饥饿检测** | 🟢 低 | 🟡 中 | 🟡 中 | 中期 | **P1** |
| **缺少优先级老化机制** | 🟢 低 | 🟡 中 | 🔴 高 | 中期 | **P1** |
| **其他入口点去重完善** | 🟢 低 | 🟡 中 | 🟡 中 | 中期 | **P2** |
| **按标签权重调度** | 🟢 低 | 🟢 低 | 🟡 中 | 长期 | **P2** |
| **代理公平排队** | 🟢 低 | 🟢 低 | 🟡 中 | 长期 | **P2** |

---

### 4.2 P0 - 必须立即修复（2周内）

#### P0.1 修复重试优先级乘法放大

**问题位置**: `worker.py:74`

**当前代码**:
```python
deferred_priority = max(1000, queued_item_data.priority * 10)
```

**修复方案**（推荐）：
```python
# 改为加法增量，每次重试延迟 60 秒
deferred_priority = queued_item_data.priority + 60

# 或者更好，带上限的指数退避：
retry_count = queued_item_data.item.get('retry_count', 0) + 1
queued_item_data.item['retry_count'] = retry_count
# 最大延迟 10 分钟，避免无限膨胀
backoff_seconds = min(30 * (2 ** retry_count), 600)
deferred_priority = queued_item_data.item.get('original_priority', queued_item_data.priority) + backoff_seconds
```

**修复后效果**:
| 重试次数 | 延迟 (秒) | 最大延迟 |
|---------|----------|---------|
| 1 | 60 | 600 秒 (10 分钟) |
| 2 | 120 | 600 秒 |
| 3 | 240 | 600 秒 |
| 4+ | 480+ | 600 秒上限 |

---

#### P0.2 修复手动触发固定优先级 1

**问题**: 手动触发与定时任务的优先级差距达 17 亿倍，导致绝对饥饿

**修复方案**（相对时间戳偏移 + 速率限制）:
```python
# 手动触发比当前时间早 1 小时，确保优先但不垄断
manual_priority = int(time.time()) - 3600

# 或者带速率限制的动态优先级
from collections import deque
import time

class ManualPriorityManager:
    def __init__(self):
        self.recent_triggers = deque()
        self.rate_limit = 5  # 每秒最多 5 个高优先级
        self.normal_priority_offset = 3600  # 正常优先级：提前 1 小时
        self.throttled_priority_offset = 60  # 超过速率限制时：提前 1 分钟
    
    def get_priority(self):
        now = time.time()
        # 清理 1 秒前的记录
        while self.recent_triggers and self.recent_triggers[0] < now - 1:
            self.recent_triggers.popleft()
        
        if len(self.recent_triggers) < self.rate_limit:
            # 速率内，高优先级
            self.recent_triggers.append(now)
            return int(now) - self.normal_priority_offset
        else:
            # 超过速率限制，降级优先级
            return int(now) - self.throttled_priority_offset
```

**修复后的优先级分布**:
| 任务类型 | 优先级值 | 相对关系 |
|---------|---------|---------|
| 速率内手动触发 | T - 3600 | 优先于最近 1 小时内的定时任务 |
| 超速率手动触发 | T - 60 | 优先于最近 1 分钟内的定时任务 |
| 新定时任务 | T | 基准 |
| 重试定时任务 | T + 60 ~ T + 600 | 最多延迟 10 分钟 |

---

#### P0.3 为 API 单个 recheck 添加去重保护

**问题位置**: `api/Watch.py:81`

**当前代码**:
```python
if request.args.get('recheck'):
    worker_pool.queue_item_async_safe(self.update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
    return "OK", 200
```

**修复代码**:
```python
if request.args.get('recheck'):
    # 添加去重检查
    if worker_pool.is_watch_running(uuid) or uuid in self.update_q.get_queued_uuids():
        return {'status': 'Watch already queued or being checked'}, 409
    worker_pool.queue_item_async_safe(self.update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
    return "OK", 200
```

---

#### P0.4 为 API 标签 recheck 添加去重保护

**问题位置**: `api/Tags.py:41-42, 49-50`

**当前代码** (两处类似):
```python
for watch_uuid in watches_to_queue:
    worker_pool.queue_item_async_safe(self.update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```

**修复代码** (同步和后台线程都需要):
```python
# 在排队前添加去重检查
queued_uuids = set(self.update_q.get_queued_uuids())
running_uuids = set(worker_pool.get_running_uuids())

watches_to_queue_filtered = [
    uuid for uuid in watches_to_queue
    if uuid not in queued_uuids and uuid not in running_uuids
]

for watch_uuid in watches_to_queue_filtered:
    worker_pool.queue_item_async_safe(self.update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```

---

#### P0.5 为重试机制添加去重保护

**问题位置**: `worker.py:74`

**修复代码**:
```python
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # 已在处理中 - 重新排队并延迟
    await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)  # 10 秒
    
    # vv 新增：检查是否已在队列中，避免重复排队 vv
    if uuid in q.get_queued_uuids():
        logger.debug(f"Worker {worker_id}: UUID {uuid} already in queue, skipping requeue")
        continue
    # ^^ 新增结束 ^^
    
    deferred_priority = queued_item_data.priority + 60  # 同时修复乘法放大
    deferred_item = PrioritizedItem(priority=deferred_priority, item=queued_item_data.item)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

---

### 4.3 P1 - 近期需要修复（1-2个月内）

#### P1.1 为 Socket.IO 实时触发添加去重保护

**问题位置**: `realtime/events.py:44`

**修复代码**:
```python
# 在排队前添加去重检查
if worker_pool.is_watch_running(uuid) or uuid in update_q.get_queued_uuids():
    logger.info(f"Socket.IO: Watch {uuid} already queued or running, skipping")
    emit('operation_result', {'success': False, 'error': 'Watch already queued or being checked'})
    return

worker_pool.queue_item_async_safe(update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
```

---

#### P1.2 提升优先级粒度到毫秒级

**当前问题**: 使用秒级时间戳导致同一秒内任务顺序不确定

**修复位置**: `flask_app.py:1249`

**修复代码**:
```python
# 使用毫秒级时间戳
priority = int(time.time() * 1000)
```

**注意**: 需要确保所有地方的优先级计算保持一致，避免整数溢出问题。

---

#### P1.3 添加饥饿检测和告警机制

**实现建议**:
```python
class StarvationDetector:
    def __init__(self, threshold_seconds=300):  # 5 分钟阈值
        self.threshold = threshold_seconds
        self.first_queued_time = {}
    
    def record_queue_entry(self, uuid, priority):
        if uuid not in self.first_queued_time:
            self.first_queued_time[uuid] = time.time()
    
    def record_processing(self, uuid):
        if uuid in self.first_queued_time:
            wait_time = time.time() - self.first_queued_time[uuid]
            if wait_time > self.threshold:
                logger.warning(f"WATCH STARVATION ALERT: UUID {uuid} waited {wait_time:.1f}s in queue")
            del self.first_queued_time[uuid]
    
    def get_starvation_candidates(self):
        now = time.time()
        return [
            uuid for uuid, first_time in self.first_queued_time.items()
            if now - first_time > self.threshold
        ]
```

---

#### P1.4 实现优先级老化机制

**老化算法**: 任务在队列中每等待 N 秒，优先级提升（数值减小）一定量

```python
# 在每次从队列获取时检查并更新优先级
def get_with_aging(self, block=True, timeout=None):
    with self._lock:
        # 扫描并应用老化
        now = time.time()
        for item in self._priority_items:
            if hasattr(item, 'enqueue_time') and now - item.enqueue_time > 60:  # 1 分钟后开始老化
                # 每等待 1 分钟，优先级提升相当于提前 10 秒
                age_minutes = (now - item.enqueue_time) // 60
                item.priority = max(1, item.original_priority - int(age_minutes * 10))
        
        # 然后正常获取最高优先级的项目
        if self._priority_items:
            return heapq.heappop(self._priority_items)
```

---

### 4.4 P2 - 中期和长期规划

1. **其他入口点去重完善**（UI批量操作、编辑后保存等）- 风险较低，可在常规开发中完成
2. **按标签/组的调度权重** - 为不同重要性的 watch 设置不同的优先级基础偏移
3. **代理使用的公平排队** - 确保使用相同代理的 watch 公平共享资源
4. **动态 worker 数量调整** - 根据队列积压自动扩缩容

---

## 5. 修复后的预期效果验证

### 5.1 公平性指标改进

| 指标 | 修复前 | 修复后 | 改进幅度 |
|------|-------|-------|---------|
| 定时任务最大等待时间 | 无限（手动触发持续时） | 约 1-2 分钟（在正常负载下） | ✅✅✅✅✅ |
| 重试任务的最大延迟 | 永久丢弃 | 最大 10 分钟 | ✅✅✅✅✅ |
| 手动触发与定时任务的优先级比 | 1 : 1,700,000,000 | 约 1 : 60（可控） | ✅✅✅✅✅ |
| 无去重入口点数量 | 8 | 0 | ✅✅✅✅✅ |
| 去重覆盖率 | 37.5% | 100%（含天然不重复） | ✅✅✅✅✅ |
| 相同优先级任务顺序 | 不确定 | 确定（毫秒级时间戳） | ✅✅✅ |
| 饥饿自动恢复 | 无 | 有（老化机制） | ✅✅✅✅✅ |
| 队列重复排队风险 | 高 | 低 | ✅✅✅✅✅ |

---

### 5.2 验收测试用例

#### 测试用例 1：API单个recheck去重验证
```
条件:
- 对同一个正在运行的 watch 连续调用 API recheck 3 次

期望结果:
- 只有第一次成功排队
- 后两次返回 409 状态码
- 队列中不会出现重复 UUID
```

#### 测试用例 2：API标签recheck去重验证
```
条件:
- 有 10 个 watch 带同一个标签
- 其中 3 个正在运行或已在队列中
- 调用标签 recheck API

期望结果:
- 只有 7 个 watch 被排队
- 不会重复排队已在队列或运行中的 watch
```

#### 测试用例 3：手动触发速率限制验证
```
条件:
- 每秒连续触发 10 次手动 recheck
- 10 个 worker 正常工作
- 有定时任务等待执行

期望结果:
- 前 5 次使用高优先级（T - 3600）
- 后 5 次使用降级优先级（T - 60）
- 定时任务仍然能在合理时间内获得执行
```

#### 测试用例 4：重试任务延迟上限验证
```
条件:
- 一个任务连续发生 5 次 UUID 认领冲突
- 每次都触发重试机制

期望结果:
- 第 5 次重试的优先级不超过原始优先级 + 600 秒
- 任务最终能在 10 分钟内获得执行
- 重试过程中不会出现重复排队（去重检查生效）
```

#### 测试用例 5：饥饿检测告警验证
```
条件:
- 一个 watch 在队列中等待超过 5 分钟

期望结果:
- 系统记录 WARNING 级别的饥饿告警日志
- 如果启用老化机制，任务优先级被提升
```

---

## 附录：相关代码位置索引

| 功能模块 | 文件 | 关键行号 | 状态 |
|---------|------|---------|------|
| 队列核心实现 | `queue_handlers.py` | 15-260 | 基础 |
| Worker 主循环 | `worker.py` | 23-698 | 基础 |
| **重试优先级乘法（需修复）** | `worker.py` | 74 | P0 |
| **重试无去重（需修复）** | `worker.py` | 70-77 | P0 |
| Worker 池管理 | `worker_pool.py` | 全部 | 基础 |
| 定时调度线程（有去重） | `flask_app.py` | 1227, 1249 | ✅ 无需修复 |
| 优先级数据结构 | `queuedWatchMetaData.py` | 7-10 | 基础 |
| UI单个手动触发（有去重） | `blueprint/ui/__init__.py` | 273 | ✅ 无需修复 |
| UI批量手动触发（有去重） | `blueprint/ui/__init__.py` | 299-301, 330 | ✅ 无需修复 |
| **UI批量操作(无去重)** | `blueprint/ui/__init__.py` | 66 | P2 |
| **Clone后触发（天然不重复）** | `blueprint/ui/__init__.py` | 257 | 🔄 无需修复 |
| **编辑后保存触发(无去重)** | `blueprint/ui/edit.py` | 277 | P2 |
| **添加新watch触发(天然不重复)** | `blueprint/ui/views.py` | 41 | 🔄 无需修复 |
| **Socket.IO触发(无去重)** | `realtime/events.py` | 44 | P1 |
| **价格数据跟踪触发(无去重)** | `blueprint/price_data_follower/__init__.py` | 24 | P2 |
| **API单个watch触发(无去重)** | `api/Watch.py` | 81 | P0 |
| API批量recheck_all（有去重） | `api/Watch.py` | 542-550, 565-575 | ✅ 无需修复 |
| **API标签recheck(无去重)** | `api/Tags.py` | 41-42, 49-50 | P0 |
| **启动时批量排队(无去重)** | `__init__.py` | 437-442, 469-480 | P2 |

---

## 修订历史

| 版本 | 日期 | 修改内容 | 修改人 |
|------|------|---------|-------|
| v1.0 | 2026-05-15 | 初始版本，基础分析 | 开发团队 |
| v2.0 | 2026-05-15 | **证据化校正**：逐行核对所有16个入口点去重，发现 API 批量 recheck_all 确实有去重，新增"新UUID天然不重复"分类，修正覆盖率统计，重排序修复优先级 | 开发团队 |
