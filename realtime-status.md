# 监控前端实时通知状态流转分析报告

## 1. 系统架构概述

changedetection.io 使用 **Blinker 信号系统 + Socket.IO 实时通信** 的架构来实现多端状态同步。整个流转链路可以分为五个核心层次：

```
┌─────────────────────────────────────────────────────────────────┐
│                     前端浏览器 (多标签页)                       │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     Socket.IO 客户端    │
│  │ 标签页1 │  │ 标签页2 │  │ 标签页N │       (realtime.js)      │
│  └────┬────┘  └────┬────┘  └────┬────┘                          │
└───────┼──────────────┼─────────────┼──────────────────────────────┘
        │              │             │
        └──────────────┼─────────────┘
                       │ WebSocket/HTTP 长连接
┌──────────────────────┼────────────────────────────────────────────┐
│              Socket.IO 服务端 (socket_server.py)                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │              SignalHandler (信号接收器)                   │    │
│  │  watch_check_update, queue_length, notification_event    │    │
│  └───────────────────────────┬──────────────────────────────┘    │
└──────────────────────────────┼───────────────────────────────────┘
                               │ Blinker 信号
┌──────────────────────────────┼───────────────────────────────────┐
│              业务逻辑层 (worker.py, store/)                      │
│  ┌─────────────┐  ┌──────────┐  ┌──────────────────────────┐    │
│  │  抓取 Worker│  │ Diff 检测│  │  状态持久化 (Watch.commit)│    │
│  └──────┬──────┘  └────┬─────┘  └───────────┬──────────────┘    │
└─────────┼───────────────┼────────────────────┼───────────────────┘
          │               │                    │
┌─────────▼───────────────▼────────────────────▼───────────────────┐
│              数据存储层 (文件系统 + 内存缓存)                      │
│  watch.json, history 快照, notification 队列                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 抓取与 Diff 事件产生机制

### 2.1 任务队列架构

**核心组件**：`RecheckPriorityQueue` (`queue_handlers.py:15-411`)

- 基于 `heapq` 的优先级队列，支持多线程/多协程安全访问
- 使用 `threading.RLock` 保证原子操作
- 优先级规则：
  - `priority=1`：立即执行（手动重新检查）
  - `priority=5`：克隆任务
  - `priority>100`：定时调度任务

**入队触发点**：
1. 定时调度器 (`ticker_thread`)：按配置的 `time_between_check` 周期性入队
2. 手动操作：点击"Recheck"按钮或 API 调用
3. 新增 Watch：自动立即入队检查一次

### 2.2 异步 Worker 处理流程

**Worker 池管理**：`worker_pool.py`

- 每个 Worker 运行在独立线程中，拥有独立的 `asyncio` 事件循环
- 默认启动 10 个 Worker（可通过 `FETCH_WORKERS` 环境变量配置）
- 支持动态扩缩容和崩溃自动重启

**抓取与 Diff 流程** (`worker.py:46-743`)：

```python
# 核心处理流程摘要
async def async_update_worker(worker_id, q, notification_q, app, datastore, executor):
    while not app.config.exit.is_set():
        # 1. 从队列获取任务 (超时等待 1s)
        queued_item_data = await q.async_get(executor=executor, timeout=1.0)
        
        # 2. 声明 UUID 所有权，防止重复处理
        if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
            # 已被其他 Worker 处理，延迟重新入队
            continue
        
        # 3. 发送开始检查信号
        watch_check_update.send(watch_uuid=uuid)
        
        # 4. 执行页面抓取 (异步)
        await update_handler.call_browser()
        
        # 5. 执行 Diff 检测 (线程池中运行，避免阻塞事件循环)
        changed_detected, update_obj, contents = await loop.run_in_executor(
            executor,
            lambda: update_handler.run_changedetection(watch=watch)
        )
        
        # 6. 保存历史快照
        if changed_detected:
            watch.save_history_blob(contents=contents, timestamp=int(fetch_start_time))
            
            # 7. 发送通知
            if watch.history_n >= 2 and not watch.get('notification_muted'):
                await send_content_changed_notification(uuid, notification_q, datastore)
        
        # 8. 更新 Watch 状态并持久化
        datastore.update_watch(uuid=uuid, update_obj=final_updates)
        
        # 9. 释放 UUID 并发送完成信号
        worker_pool.release_uuid_from_processing(uuid, worker_id=worker_id)
        watch_check_update.send(watch_uuid=watch['uuid'])
```

**关键信号触发点**：

| 阶段 | 信号 | 发送位置 |
|------|------|----------|
| 开始检查 | `watch_check_update` | `worker.py:163-164` |
| 状态更新 | `watch_small_status_comment` | `worker.py:43` |
| 队列变化 | `queue_length` | `queue_handlers.py:398,408` |
| 通知事件 | `notification_event` | `queue_handlers.py:546` |
| Favicon 更新 | `watch_favicon_bump` | `Watch.py:bump_favicon()` |
| Watch 删除 | `watch_deleted` | `store/__init__.py:623-625` |

---

## 3. 事件推送到前端界面

### 3.1 信号到 Socket.IO 的桥梁

**SignalHandler** (`socket_server.py:14-136`) 是连接后端信号和前端实时更新的核心桥梁：

```python
class SignalHandler:
    def __init__(self, socketio_instance, datastore):
        # 注册所有需要监听的信号
        watch_check_update.connect(self.handle_signal, weak=False)
        queue_length_signal.connect(self.handle_queue_length, weak=False)
        watch_delete_signal.connect(self.handle_deleted_signal, weak=False)
        watch_favicon_bumped_signal.connect(self.handle_watch_bumped_favicon_signal, weak=False)
        watch_small_status_comment_signal.connect(self.handle_watch_small_status_update, weak=False)
        notification_event_signal.connect(self.handle_notification_event, weak=False)
```

### 3.2 事件类型与数据结构

| 事件名称 | 触发时机 | 数据结构 |
|----------|----------|----------|
| `watch_update` | Watch 状态变化时 | `{watch: {uuid, checking_now, queued, unviewed, has_error, paused, last_changed_text, ...}}` |
| `general_stats_update` | 全局统计变化时 | `{count_errors, unread_changes_count}` |
| `queue_size` | 队列长度变化时 | `{q_length, event_timestamp}` |
| `watch_small_status_comment` | 临时状态更新 | `{uuid, status, event_timestamp}` |
| `watch_bumped_favicon` | Favicon 更新时 | `{uuid, event_timestamp}` |
| `watch_deleted` | Watch 删除时 | `{uuid, event_timestamp}` |
| `notification_event` | 通知发送时 | `{watch_uuid, event_timestamp}` |

### 3.3 前端事件处理 (`realtime.js`)

```javascript
// 连接 Socket.IO
const socket = io({
    path: socketio_url,
    transports: ['websocket', 'polling'],
    reconnectionDelay: 3000,
    reconnectionAttempts: 25
});

// 核心事件处理器
socket.on('watch_update', function (data) {
    const watch = data.watch;
    const $watchRow = $('tr[data-watch-uuid="' + watch.uuid + '"]');
    
    // 更新行 CSS 类
    $watchRow.toggleClass('checking-now', watch.checking_now);
    $watchRow.toggleClass('queued', watch.queued);
    $watchRow.toggleClass('unviewed', watch.unviewed);
    $watchRow.toggleClass('has-error', watch.has_error);
    $watchRow.toggleClass('paused', watch.paused);
    
    // 更新文本内容
    $('td.last-changed', $watchRow).text(watch.last_changed_text);
    $('td.last-checked .innertext', $watchRow).text(watch.last_checked_text);
});

// 全局统计更新
socket.on('general_stats_update', function (general_stats) {
    $('#unread-tab-counter').text(general_stats.unread_changes_count);
    $('#post-list-with-errors a').text(`With errors (${general_stats.count_errors})`);
});
```

---

## 4. 状态计数器更新机制

### 4.1 未读变更计数 (`unread_changes_count`)

**计算逻辑** (`store/__init__.py:573-579`)：
```python
@property
def unread_changes_count(self):
    unread_changes_count = 0
    for uuid, watch in self.__data['watching'].items():
        # 有2条以上历史记录 且 未被查看
        if watch.history_n >= 2 and watch.viewed == False:
            unread_changes_count += 1
    return unread_changes_count
```

**Viewed 属性判断** (`Watch.py:256-265`)：
```python
@property
def viewed(self):
    # last_viewed 时间戳 >= 最新历史记录时间戳 即为已查看
    if int(self['last_viewed']) and int(self['last_viewed']) >= int(self.newest_history_key):
        return True
    return False

@property
def has_unviewed(self):
    # 最新历史记录时间戳 > last_viewed 且 有2条以上历史记录
    return int(self.newest_history_key) > int(self['last_viewed']) and self.__history_n >= 2
```

### 4.2 错误计数 (`count_errors`)

**计算逻辑** (`socket_server.py:178-181`)：
```python
errored_count = 0
for watch_uuid_iter, watch_iter in datastore.data['watching'].items():
    if watch_iter.get('last_error'):
        errored_count += 1
```

### 4.3 标记为已查看

**触发场景**：
1. 点击"Mark viewed"按钮（单个或批量）
2. 点击"Mark all viewed"按钮
3. 访问 Diff 页面或历史记录页面

**实现** (`store/__init__.py:458-465`)：
```python
def set_last_viewed(self, uuid, timestamp):
    self.data['watching'][uuid].update({'last_viewed': int(timestamp)})
    self.data['watching'][uuid].commit()
    # 触发信号更新前端
    watch_check_update = signal('watch_check_update')
    watch_check_update.send(watch_uuid=uuid)
```

---

## 5. 多浏览器标签状态同步机制

### 5.1 同步原理

changedetection.io **不使用** `localStorage` 或 `BroadcastChannel` 进行标签页间同步，而是采用 **"服务端广播 + 多连接独立接收"** 的架构：

```
┌─────────────┐
│  浏览器标签1 │───┐
└─────────────┘   │  独立 Socket.IO 连接
                  ▼
┌─────────────┐  Socket.IO Server   ┌─────────────┐
│  浏览器标签2 │───►  广播给所有连接  ◄───  浏览器标签3 │
└─────────────┘                     └─────────────┘
```

### 5.2 关键特性

1. **每个标签页独立连接**：每个浏览器标签页都会建立独立的 Socket.IO 连接
2. **服务端全量广播**：`socketio.emit()` 默认向所有已连接的客户端发送事件
3. **连接认证**：有密码保护时，只有已登录用户才能建立连接
4. **重连机制**：连接断开时自动重试（最多 25 次，间隔 3 秒）

### 5.3 初始化同步

新连接建立时，服务端会发送初始队列状态：
```python
@socketio.on('connect')
def handle_connect():
    # 发送当前队列大小给新连接的客户端
    queue_size = update_q.qsize()
    socketio.emit("queue_size", {
        "q_length": queue_size,
        "event_timestamp": time.time()
    }, room=request.sid)  # 只发送给这个客户端
```

### 5.4 操作闭环链路（Pause/Mute/Recheck/批量操作）

从任一标签发起操作到所有标签同步更新的完整链路：

#### 5.4.1 单个 Watch 操作（Pause/Mute/Recheck）

**链路追踪**：

```
┌─────────────────────────────────────────────────────────────────┐
│  标签页A（操作发起端）                                          │
│  1. 用户点击 pause/mute/recheck 按钮 (.ajax-op)                  │
│  2. realtime.js:12-27 → socket.emit('watch_operation', data)    │
│  3. 本地无立即更新，等待服务端回推                                │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼  WebSocket
┌─────────────────────────────────────────────────────────────────┐
│  Socket.IO 服务端 (realtime/events.py:9-60)                     │
│  4. handle_watch_operation() 接收事件                            │
│  5. 根据 op 执行不同操作：                                       │
│     - pause/mute: watch.toggle_pause()/toggle_mute() → commit() │
│     - recheck: worker_pool.queue_item_async_safe() 入队          │
│  6. watch_check_update.send(watch_uuid=uuid) 发送信号             │
│  7. emit('operation_result') 回执给发起端                         │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼  Blinker 信号
┌─────────────────────────────────────────────────────────────────┐
│  SignalHandler (socket_server.py:62-79)                         │
│  8. handle_signal() 接收 watch_check_update 信号                 │
│  9. 组装 watch_data（包含 paused/notification_muted/queued 等）  │
│ 10. 计算 general_stats（count_errors, unread_changes_count）     │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼  Socket.IO 广播
┌─────────────────────────────────────────────────────────────────┐
│  所有连接的浏览器标签页（A/B/C...）                              │
│ 11. realtime.js:216-243 → socket.on('watch_update')              │
│ 12. 更新对应 tr 的 CSS 类（paused/notification_muted/queued）    │
│ 13. realtime.js:208-214 → socket.on('general_stats_update')      │
│ 14. 更新未读计数和错误计数                                       │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码路径**：

| 步骤 | 文件 | 位置 | 说明 |
|------|------|------|------|
| 2 | `realtime.js` | L12-27 | 发送 `watch_operation` 事件 |
| 4-6 | `events.py` | L9-60 | 处理操作、修改状态、发送信号 |
| 5-pause | `Watch.py` | L1052-1053 | `toggle_pause()` 切换暂停状态 |
| 5-mute | `Watch.py` | L1061-1062 | `toggle_mute()` 切换静音状态 |
| 5-recheck | `queue_handlers.py` | L65-95 | `RecheckPriorityQueue.put()` 入队并发送 `queue_length` 信号 |
| 8-10 | `socket_server.py` | L62-79, L139-196 | 接收信号、组装数据、广播给所有客户端 |

#### 5.4.2 批量 Checkbox 操作

**链路追踪**：

```
┌─────────────────────────────────────────────────────────────────┐
│  标签页A（操作发起端）                                          │
│  1. 勾选多个 Watch，点击批量操作按钮                             │
│  2. realtime.js:30-72 → socket.emit('checkbox-operation', data)  │
│  3. 数据包含 op, uuids[], extra_data                             │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼  WebSocket
┌─────────────────────────────────────────────────────────────────┐
│  Socket.IO 服务端 (socket_server.py:274-305)                    │
│  4. event_checkbox_operations() 接收事件                        │
│  5. 启动后台线程执行：                                          │
│     thread = threading.Thread(target=run_operation)              │
│  6. 后台线程调用 _handle_operations()                            │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼  后台线程
┌─────────────────────────────────────────────────────────────────┐
│  _handle_operations() (blueprint/ui/__init__.py:13-117)         │
│  7. 根据 op 批量执行操作：                                       │
│     - pause/unpause: 设置 watch['paused'] → commit()            │
│     - mute/unmute: 设置 watch['notification_muted'] → commit()  │
│     - recheck: 批量入队 update_q                                 │
│     - mark-viewed: datastore.set_last_viewed()                   │
│     - clear-errors: 设置 last_error=False → commit()             │
│     - clear-history: datastore.clear_watch_history()             │
│     - delete: datastore.delete(uuid)                             │
│  8. 循环所有 uuids: watch_check_update.send(watch_uuid=uuid)      │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼  Blinker 信号（多次，每个UUID一次）
┌─────────────────────────────────────────────────────────────────┐
│  SignalHandler → 广播 watch_update + general_stats_update       │
│  所有标签页同步更新所有涉及的 Watch 状态                          │
└─────────────────────────────────────────────────────────────────┘
```

**设计特点**：
- **异步执行**：批量操作在后台线程中执行，不阻塞 Socket.IO 事件循环
- **逐个信号**：每个 UUID 单独发送 `watch_check_update` 信号，确保每个标签页逐个更新
- **操作原子性**：每个 Watch 的修改和 commit 是原子的，中间状态不会被广播

#### 5.4.3 与未读计数、队列状态、历史记录页的联动

**1. 未读计数联动 (`unread_changes_count`)**

触发操作：`mark-viewed`（单个/批量/全部）
- 操作执行：`datastore.set_last_viewed(uuid, timestamp)`
- 状态变更：`watch['last_viewed']` 更新为当前时间戳
- 信号触发：`watch_check_update` 信号发送
- 服务端计算：`handle_watch_update()` 中重新计算 `datastore.unread_changes_count`
- 前端更新：`general_stats_update` 事件 → `$('#unread-tab-counter').text(...)`

**2. 队列状态联动 (`queue_size`)**

触发操作：`recheck`（单个/批量）
- 操作执行：`worker_pool.queue_item_async_safe(update_q, PrioritizedItem(...))`
- 队列变更：`RecheckPriorityQueue.put()` → `_emit_put_signals()`
- 信号触发：`queue_length` 信号发送
- 服务端广播：`SignalHandler.handle_queue_length()` → `socketio.emit("queue_size", ...)`
- 前端更新：`realtime.js:127-167` → 更新队列数字、切换 `has-queue` CSS 类

**3. 历史记录页标记联动**

触发操作：`clear-history`、访问 Diff 页面
- `clear-history` 操作：`datastore.clear_watch_history(uuid)`
  - 清空历史快照 → `history_n` 重置为 0
  - `viewed` 状态变为 `True`（因为 `history_n < 2`）
  - `has_unviewed` 变为 `False`
  - 触发信号 → 所有标签页移除 `unviewed` 类

- 访问 Diff 页面：
  - 前端点击：`watch-overview.js:20-22` → 临时移除 `unviewed` 类（即时反馈）
  - 后端处理：`diff.py:285,320` → `datastore.set_last_viewed(uuid, timestamp)`
  - 持久化变更：`watch.commit()` 写入磁盘
  - 信号触发：`watch_check_update` → 所有标签页同步状态

---

## 6. 与后台队列、历史记录页的关系

### 6.1 后台队列状态同步

**队列长度实时更新**：
- 入队时：`RecheckPriorityQueue._emit_put_signals()` 发送 `queue_length` 信号
- 出队时：`RecheckPriorityQueue._emit_get_signals()` 发送 `queue_length` 信号
- 前端显示在 "QUEUE" 标签和页面顶部指示器

**队列中的 Watch 状态**：
- 前端通过 `watch.queued` 标记显示 "Queued" 状态
- 数据来源：`update_q.get_queued_uuids()` 获取当前队列中的所有 UUID

### 6.2 历史记录页交互

**查看 Diff 时的状态变化**：

1. 点击 Diff 链接 → 前端移除 `unviewed` CSS 类（即时视觉反馈）
2. 访问 `/diff/<uuid>` 页面 → 后端调用 `datastore.set_last_viewed(uuid, timestamp)`
3. 触发 `watch_check_update` 信号 → 所有标签页同步更新 `unviewed` 状态
4. 全局 `unread_changes_count` 重新计算并推送

**历史记录与状态的关系**：
- `history_n >= 2` 是判断是否有变更的前提条件
- `last_viewed` 与历史记录时间戳比较决定 `viewed` 状态
- `get_from_version_based_on_last_viewed()` 根据查看进度智能选择 Diff 对比版本

### 6.3 通知队列与实时事件

**双队列架构**：
1. **`update_q`** (RecheckPriorityQueue)：存放待抓取的 Watch 任务
2. **`notification_q`** (NotificationQueue)：存放待发送的通知

**通知触发实时事件**：
```python
# 通知入队时触发 notification_event 信号
self.notification_event_signal.send(watch_uuid=watch_uuid)

# SignalHandler 接收后广播给前端
socketio.emit("notification_event", {
    "watch_uuid": watch_uuid,
    "event_timestamp": time.time()
})
```

---

## 7. 关键时序图

### 7.1 一次完整的抓取与状态更新流程

```
用户操作/定时器
    │
    ▼
┌─────────────────┐
│ 任务入队 update_q │
│ 发送 queue_length │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│ Worker 领取任务  │
│ claim_uuid()     │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐        ┌─────────────────┐
│ watch_check_update │──────►│ 前端显示 "Checking" │
│ 信号             │        │ 状态             │
└─────────┬───────┘        └─────────────────┘
          │
          ▼
┌─────────────────┐
│ 抓取页面 + Diff  │
│ 检测变更         │
└─────────┬───────┘
          │
          ├──────────────────────────────────┐
          │ 有变更                           │ 无变更
          ▼                                  ▼
┌─────────────────┐                  ┌─────────────────┐
│ 保存历史快照     │                  │ 更新 last_error  │
│ notification_q  │                  │ 状态             │
│ 入队             │                  └─────────┬───────┘
└─────────┬───────┘                            │
          │                                    │
          ▼                                    │
┌─────────────────┐                            │
│ watch_check_update │◄──────────────────────────┘
│ 信号 (完成)       │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐        ┌─────────────────┐
│ handle_signal()  │───────►│ 前端更新状态     │
│ 组装 watch_data  │        │ general_stats   │
│ general_stats    │        │ 更新             │
└─────────────────┘        └─────────────────┘
```

---

## 8. 核心文件索引

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| 实时通信服务端 | `changedetectionio/realtime/socket_server.py` | Socket.IO 初始化、信号处理、事件广播 |
| 实时通信事件 | `changedetectionio/realtime/events.py` | 处理前端发起的操作（暂停、重新检查等） |
| 前端实时逻辑 | `changedetectionio/static/js/realtime.js` | Socket.IO 客户端、事件处理、DOM 更新 |
| 抓取 Worker | `changedetectionio/worker.py` | 异步抓取、Diff 检测、信号触发 |
| Worker 池管理 | `changedetectionio/worker_pool.py` | Worker 生命周期、UUID 所有权管理 |
| 队列实现 | `changedetectionio/queue_handlers.py` | 优先级队列、通知队列、信号发射 |
| Watch 模型 | `changedetectionio/model/Watch.py` | Watch 领域模型、viewed 状态计算 |
| 数据存储 | `changedetectionio/store/__init__.py` | Watch CRUD、set_last_viewed、持久化 |
| 通知服务 | `changedetectionio/notification_service.py` | 通知内容组装、队列发送 |

---

## 9. 设计要点总结

1. **解耦设计**：通过 Blinker 信号系统解耦业务逻辑和实时通信层，新增事件只需发送信号即可
2. **最终一致性**：不保证强一致性，通过信号重发和页面刷新保证最终状态一致
3. **广播模式**：服务端向所有连接客户端广播，前端根据 UUID 选择性更新，简化架构
4. **优雅降级**：Socket.IO 连接失败时，页面仍可正常工作，只是失去实时更新能力
5. **内存优化**：Diff 检测和通知渲染在 ThreadPoolExecutor 中执行，避免阻塞 asyncio 事件循环
