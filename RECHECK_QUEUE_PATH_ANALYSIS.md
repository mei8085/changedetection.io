# 手动 recheck 入队路径深度分析报告

## 一、三类 priority=1 触发通路

### 1.1 路径汇总

| 触发路径 | API端点 | 代码位置 | priority值 |
|----------|---------|----------|------------|
| **单个watch recheck** | `/api/v1/watch/<uuid>?recheck=true` | `api/Watch.py:80-82` | 1 |
| **批量 recheck_all** | `/api/v1/watch?recheck_all=1` | `api/Watch.py:536-589` | 1 |
| **Tag级recheck** | `/api/v1/tag/<uuid>?recheck=true` | `api/Tags.py:29-59` | 1 |
| **Socket.IO触发** | 事件 `watch_operation` + `op='recheck'` | `realtime/events.py:38-45` | 1 |

---

## 二、各路径入队前检查对比

### 2.1 单个watch recheck (`?recheck=true`)

**代码位置**: `api/Watch.py:80-82`

```python
if request.args.get('recheck'):
    worker_pool.queue_item_async_safe(self.update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
    return "OK", 200
```

**入队前检查**:
| 检查项 | 是否检查 | 说明 |
|--------|----------|------|
| running_uuids | ✗ | 不检查 |
| queued_uuids | ✗ | 不检查 |
| 暂停状态 | ✗ | 不检查 |
| 时间间隔 | ✗ | 不检查 |

**特点**: 无条件直接入队，无任何去重检查

---

### 2.2 批量 recheck_all (`?recheck_all=1`)

**代码位置**: `api/Watch.py:536-589`

```python
if request.args.get('recheck_all'):
    watches_to_queue = self.datastore.data['watching'].keys()

    if len(watches_to_queue) < 20:
        # 同步入队
        queued_uuids = set(self.update_q.get_queued_uuids())
        running_uuids = set(worker_pool.get_running_uuids())

        watches_to_queue_filtered = [
            uuid for uuid in watches_to_queue
            if uuid not in queued_uuids and uuid not in running_uuids
        ]

        for uuid in watches_to_queue_filtered:
            worker_pool.queue_item_async_safe(self.update_q, 
                queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
    else:
        # 异步入队（后台线程）
        queued_uuids = set(self.update_q.get_queued_uuids())
        running_uuids = set(worker_pool.get_running_uuids())

        def queue_all_watches_background():
            for uuid in watches_to_queue:
                if uuid not in queued_uuids and uuid not in running_uuids:
                    worker_pool.queue_item_async_safe(self.update_q, 
                        queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
```

**入队前检查**:
| 检查项 | 是否检查 | 说明 |
|--------|----------|------|
| running_uuids | ✓ | 调用 `worker_pool.get_running_uuids()` |
| queued_uuids | ✓ | 调用 `update_q.get_queued_uuids()` |
| 暂停状态 | ✗ | 不检查 |
| 时间间隔 | ✗ | 不检查 |

**去重发生位置**: API层，入队前过滤

---

### 2.3 Tag级recheck (`/api/v1/tag/<uuid>?recheck=true`)

**代码位置**: `api/Tags.py:29-59`

```python
if request.args.get('recheck'):
    watches_to_queue = []
    for k in sorted(self.datastore.data['watching'].items(), key=lambda item: item[1].get('last_checked', 0)):
        watch_uuid = k[0]
        watch = k[1]
        if not watch['paused'] and tag['uuid'] in watch['tags']:
            watches_to_queue.append(watch_uuid)

    if len(watches_to_queue) < 20:
        for watch_uuid in watches_to_queue:
            worker_pool.queue_item_async_safe(self.update_q, 
                queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
    else:
        def queue_watches_background():
            for watch_uuid in watches_to_queue:
                worker_pool.queue_item_async_safe(self.update_q, 
                    queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```

**入队前检查**:
| 检查项 | 是否检查 | 说明 |
|--------|----------|------|
| running_uuids | ✗ | 不检查 |
| queued_uuids | ✗ | 不检查 |
| 暂停状态 | ✓ | `if not watch['paused']` |
| 时间间隔 | ✗ | 不检查 |

**特点**: 仅检查暂停状态，不检查运行/队列状态

---

### 2.4 Socket.IO触发 recheck

**代码位置**: `realtime/events.py:38-45`

```python
elif op == 'recheck':
    from changedetectionio.flask_app import update_q
    from changedetectionio import queuedWatchMetaData
    from changedetectionio import worker_pool
    
    worker_pool.queue_item_async_safe(update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
```

**入队前检查**:
| 检查项 | 是否检查 | 说明 |
|--------|----------|------|
| running_uuids | ✗ | 不检查 |
| queued_uuids | ✗ | 不检查 |
| 暂停状态 | ✗ | 不检查 |
| 时间间隔 | ✗ | 不检查 |

**特点**: 无条件直接入队，无任何去重检查

---

## 三、Worker 层去重处理

### 3.1 Claim 机制

**代码位置**: `worker.py:67-77`

```python
# CRITICAL: Claim UUID immediately after getting from queue to prevent race condition
uuid = queued_item_data.item.get('uuid')
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # Already being processed - re-queue and continue
    logger.trace(f"Worker {worker_id} detected UUID {uuid} already processing during claim - deferring")
    await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)
    deferred_priority = max(1000, queued_item_data.priority * 10)
    deferred_item = PrioritizedItem(priority=deferred_priority, item=queued_item_data.item)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

**去重逻辑**:
1. Worker从队列获取任务后立即调用 `claim_uuid_for_processing()`
2. 如果UUID已被其他Worker claim，则:
   - 延迟 `DEFER_SLEEP_TIME_ALREADY_QUEUED` 秒
   - 提升优先级为 `max(1000, priority * 10)`
   - 重新入队

**去重发生位置**: Worker层，claim失败后延迟重入队

---

## 四、定时调度路径对比

### 4.1 定时调度入队逻辑

**代码位置**: `flask_app.py:1107-1268`

```python
for uuid in watch_uuid_list:
    # 检查暂停状态
    if watch['paused']:
        continue

    # 检查时间调度限制
    if time_schedule_limit and time_schedule_limit.get('enabled'):
        result = is_within_schedule(...)
        if not result:
            continue

    # 检查时间间隔
    threshold = recheck_time_system_seconds if watch.get('time_between_check_use_default') else watch.threshold_seconds()
    seconds_since_last_recheck = now - watch['last_checked']
    
    if seconds_since_last_recheck >= (threshold + watch.jitter_seconds) \
       and seconds_since_last_recheck >= recheck_time_minimum_seconds:
        
        # 检查运行/队列状态
        if not uuid in running_uuids and uuid not in queued_uuids:
            
            # 检查代理复用时间
            if watch_proxy:
                time_since_proxy_used = int(time.time() - proxy_last_used_time)
                if time_since_proxy_used < proxy_list_reuse_time_minimum:
                    continue
                
            # 入队
            priority = int(time.time())
            queuedWatchMetaData.PrioritizedItem(priority=priority, item={'uuid': uuid})
```

---

## 五、入队条件逐项对照

### 5.1 完整对比表

| 检查项 | 单个recheck | 批量recheck_all | Tag级recheck | Socket.IO | 定时调度 |
|--------|-------------|-----------------|--------------|-----------|----------|
| **优先级** | 1 | 1 | 1 | 1 | timestamp |
| **running_uuids检查** | ✗ | ✓ | ✗ | ✗ | ✓ |
| **queued_uuids检查** | ✗ | ✓ | ✗ | ✗ | ✓ |
| **暂停状态检查** | ✗ | ✗ | ✓ | ✗ | ✓ |
| **时间间隔检查** | ✗ | ✗ | ✗ | ✗ | ✓ |
| **最小间隔检查** | ✗ | ✗ | ✗ | ✗ | ✓ |
| **时间窗口检查** | ✗ | ✗ | ✗ | ✗ | ✓ |
| **代理复用检查** | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Worker claim去重** | ✓(间接) | ✓(间接) | ✓(间接) | ✓(间接) | ✓(间接) |

### 5.2 去重机制总结

| 去重层级 | 覆盖路径 | 实现方式 |
|----------|----------|----------|
| **API层去重** | 批量recheck_all | 入队前过滤 running_uuids + queued_uuids |
| **Worker层去重** | 所有路径 | claim失败后延迟重入队 |

---

## 六、流程图：去重机制整体视图

```
用户触发 recheck
       │
       ├── 单个watch recheck ────→ 直接入队(priority=1) ────┐
       ├── 批量recheck_all ────→ 过滤去重 ────→ 入队(priority=1) ────┐
       ├── Tag级recheck ────→ 暂停检查 ────→ 直接入队(priority=1) ────┐
       └── Socket.IO ────→ 直接入队(priority=1) ────┐
                                                     │
                                                     ↓
                                           Worker获取任务
                                                     │
                                                     ↓
                                           claim_uuid_for_processing()
                                                     │
                                            ┌────────┴────────┐
                                            ↓                 ↓
                                        claim成功          claim失败
                                            │                 │
                                            ↓                 ↓
                                        执行抓取        延迟重入队(priority*10)
```

---

## 七、关键发现

### 7.1 潜在问题

| 问题 | 影响路径 | 风险描述 |
|------|----------|----------|
| **重复入队** | 单个recheck、Socket.IO、Tag级recheck | 同一watch可能被多次入队 |
| **无暂停检查** | 单个recheck、批量recheck_all、Socket.IO | 暂停的watch也会被入队 |
| **优先级相同** | 所有priority=1路径 | 大量recheck请求可能阻塞队列 |

### 7.2 Worker层最终去重

即使前端多次触发同一watch的recheck，Worker层的claim机制会确保同一时刻只有一个Worker处理该watch：

```python
# worker.py:70-77
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # 已被处理，延迟重入队
    deferred_priority = max(1000, queued_item_data.priority * 10)
    deferred_item = PrioritizedItem(priority=deferred_priority, item=queued_item_data.item)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

### 7.3 延迟重入队机制

当claim失败时：
1. 等待 `DEFER_SLEEP_TIME_ALREADY_QUEUED` 秒（默认短延迟）
2. 优先级提升为 `max(1000, priority * 10)`
   - 原priority=1 → 新priority=10
   - 原priority=timestamp → 新priority=timestamp*10 或 1000

---

## 八、总结

### 8.1 入队检查策略差异

| 路径 | 检查策略 | 设计意图 |
|------|----------|----------|
| 单个recheck | 无检查 | 即时响应，信任用户意图 |
| 批量recheck_all | 完整去重 | 避免重复处理，优化资源使用 |
| Tag级recheck | 仅暂停检查 | 尊重用户暂停设置，快速响应 |
| Socket.IO | 无检查 | 实时性优先，简化逻辑 |
| 定时调度 | 完整检查链 | 系统级调度，严格控制 |

### 8.2 去重机制层次

```
┌─────────────────────────────────────────────────────────────────┐
│                     去重机制层次                               │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: API层去重（仅批量recheck_all）                      │
│           - 入队前过滤 running_uuids + queued_uuids           │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Worker层去重（所有路径）                            │
│           - claim_uuid_for_processing()                      │
│           - 失败后延迟重入队                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 改进建议

1. **统一入队检查**: 为所有recheck路径添加基础检查（暂停状态、运行状态）
2. **优先级差异化**: 根据触发来源设置不同优先级，避免高优先级阻塞
3. **幂等性保证**: 添加请求去重机制，防止重复触发
