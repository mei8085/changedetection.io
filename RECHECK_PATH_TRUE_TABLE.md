# 手动 recheck 全量通路真值表分析报告

## 一、全量通路汇总

### 1.1 通路分类

| 通路编号 | 触发路径 | API端点/事件 | 代码位置 |
|----------|----------|--------------|----------|
| **R1** | API 单watch recheck | `/api/v1/watch/<uuid>?recheck=true` | `api/Watch.py:80-82` |
| **R2** | API recheck_all | `/api/v1/watch?recheck_all=1` | `api/Watch.py:536-589` |
| **R3** | UI /checknow (单watch) | `/checknow?uuid=<uuid>` | `blueprint/ui/__init__.py:271-277` |
| **R4** | UI /checknow (批量) | `/checknow?tag=<tag>&with_errors=<0/1>` | `blueprint/ui/__init__.py:278-338` |
| **R5** | API Tag级recheck | `/api/v1/tag/<uuid>?recheck=true` | `api/Tags.py:29-59` |
| **R6** | Socket.IO recheck | `watch_operation` 事件 | `realtime/events.py:38-45` |

---

## 二、入队前检查真值表

### 2.1 完整真值表

| 通路 | priority | running_uuids | queued_uuids | paused | 代码位置 |
|------|----------|---------------|--------------|--------|----------|
| **R1: API单watch** | 1 | ✗ | ✗ | ✗ | `api/Watch.py:80-82` |
| **R2: API recheck_all** | 1 | ✓ | ✓ | ✗ | `api/Watch.py:536-589` |
| **R3: UI /checknow (单)** | 1 | ✓ | ✓ | ✗ | `blueprint/ui/__init__.py:273` |
| **R4: UI /checknow (批量)** | 1 | ✓ | ✓ | ✓ | `blueprint/ui/__init__.py:284,294-295` |
| **R5: API Tag级** | 1 | ✗ | ✗ | ✓ | `api/Tags.py:36` |
| **R6: Socket.IO** | 1 | ✗ | ✗ | ✗ | `realtime/events.py:38-45` |
| **S1: 定时调度** | timestamp | ✓ | ✓ | ✓ | `flask_app.py:1107-1268` |

### 2.2 各通路检查逻辑详解

**R1: API单watch recheck**

```python
# api/Watch.py:80-82
if request.args.get('recheck'):
    worker_pool.queue_item_async_safe(self.update_q, 
        queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
    return "OK", 200
```
- 无任何检查，直接入队

---

**R2: API recheck_all**

```python
# api/Watch.py:543-554
queued_uuids = set(self.update_q.get_queued_uuids())
running_uuids = set(worker_pool.get_running_uuids())

watches_to_queue_filtered = [
    uuid for uuid in watches_to_queue
    if uuid not in queued_uuids and uuid not in running_uuids
]

for uuid in watches_to_queue_filtered:
    worker_pool.queue_item_async_safe(self.update_q, 
        queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
```
- ✓ 检查 running_uuids
- ✓ 检查 queued_uuids
- ✗ 不检查 paused

---

**R3: UI /checknow (单watch)**

```python
# blueprint/ui/__init__.py:273-277
if worker_pool.is_watch_running(uuid) or uuid in update_q.get_queued_uuids():
    flash(gettext("Watch is already queued or being checked."))
else:
    worker_pool.queue_item_async_safe(update_q, 
        queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
    flash(gettext("Queued 1 watch for rechecking."))
```
- ✓ 检查 running_uuids (via `is_watch_running`)
- ✓ 检查 queued_uuids
- ✗ 不检查 paused

---

**R4: UI /checknow (批量)**

```python
# blueprint/ui/__init__.py:281-305
for k in sorted(datastore.data['watching'].items(), ...):
    watch_uuid = k[0]
    watch = k[1]
    if not watch['paused'] and watch_uuid:  # ✓ paused检查
        if with_errors and not watch.get('last_error'):
            continue
        if tag != None and tag not in watch['tags']:
            continue
        watches_to_queue.append(watch_uuid)

# 过滤已运行/已队列的
queued_uuids = set(update_q.get_queued_uuids())
running_uuids = set(worker_pool.get_running_uuids())
for watch_uuid in watches_to_queue:
    if watch_uuid not in queued_uuids and watch_uuid not in running_uuids:
        watches_to_queue_filtered.append(watch_uuid)
```
- ✓ 检查 running_uuids
- ✓ 检查 queued_uuids
- ✓ 检查 paused

---

**R5: API Tag级recheck**

```python
# api/Tags.py:29-43
if request.args.get('recheck'):
    watches_to_queue = []
    for k in sorted(self.datastore.data['watching'].items(), ...):
        watch_uuid = k[0]
        watch = k[1]
        if not watch['paused'] and tag['uuid'] in watch['tags']:  # ✓ paused检查
            watches_to_queue.append(watch_uuid)

    for watch_uuid in watches_to_queue:
        worker_pool.queue_item_async_safe(self.update_q, 
            queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```
- ✗ 不检查 running_uuids
- ✗ 不检查 queued_uuids
- ✓ 检查 paused

---

**R6: Socket.IO recheck**

```python
# realtime/events.py:38-45
elif op == 'recheck':
    worker_pool.queue_item_async_safe(update_q, 
        queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
```
- 无任何检查，直接入队

---

**S1: 定时调度**

```python
# flask_app.py:1185-1264
if watch['paused']:  # ✓ paused检查
    continue

if seconds_since_last_recheck >= (threshold + watch.jitter_seconds) \
   and seconds_since_last_recheck >= recheck_time_minimum_seconds:
    if not uuid in running_uuids and uuid not in queued_uuids:  # ✓ running/queued检查
        # 入队
```
- ✓ 检查 running_uuids
- ✓ 检查 queued_uuids
- ✓ 检查 paused
- ✓ 检查时间间隔
- ✓ 检查时间窗口

---

## 三、Worker Claim失败后的延迟优先级公式

### 3.1 公式定义

**代码位置**: `worker.py:70-77`

```python
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)
    deferred_priority = max(1000, queued_item_data.priority * 10)
    deferred_item = PrioritizedItem(priority=deferred_priority, item=queued_item_data.item)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

**公式**: 
```
deferred_priority = max(1000, original_priority × 10)
```

### 3.2 priority=1 的实际结果

| 原始优先级 | 延迟优先级计算 | 最终延迟优先级 |
|------------|----------------|----------------|
| 1 | max(1000, 1 × 10) = max(1000, 10) | **1000** |
| 1000 | max(1000, 1000 × 10) = max(1000, 10000) | 10000 |
| timestamp (约1.7×10^9) | max(1000, 1.7×10^10) | 1.7×10^10 |

**结论**: 当 `priority=1` 时，延迟优先级为 **1000**

### 3.3 延迟机制流程

```
Worker获取任务
       │
       ↓
claim_uuid_for_processing()
       │
  ┌────┴────┐
  ↓         ↓
成功      失败
  │         │
  ↓         ↓
执行     await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)
抓取           │
               ↓
          deferred_priority = max(1000, priority × 10)
               │
               ↓
          重新入队
```

---

## 四、与定时调度路径对照

### 4.1 完整对照表

| 检查项 | R1 | R2 | R3 | R4 | R5 | R6 | S1 (定时) |
|--------|----|----|----|----|----|----|-----------|
| **优先级** | 1 | 1 | 1 | 1 | 1 | 1 | timestamp |
| **running_uuids** | ✗ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ |
| **queued_uuids** | ✗ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ |
| **paused** | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ | ✓ |
| **时间间隔** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **最小间隔** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **时间窗口** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **代理复用** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Worker claim去重** | ✓(间接) | ✓(间接) | ✓(间接) | ✓(间接) | ✓(间接) | ✓(间接) | ✓(间接) |

### 4.2 检查策略分类

| 策略类型 | 通路 | 特点 |
|----------|------|------|
| **无检查** | R1, R6 | 无条件直接入队 |
| **部分检查** | R2, R3 | 检查运行/队列状态，不检查暂停 |
| **完整检查** | R4, S1 | 检查运行/队列/暂停状态 |
| **有限检查** | R5 | 仅检查暂停状态 |

---

## 五、关键发现

### 5.1 检查完整性评分

| 通路 | 完整性评分 | 风险评估 |
|------|-----------|----------|
| R1: API单watch | 0/3 | **高风险** - 可能重复入队、暂停watch也会被触发 |
| R2: API recheck_all | 2/3 | 中风险 - 不检查暂停状态 |
| R3: UI单watch | 2/3 | 中风险 - 不检查暂停状态 |
| R4: UI批量 | 3/3 | 低风险 - 完整检查 |
| R5: API Tag级 | 1/3 | 中风险 - 不检查运行/队列状态 |
| R6: Socket.IO | 0/3 | **高风险** - 可能重复入队、暂停watch也会被触发 |
| S1: 定时调度 | 3/3 + 额外检查 | 低风险 - 最完整的检查链 |

### 5.2 一致性问题

| 问题 | 影响通路 | 描述 |
|------|----------|------|
| **暂停状态检查不一致** | R1, R2, R3, R6 | 暂停的watch仍可被入队 |
| **去重检查不一致** | R1, R5, R6 | 同一watch可能被多次入队 |
| **优先级统一** | 所有recheck路径 | 均使用priority=1，可能导致队列阻塞 |

### 5.3 Worker层最终防护

所有通路最终都会经过Worker层的claim机制进行去重：

```python
# worker.py:70-77
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    deferred_priority = max(1000, queued_item_data.priority * 10)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

---

## 六、总结

### 6.1 真值表速览

```
                        running_uuids | queued_uuids | paused
───────────────────────────────────────────────────────────────
R1: API单watch            ✗              ✗            ✗
R2: API recheck_all       ✓              ✓            ✗
R3: UI单watch             ✓              ✓            ✗
R4: UI批量                ✓              ✓            ✓
R5: API Tag级             ✗              ✗            ✓
R6: Socket.IO             ✗              ✗            ✗
S1: 定时调度              ✓              ✓            ✓
```

### 6.2 延迟优先级计算

- **公式**: `deferred_priority = max(1000, priority × 10)`
- **priority=1 的结果**: **1000**
- **作用**: 避免重复处理，延迟后重新入队

### 6.3 改进建议

1. **统一检查标准**: 为所有recheck路径添加基础检查（暂停状态、运行/队列状态）
2. **优先级差异化**: 根据触发来源设置不同优先级
3. **幂等性保证**: 添加请求去重机制
