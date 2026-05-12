# ChangeDetection.io 通知失败与重试机制分析

> 本文档通过代码阅读分析，重点回答三个问题：
> 1. 通知队列被消费后到实际发送的调用链路是什么？
> 2. 某个通道发送报错时，错误具体记录到哪里、界面是怎么感知到的？
> 3. 系统会不会对同一条失败通知自动重试？如果不会，它和"下一次定时检查再次触发通知"有什么区别？

---

## 目录
1. [通知队列消费到发送的完整调用链路](#1-通知队列消费到发送的完整调用链路)
2. [发送失败时的错误记录与界面感知](#2-发送失败时的错误记录与界面感知)
3. [重试机制辨析：自动重试 vs 定时检查触发](#3-重试机制辨析自动重试-vs-定时检查触发)
4. [精简时序图](#4-精简时序图)
5. [关键代码位置索引](#5-关键代码位置索引)

---

## 1. 通知队列消费到发送的完整调用链路

### 1.1 整体架构概览

通知系统采用**生产-消费模型**，由两个独立的线程池处理：

| 池 | 线程类型 | 数量配置 | 职责 |
|---|---------|---------|------|
| **检查工作器池** | `PageFetchAsyncUpdateWorker-{N}` | `FETCH_WORKERS` (默认 10) | 抓取页面、检测变化、产生通知 |
| **通知工作器池** | `NotificationRunner-{N}` | `NOTIFICATION_WORKERS` (默认 1) | 消费通知队列、实际发送通知 |

---

### 1.2 通知产生链路（生产者）

当某个 watch 检测到变化后，通知被放入通知队列：

**代码路径**: `notification_service.py:384`

```python
# NotificationService 类中的方法
def _do_notification(...):
    # ... 构建通知上下文 NotificationContextData ...
    if self.notification_q:
        logger.debug("Queued notification for sending")
        self.notification_q.put(n_object)  # ← 放入队列
```

**调用入口**:
- `send_content_changed_notification()` - 内容变化时调用
- `send_filter_failure_notification()` - 过滤器失败时调用
- `send_step_failure_notification()` - 浏览器步骤失败时调用

---

### 1.3 通知消费链路（消费者）

#### 1.3.1 通知工作器线程启动

**代码路径**: `flask_app.py:1001-1010`

```python
# 应用启动时
notification_workers = int(os.getenv("NOTIFICATION_WORKERS", "1"))
for i in range(notification_workers):
    threading.Thread(
        target=notification_runner,  # ← 消费者入口函数
        args=(i,),
        daemon=True,
        name=f"NotificationRunner-{i}"
    ).start()
```

#### 1.3.2 消费者主循环

**代码路径**: `flask_app.py:1053-1102`

```python
def notification_runner(worker_id=0):
    global notification_debug_log
    
    with app.app_context():
        while not app.config.exit.is_set():
            try:
                # 1. 从队列取出通知（非阻塞）
                n_object = notification_q.get(block=False)
            except queue.Empty:
                # 队列为空，等待 1 秒后重试
                app.config.exit.wait(1)
            
            else:
                # 2. 有通知，准备发送
                sent_obj = None
                
                try:
                    # 3. 回退到全局通知配置
                    if not n_object.get('notification_body') and ...:
                        n_object['notification_body'] = datastore.data['settings']['application'].get('notification_body')
                    
                    # 4. 实际发送
                    if n_object.get('notification_urls', {}):
                        sent_obj = process_notification(n_object, datastore)  # ← 核心调用
                    
                except Exception as e:
                    # 5. 发送失败处理（详见第 2 节）
                    logger.error(f"Notification worker {worker_id} - Watch URL: {n_object['watch_url']} Error {str(e)}")
                    if 'uuid' in n_object:
                        datastore.update_watch(
                            uuid=n_object['uuid'],
                            update_obj={'last_notification_error': "Notification error detected, goto notification log."}
                        )
                    log_lines = str(e).splitlines()
                    notification_debug_log += log_lines
                
                # 6. 记录发送日志（无论成功失败）
                notification_debug_log += ["{} - SENDING - {}".format(now.strftime("%c"), json.dumps(sent_obj))]
                notification_debug_log = notification_debug_log[-100:]  # 保留最近 100 条
```

---

### 1.4 实际发送链路 (`process_notification`)

**代码路径**: `notification/handler.py:307-496`

```python
def process_notification(n_object: NotificationContextData, datastore):
    
    # 步骤 1: 创建通知参数（模板变量）
    notification_parameters = create_notification_parameters(n_object, datastore)
    
    # 步骤 2: 按需渲染差异变量
    n_object.update(add_rendered_diff_to_notification_vars(
        notification_scan_text=n_object.get('notification_body', '')+n_object.get('notification_title', ''),
        current_snapshot=n_object.get('current_snapshot'),
        prev_snapshot=n_object.get('prev_snapshot'),
        word_diff=...
    ))
    
    # 步骤 3: 初始化 Apprise
    apobj = apprise.Apprise(debug=True, asset=apprise_asset)
    apprise.plugins.N_MGR.remove('discord')  # 替换为自定义 Discord 插件
    apprise.plugins.N_MGR.add(NotifyDiscordCustom, schemas='discord')
    
    # 步骤 4: 遍历所有通知 URL（通道）
    with apprise.LogCapture(level=apprise.logging.DEBUG) as logs:
        for url in n_object['notification_urls']:
            
            # 4a: Jinja2 渲染模板
            n_body = jinja_render(template_str=n_object.get('notification_body', ''), **notification_parameters)
            n_title = jinja_render(template_str=n_object.get('notification_title', ''), **notification_parameters)
            
            # 4b: 服务特定调整（Discord/Telegram/Email 等格式转换）
            (url, n_body, n_title) = apply_service_tweaks(
                url=url, 
                n_body=n_body, 
                n_title=n_title, 
                requested_output_format=requested_output_format_original
            )
            
            # 4c: 添加到 Apprise
            if not url.startswith('null://'):
                apobj.add(url)
            
            # 4d: 邮件特殊处理（等宽字体包装）
            if url.startswith('mail') and 'html' in requested_output_format:
                n_body = as_monospaced_html_email(content=n_body, title=n_title)
        
        # 步骤 5: 调用 Apprise 发送（注意：在 for 循环外面，一次 notify 发送所有通道）
        if not url.startswith('null://'):
            apobj.notify(
                title=n_title,
                body=n_body,
                body_format=apprise_input_format,
                attach=n_object.get('screenshot', None)
            )
        
        # 步骤 6: 检查 Apprise 日志中的错误
        log_value = logs.getvalue()
        if log_value and ('WARNING' in log_value or 'ERROR' in log_value):
            logger.critical(log_value)
            raise Exception(log_value)  # ← 抛出异常，通知失败
    
    return sent_objs
```

**关键点**: 
- `apobj.notify()` 在 `for url in ...` 循环**外面**，意味着一次调用发送所有通道
- 任何一个通道报错，整个 `process_notification` 抛出异常

---

## 2. 发送失败时的错误记录与界面感知

### 2.1 错误发生时机

当 `process_notification()` 抛出异常时，异常被 `notification_runner` 中的 `except` 块捕获：

**代码路径**: `flask_app.py:1085-1097`

```python
try:
    sent_obj = process_notification(n_object, datastore)
except Exception as e:
    # 异常捕获点
    logger.error(f"Notification worker {worker_id} - Watch URL: {n_object['watch_url']}  Error {str(e)}")
    
    # 1. 写入 watch 的 last_notification_error 字段
    if 'uuid' in n_object:
        datastore.update_watch(
            uuid=n_object['uuid'],
            update_obj={'last_notification_error': "Notification error detected, goto notification log."}
        )
    
    # 2. 写入全局通知调试日志
    log_lines = str(e).splitlines()
    notification_debug_log += log_lines
    
    # 3. 发送信号通知 UI 更新
    with app.app_context():
        app.config['watch_check_update_SIGNAL'].send(app_context=app, watch_uuid=n_object.get('uuid'))
```

---

### 2.2 错误记录的两个位置

#### 位置 1: Watch 级别的 `last_notification_error`

**存储**: `datastore.data['watching'][uuid]['last_notification_error']`

**内容**: 固定字符串 `"Notification error detected, goto notification log."`

**持久化**: 通过 `datastore.update_watch()` 写入，最终保存到 JSON 文件

**重置时机**: `Watch.clear_watch()` 时重置为 `False` (代码路径: `model/Watch.py:340`)

---

#### 位置 2: 全局 `notification_debug_log` 列表

**存储**: 全局变量 `notification_debug_log` (定义在 `flask_app.py:160`)

**内容**: 包含两类记录：
1. 成功发送记录: `"{timestamp} - SENDING - {JSON}"`
2. 错误详情: 异常 `str(e).splitlines()` 逐行追加

**大小限制**: 只保留最近 100 条 (`notification_debug_log = notification_debug_log[-100:]`)

**持久化**: **内存中的列表**，不写入磁盘，应用重启后丢失

---

### 2.3 界面感知机制

#### 2.3.1 实时推送路径

```
异常发生
    ↓
watch_check_update_SIGNAL.send()
    ↓
SignalHandler.handle_signal() [socket_server.py:62]
    ↓
handle_watch_update() [socket_server.py:139]
    ↓
socketio.emit("watch_update", {'watch': watch_data})
    ↓
前端收到 watch_update 事件
    ↓
UI 更新 watch 卡片（显示错误状态）
```

#### 2.3.2 错误文本编译

当 Socket.IO 推送 watch 更新时，会调用 `compile_error_texts()` 编译错误信息：

**代码路径**: `model/Watch.py:1264-1300`

```python
def compile_error_texts(self, has_proxies=None):
    output = []
    
    # ... 处理 last_error ...
    
    # 处理通知错误
    if self.get('last_notification_error'):
        txt = safe_jinja.render_fully_escaped(self.get('last_notification_error'))
        # 包装为带链接的 HTML，指向通知日志页面
        result = f'<div class="notification-error"><a href="{url_for("settings.notification_logs")}">{txt}</a></div>'
        output.append(result)
    
    res = "\n".join(output)
    return res
```

**发送到前端的数据** (socket_server.py:160-176):
```python
watch_data = {
    'error_text': error_texts,        # ← 编译后的错误文本（含 HTML）
    'has_error': True if error_texts else False,  # ← 是否有错误标记
    # ... 其他字段
}
socketio.emit("watch_update", {'watch': watch_data})
```

---

#### 2.3.3 通知日志页面

**路由**: `GET /notification-logs` (定义在 `blueprint/settings/__init__.py:313`)

**模板**: `blueprint/settings/templates/notification-log.html`

```html
<div id="notification-error-log">
    <ul>
    {% for log in logs|reverse %}
        <li>{{log}}</li>  <!-- 倒序显示最近 100 条 -->
    {% endfor %}
    </ul>
</div>
```

**数据来源**: 全局 `notification_debug_log` 列表

---

### 2.4 界面显示效果总结

| 显示位置 | 信息来源 | 显示内容 |
|---------|---------|---------|
| Watch 卡片上的错误提示 | `watch['last_notification_error']` → `compile_error_texts()` | "Notification error detected, goto notification log." (带链接) |
| Watch 卡片的错误标记 | `has_error: True if error_texts else False` | 红色指示器或图标 |
| 通知日志页面 (`/notification-logs`) | `notification_debug_log` 全局列表 | 最近 100 条发送记录和错误详情，倒序显示 |

---

## 3. 重试机制辨析：自动重试 vs 定时检查触发

### 3.1 核心结论

**系统不会对同一条失败通知自动重试。**

证据：

**代码路径**: `flask_app.py:1053-1102`

```python
def notification_runner(worker_id=0):
    while not app.config.exit.is_set():
        try:
            # 从队列取出
            n_object = notification_q.get(block=False)  # ← 取出
        except queue.Empty:
            app.config.exit.wait(1)
        else:
            try:
                sent_obj = process_notification(n_object, datastore)
            except Exception as e:
                # 失败时的处理
                logger.error(...)
                if 'uuid' in n_object:
                    datastore.update_watch(...)  # ← 记录错误
                log_lines = str(e).splitlines()
                notification_debug_log += log_lines
                
                # ⚠️ 关键：这里没有 notification_q.put(n_object)
                # ⚠️ 没有重新入队，没有重试机制
            
            # ⚠️ 无论成功失败，都继续循环处理下一条
            notification_debug_log += ["{} - SENDING - {}".format(...)]
```

**缺失的代码**: 没有任何形式的 `notification_q.put(n_object)` 将失败的通知重新入队。

---

### 3.2 可能被误解的"重试"：定时检查再次触发通知

当用户看到"下一次检查又发了通知"时，这**不是**对失败通知的重试，而是：

```
场景：
- Watch A 每隔 5 分钟检查一次
- 第 0 分钟：检测到变化，产生通知 N1，发送失败
- 第 5 分钟：定时检查，再次检测到变化（或内容仍不同），产生通知 N2，再次尝试发送
```

**关键区别**:

| 维度 | 同一条通知的自动重试 | 定时检查再次触发通知 |
|-----|---------------------|---------------------|
| **通知对象** | 同一个 `n_object` (同一个 `NotificationContextData`) | 新的 `n_object` (新时间戳、新快照) |
| **触发时机** | 失败后立即/延迟重试 | 下次 `check_interval` 到期 |
| **触发条件** | 前一次发送失败 | 内容仍有差异（`history_n >= 2`） |
| **差异内容** | 相同（同一次检测的快照） | 可能变化（如果页面又更新了） |
| **通知计数** | `notification_alert_count` 不增加 | `notification_alert_count` 增加 |
| **代码实现** | 不存在（没有重新入队） | `NotificationService.send_content_changed_notification()` |

---

### 3.3 通知发送的触发条件

**代码路径**: `worker.py` (实际调用 `NotificationService.send_content_changed_notification`)

```python
# 通知只会在这些条件下产生新的一条
if (
    watch.get('history_n') >= 2 
    and not watch.get('notification_muted')
    and (
        prev_md5 != fetched_md5  # 内容变了
        or explicit_reason_to_send_notification  # 或其他显式原因
    )
):
    # 产生新通知
    notification_service.send_content_changed_notification(watch_uuid)
```

**不会触发的情况**:
- 如果页面在下次检查前**恢复原状**，`history_n` 可能只有 1，不会触发通知
- 如果用户手动 `clear_watch()`，历史被清空，不会触发
- 如果内容已经在之前的检查中被确认"已查看"（`last_viewed` 更新），可能不触发

---

### 3.4 对比表格

| 特性 | 真正的重试机制 (不存在) | 定时检查再次触发 (实际行为) |
|-----|------------------------|---------------------------|
| **通知对象** | 同一个 `n_object` | 新建 `n_object` |
| **快照版本** | `from_ver` → `to_ver` 固定 | 可能是新的快照对 |
| **时间戳** | 相同 | 更新 |
| **失败计数** | 递增 (如果实现的话) | 全新计数 |
| **依赖检查结果** | 不依赖（失败即重试） | 依赖下次检查仍有差异 |
| **用户感知** | "系统在重试失败的通知" | "页面又变了，又发了一次" |

---

## 4. 精简时序图

### 4.1 正常通知发送时序

```
TickerThread           Worker                  NotificationRunner     Apprise/通道
     │                    │                          │                    │
     │  定时到期           │                          │                    │
     ├───────────────────►│                          │                    │
     │                    │  抓取、检测变化            │                    │
     │                    │  ─────────────            │                    │
     │                    │  产生 n_object            │                    │
     │                    │  notification_q.put()     │                    │
     │                    │  ────────────────         │                    │
     │                    │                          │                    │
     │                    │                          │ notification_q.get()│
     │                    │                          │ ─────────────────   │
     │                    │                          │                    │
     │                    │                          │ process_notification()
     │                    │                          │ ────────────────── │
     │                    │                          │                    │ apobj.add(url1)
     │                    │                          │                    │ ────────────────►
     │                    │                          │                    │ apobj.add(url2)
     │                    │                          │                    │ ────────────────►
     │                    │                          │                    │
     │                    │                          │                    │ apobj.notify()
     │                    │                          │                    │ ────────────────►
     │                    │                          │                    │        │
     │                    │                          │                    │        │ 发送成功
     │                    │                          │                    │ ◄────────
     │                    │                          │  返回 sent_objs     │
     │                    │                          │  ───────────────    │
```

---

### 4.2 通知发送失败时序（关键）

```
TickerThread           Worker                  NotificationRunner     Apprise/通道         Watch/存储
     │                    │                          │                    │                    │
     │  定时到期           │                          │                    │                    │
     ├───────────────────►│                          │                    │                    │
     │                    │  产生 n_object            │                    │                    │
     │                    │  notification_q.put()     │                    │                    │
     │                    │  ────────────────         │                    │                    │
     │                    │                          │                    │                    │
     │                    │                          │ notification_q.get()│                    │
     │                    │                          │ ─────────────────   │                    │
     │                    │                          │                    │                    │
     │                    │                          │ process_notification()                    │
     │                    │                          │ ────────────────── │                    │
     │                    │                          │                    │ apobj.notify()     │
     │                    │                          │                    │ ────────────────►  │
     │                    │                          │                    │        │           │
     │                    │                          │                    │        │ 发送失败 │
     │                    │                          │                    │        X           │
     │                    │                          │                    │                    │
     │                    │                          │ raise Exception()  │                    │
     │                    │                          │ ─────────────────  │                    │
     │                    │                          │    │               │                    │
     │                    │                          │    ▼ 捕获异常       │                    │
     │                    │                          │                    │                    │
     │                    │                          │ 1. logger.error()  │                    │
     │                    │                          │                    │                    │
     │                    │                          │ 2. datastore.update_watch({
     │                    │                          │      'last_notification_error': '...'
     │                    │                          │    })               │                    │
     │                    │                          │ ────────────────────────────────────────►│
     │                    │                          │                    │                    │  写入 JSON
     │                    │                          │                    │                    │
     │                    │                          │ 3. notification_debug_log += lines       │
     │                    │                          │                    │                    │
     │                    │                          │ 4. SIGNAL.send()  │                    │
     │                    │                          │ ───┐               │                    │
     │                    │                          │    │               │                    │
     │                    │ ◄────────────────────────┘    │               │                    │
     │                    │  Socket.IO 推送更新 UI        │               │                    │
     │                    │                               │               │                    │
     │                    │                          │ ⚠️ 没有重新入队    │                    │
     │                    │                          │ ⚠️ notification_q.put(n_object) 不存在   │
     │                    │                          │    │               │                    │
     │                    │                          │    ▼ 继续循环       │                    │
     │                    │                          │ notification_q.get() (下一条)            │
```

---

### 4.3 定时检查再次触发（非重试）

```
时间线: ──────────────────────────────────────────────────────────────────────►
         t0              t1              t2              t3              t4

t0: 第一次检查
    ├── 检测到变化（快照 v1 → v2）
    ├── 产生通知 N1 (from_v1, to_v2, timestamp=t0)
    └── N1 发送失败

t1: 下一个 check_interval 到期
    ├── 第二次检查
    ├── 检测仍有差异（快照仍是 v2，或已更新到 v3）
    ├── 产生通知 N2 (from_vx, to_vy, timestamp=t1)  ← 全新的通知对象
    └── 尝试发送 N2

关键区别:
  N1 永远丢失了（没有重新入队）
  N2 是全新生成的，不是对 N1 的重试
```

---

## 5. 关键代码位置索引

| 功能 | 文件 | 行号 | 关键函数/类 |
|-----|------|------|-----------|
| **通知工作器入口** | `flask_app.py` | 1053-1102 | `notification_runner()` |
| **通知工作器启动** | `flask_app.py` | 1001-1010 | `threading.Thread(target=notification_runner, ...)` |
| **实际发送** | `notification/handler.py` | 307-496 | `process_notification()` |
| **通知入队** | `notification_service.py` | 384, 475, 523 | `self.notification_q.put(n_object)` |
| **队列定义** | `queue_handlers.py` | 413-550 | `NotificationQueue` 类 |
| **错误记录** | `flask_app.py` | 1085-1097 | `except Exception as e:` 块 |
| **错误编译** | `model/Watch.py` | 1264-1300 | `compile_error_texts()` |
| **Socket.IO 推送** | `realtime/socket_server.py` | 62-84 | `SignalHandler.handle_signal()` |
| **Watch 数据推送** | `realtime/socket_server.py` | 139-199 | `handle_watch_update()` |
| **通知日志页面** | `blueprint/settings/__init__.py` | 313-319 | `notification_logs()` 路由 |
| **日志模板** | `blueprint/settings/templates/notification-log.html` | 1-19 | 显示 `notification_debug_log` |

---

## 6. 总结

### 6.1 三个核心问题的答案

**问题 1: 通知队列消费到发送的调用链路？**
```
notification_q.get()
    ↓
process_notification(n_object, datastore)
    ├── create_notification_parameters()
    ├── add_rendered_diff_to_notification_vars()
    ├── jinja_render() 模板渲染
    ├── apply_service_tweaks() 格式转换
    ├── apobj.add(url) 添加所有通道
    └── apobj.notify() 实际发送
```

**问题 2: 发送失败时错误记录到哪里、界面怎么感知？**
- **记录位置 1**: `watch['last_notification_error']`（持久化到 JSON）
- **记录位置 2**: 全局 `notification_debug_log` 列表（内存，最近 100 条）
- **界面感知**: 通过 `watch_check_update_SIGNAL` → `compile_error_texts()` → Socket.IO `watch_update` 事件推送到前端；用户可点击链接跳转到 `/notification-logs` 查看详细日志

**问题 3: 系统会不会自动重试失败通知？和定时检查再次触发有什么区别？**
- **不会自动重试**：失败的 `n_object` 没有重新入队，直接丢弃
- **区别**：定时检查再次触发的是**全新的通知对象**（新时间戳、可能新快照），不是对失败通知的重试；原失败通知已永久丢失

### 6.2 设计评价

**优点**:
- 简单：失败即丢弃，避免队列积压
- 可观测：错误记录明确，用户可通过日志排查
- 灵活：通过定时检查实现"最终一致性"（只要页面继续变化，总会有新通知）

**潜在问题**:
- 瞬态网络错误导致的通知可能永久丢失
- 无法区分"发送失败"和"内容已变化需重新通知"
- 多通道配置下，一个通道失败导致其他已成功的通道也被视为失败（因为 `apobj.notify()` 在循环外）

---

*报告生成时间: 2026-05-12*
*分析方法: 纯代码阅读，无运行验证*
