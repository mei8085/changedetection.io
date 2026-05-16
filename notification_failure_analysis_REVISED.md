# 通知失败完整链路与并发分析报告（修订版）

## 目录
1. [概述](#1-概述)
2. [完整调用链路](#2-完整调用链路)
3. [NOTIFICATION_WORKERS并发机制与竞争风险](#3-notification_workers并发机制与竞争风险)
4. [数据模型中的存储结构](#4-数据模型中的存储结构)
5. [错误字段落盘机制](#5-错误字段落盘机制)
6. [last_notification_error展示读取链路](#6-last_notification_error展示读取链路)
7. [日志页与监测项视图的数据源边界](#7-日志页与监测项视图的数据源边界)
8. [关键代码位置速查](#8-关键代码位置速查)
9. [设计评价与改进建议](#9-设计评价与改进建议)

---

## 1. 概述

本报告在原始分析基础上，补充了以下关键内容：
- ✅ 基于 `NOTIFICATION_WORKERS` 配置的并发通知机制深度分析
- ✅ 多线程并发写入 `notification_debug_log` 的竞争风险评估
- ✅ 现有代码中的并发保护（或缺）分析
- ✅ `last_notification_error` 从写入到页面展示的完整读取链路
- ✅ 日志页与监测项视图各自数据源的明确边界划分

---

## 2. 完整调用链路

### 2.1 链路总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     通知失败完整处理链路                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  1. [Worker] 变更检测触发                                                 │
│     └── async_update_worker() → processor.perform_site_check()            │
│                                                                           │
│  2. [Processor] 检测到变更 → changed_detected=True                         │
│     └── update_handler.run_changedetection()                               │
│                                                                           │
│  3. [Notification Service] 通知入队                                        │
│     └── send_content_changed_notification() → notification_q.put()        │
│                                                                           │
│  4. [Notification Runner Pool] N个并发线程消费队列                         │
│     └── notification_runner(0), notification_runner(1)...                 │
│                                                                           │
│  5. [Exception] 发送失败 → 异常捕获                                        │
│     └── try-except 块捕获 Apprise/网络异常                                  │
│                                                                           │
│  6. [错误持久化] 双路写入  ⚠️ 并发风险区                                    │
│     ├── 写入1: Watch对象 → last_notification_error 字段 (有锁保护)        │
│     └── 写入2: 全局内存 → notification_debug_log 列表 (无锁！)            │
│                                                                           │
│  7. [落盘] datastore.update_watch() → watch.commit() → JSON文件           │
│                                                                           │
│  8. [展示] 两条独立读取链路                                                │
│     ├── 链路A: 监测项页面 → 读取 watch['last_notification_error']        │
│     └── 链路B: 日志页面 → 读取全局 notification_debug_log                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. NOTIFICATION_WORKERS并发机制与竞争风险

### 3.1 并发配置与启动

**配置位置**: `changedetectionio/flask_app.py:1001-1010`

```python
# flask_app.py:1001-1010 - 并发通知工作者启动
notification_workers = int(os.getenv("NOTIFICATION_WORKERS", "1"))
for i in range(notification_workers):
    threading.Thread(
        target=notification_runner,
        args=(i,),
        daemon=True,
        name=f"NotificationRunner-{i}"
    ).start()
logger.info(f"Started {notification_workers} notification worker(s)")
```

**配置说明**:
- 环境变量: `NOTIFICATION_WORKERS`
- 默认值: `1` (单线程)
- 线程命名: `NotificationRunner-0`, `NotificationRunner-1`, ...
- 守护线程: 随主进程退出而终止

### 3.2 并发消费模型

```
                          notification_q (Queue.Queue)
                              ┌──────────────┐
                              │  通知对象1   │
┌─────────────────┐           │  通知对象2   │           ┌─────────────────┐
│ NotificationRunner-0 │ ←──  │  通知对象3   │  ──→ │ NotificationRunner-1 │
└─────────────────┘           │  ...         │           └─────────────────┘
       │                      └──────────────┘                  │
       │                                                         │
       ↓                                                         ↓
process_notification()                              process_notification()
       │                                                         │
       ↓                                                         ↓
成功/失败 → 写入日志                                  成功/失败 → 写入日志
```

**队列特性**:
- 使用 `queue.Queue` 标准库
- `get(block=False)`: 非阻塞获取
- Queue内部有锁保护，消费操作是线程安全的

### 3.3 并发写入notification_debug_log的实际行为

**关键代码**: `flask_app.py:1085-1102`

```python
def notification_runner(worker_id=0):
    global notification_debug_log
    with app.app_context():
        while not app.config.exit.is_set():
            try:
                n_object = notification_q.get(block=False)
            except queue.Empty:
                app.config.exit.wait(1)
            else:
                try:
                    sent_obj = process_notification(n_object, datastore)
                except Exception as e:
                    # ⚠️ 并发写入点1: 异常堆栈写入
                    log_lines = str(e).splitlines()
                    notification_debug_log += log_lines  # ← 非原子操作!
                    
                    # 写入watch对象 (有锁保护)
                    if 'uuid' in n_object:
                        datastore.update_watch(uuid=n_object['uuid'],
                                               update_obj={'last_notification_error': "..."})
                
                # ⚠️ 并发写入点2: 发送记录写入
                notification_debug_log += ["{} - SENDING - {}".format(now.strftime("%c"), json.dumps(sent_obj))]
                
                # ⚠️ 并发写入点3: 截断操作
                notification_debug_log = notification_debug_log[-100:]  # ← 读-修改-写!
```

### 3.4 竞争风险详细分析

#### 风险1: `list +=` 操作不是原子的

```python
# 看似一行代码，实际包含3个步骤
notification_debug_log += log_lines

# 等价于:
temp = notification_debug_log.__iadd__(log_lines)  # 步骤1: 读取并扩展
notification_debug_log = temp                       # 步骤2: 赋值
```

**并发场景下的问题**:
```
时序:
  线程A: 读取列表 [a, b, c]
  线程B: 读取列表 [a, b, c]
  线程A: 追加 [d, e] → [a, b, c, d, e]
  线程B: 追加 [f, g] → [a, b, c, f, g]  ← 丢失了d, e!
  结果: 列表变成 [a, b, c, f, g]，丢失了线程A的写入
```

#### 风险2: 截断操作的读-修改-写竞争

```python
# 截断操作也是非原子的
notification_debug_log = notification_debug_log[-100:]

# 等价于:
temp_slice = notification_debug_log[-100:]  # 步骤1: 读取切片
notification_debug_log = temp_slice          # 步骤2: 赋值
```

**并发场景下的问题**:
```
时序:
  线程A: 读取切片 (第101-200条)
  线程B: 写入新的3条日志
  线程A: 赋值切片回列表  ← 线程B的3条日志被覆盖丢失!
```

#### 风险3: GIL不能提供保护

**常见误解**: "Python有GIL，列表操作是线程安全的"

**真相**:
- GIL只保证**单个字节码指令**的原子性
- `list +=` 是**多个字节码**的复合操作
- 线程切换可能发生在字节码之间
- 证据: `dis.dis("lst += items")` 显示多条指令

```python
import dis
dis.dis("lst += [1, 2, 3]")

# 输出（简化）:
#   LOAD_NAME     0 (lst)
#   LOAD_CONST    0 ([1, 2, 3])
#   INPLACE_ADD        ← 这里可能发生线程切换!
#   STORE_NAME    0 (lst)
```

### 3.5 现有保护机制分析

| 写入目标 | 是否有保护 | 保护方式 | 位置 |
|---------|-----------|---------|------|
| `notification_debug_log` | ❌ **无保护** | - | `flask_app.py:1094, 1100, 1102` |
| `watch['last_notification_error']` | ✅ 有保护 | `datastore.lock` 互斥锁 | `store/__init__.py` |

**为什么datastore.update_watch是安全的**:

```python
def update_watch(self, uuid, update_obj):
    with self.lock:  # ← 关键: 互斥锁保护整个操作
        if uuid not in self.__data['watching']:
            raise KeyError(f"Watch UUID {uuid} not found")
        watch = self.__data['watching'][uuid]
        watch.update(update_obj)
        watch.commit()  # ← 锁保护下的磁盘写入
```

### 3.6 并发问题的实际影响

**低概率但真实存在**:
- 高并发通知场景下（`NOTIFICATION_WORKERS > 5`）
- 大量通知同时失败时
- 概率性出现日志丢失、列表损坏

**可能的症状**:
1. 部分通知日志条目丢失（最常见）
2. 日志条目顺序错乱
3. 极端情况下 `IndexError`（如果切片操作时列表被清空）

**当前风险等级**: 低 → 中
- 默认配置是单线程（`NOTIFICATION_WORKERS=1`），无问题
- 用户显式配置多线程时才会暴露风险

---

## 4. 数据模型中的存储结构

### 4.1 双存储体系概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        通知错误双存储体系                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────┐    ┌──────────────────────────┐  │
│  │     持久化存储 (A数据源)     │    │    内存存储 (B数据源)    │  │
│  │                              │    │                          │  │
│  │  位置: watch.json           │    │  位置: 全局变量          │  │
│  │  字段: last_notification_error │  │  变量: notification_debug_log │
│  │  类型: str/False            │    │  类型: list[str]        │  │
│  │  范围: 单Watch              │    │  范围: 所有通知          │  │
│  │  并发: 有datastore.lock保护 │    │  并发: 无锁保护          │  │
│  └──────────────┬───────────────┘    └────────────┬─────────────┘  │
│                 │                                   │                │
│                 ▼                                   ▼                │
│  ┌──────────────────────────────┐    ┌──────────────────────────┐  │
│  │   监测项列表/编辑页展示      │    │    通知日志页展示        │  │
│  │   (Watch Overview)          │    │    (Notification Logs)   │  │
│  └──────────────────────────────┘    └──────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Watch对象字段定义

**定义位置**: `changedetectionio/model/__init__.py:339-340`

```python
# model/__init__.py - watch_base类默认值
class watch_base(dict):
    def __init__(self, *arg, **kw):
        self.update({
            # ... 其他字段
            'last_error': False,              # 通用检测错误
            'last_notification_error': False, # 通知错误标记 (新字段)
            # ... 其他字段
        })
```

**字段状态说明**:
- `False`: 无通知错误（正常状态）
- 字符串: 存在通知错误（错误提示文本）

---

## 5. 错误字段落盘机制

### 5.1 落盘完整调用链

```
notification_runner捕获Exception
        │
        ▼
  datastore.update_watch()
        │
        ▼
  ┌────────────────────┐
  │  获取 datastore.lock
  │    (阻塞等待)
  └──────────┬─────────┘
        │
        ▼
  watch.update(update_obj)
  (内存中更新last_notification_error)
        │
        ▼
  watch.commit()
        │
        ▼
  EntityPersistenceMixin._save_to_disk()
        │
        ▼
  save_entity_atomic()
        │
        ▼
  save_json_atomic() → 原子写入磁盘
        │
        ▼
  释放 datastore.lock
```

### 5.2 关键落盘代码

```python
# notification_runner中的写入触发
except Exception as e:
    if 'uuid' in n_object:
        # ↓ 进入带锁保护的写入流程
        datastore.update_watch(
            uuid=n_object['uuid'],
            update_obj={'last_notification_error': "Notification error detected, goto notification log."}
        )
```

**原子写入保证**:
- 临时文件写入 + `os.replace()` 重命名
- 避免崩溃时的文件损坏
- 同目录操作保证POSIX原子性

---

## 6. last_notification_error展示读取链路

### 6.1 完整读取链路图

```
┌─────────────────────────────────────────────────────────────────────┐
│               last_notification_error 读取展示链路                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. [HTTP Request] 用户访问监测项页面                               │
│     GET / → watchlist.index()                                       │
│     GET /edit/<uuid> → ui.edit()                                    │
│                                                                     │
│  2. [Blueprint Controller] 从datastore读取watch对象                 │
│     watch = datastore.data['watching'][uuid]                        │
│     (直接内存读取，不访问磁盘)                                       │
│                                                                     │
│  3. [Model Method] 调用Watch.extra_notification_error_text()        │
│     生成HTML格式的错误提示                                           │
│                                                                     │
│  4. [Template Render] Jinja2模板渲染到HTML                          │
│     watch-overview.html → Watch列表展示                             │
│     edit.html → 编辑页展示                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 核心展示方法实现

**位置**: `changedetectionio/model/Watch.py:1293-1299`

```python
# Watch.py:1293-1299 - 生成通知错误展示HTML
def extra_notification_error_text(self):
    """
    生成监测项页面中展示的通知错误HTML
    
    返回示例:
    '<div class="notification-error">
      <a href="/settings/notification-logs">Notification error detected, goto notification log.</a>
    </div>'
    """
    output = []
    
    # ... 其他错误处理逻辑
    
    if self.get('last_notification_error'):
        # 1. 安全转义用户输入（防止XSS）
        txt = safe_jinja.render_fully_escaped(self.get('last_notification_error'))
        # 2. 生成带日志页链接的HTML
        result = f'<div class="notification-error"><a href="{url_for("settings.notification_logs")}">{txt}</a></div>'
        output.append(str(Markup(result)))  # Markup标记为安全HTML
    
    return ''.join(output)
```

**方法特性**:
1. **XSS防护**: 使用 `render_fully_escaped()` 完全转义
2. **语义化HTML**: 带 `notification-error` CSS类的div
3. **跳转链接**: 直接链接到通知日志页面
4. **返回类型**: 安全的HTML字符串（通过Markup包装）

### 6.3 监测项列表页面展示

**调用位置1**: Watch列表页面

```python
# blueprint/watchlist/__init__.py:90-120 - 渲染watch-overview.html
output = render_template(
    "watch-overview.html",
    watches=sorted_watches,  # ← 每个watch都有last_notification_error字段
    # ... 其他参数
)
```

**模板中的展示**:
- 在每个Watch卡片的状态区域展示
- 红色/警告样式
- 点击链接跳转到通知日志页

**调用位置2**: Watch编辑页面

```python
# blueprint/ui/__init__.py - 渲染编辑页
# Watch对象直接传递给模板
# 通过 {{ watch.last_notification_error }} 访问
```

### 6.4 读取性能特性

| 特性 | 值 |
|------|----|
| 读取源 | 内存中的Watch对象（不是磁盘） |
| IO操作 | 0次（完全内存访问） |
| 延迟 | <1μs |
| 线程安全 | 只读操作天然安全 |
| 一致性 | 最终一致（写入后立即可读） |

---

## 7. 日志页与监测项视图的数据源边界

### 7.1 数据源对比矩阵

| 维度 | 监测项视图 (Watch Overview) | 通知日志页 (Notification Logs) |
|------|----------------------------|--------------------------------|
| **数据源A** | `watch['last_notification_error']` | - |
| **数据源B** | - | `notification_debug_log` |
| **存储类型** | 磁盘持久化(JSON) | 内存列表 |
| **数据粒度** | 单Watch状态标记 | 所有通知的详细日志 |
| **内容类型** | 固定提示文本 | 异常堆栈 + 发送记录 |
| **历史保留** | 仅最新状态(覆盖) | 最近100条(滚动窗口) |
| **重启后保留** | ✓ 持久化 | ✗ 丢失 |
| **读取方式** | datastore内存对象直接访问 | 全局变量import |
| **展示位置** | Watch列表/编辑页 | /settings/notification-logs |
| **URL** | / (首页) | /settings/notification-logs |
| **并发保护** | 读取无锁(只读安全) | 读取无锁（只读安全） |
| **面向用户** | 普通监控用户 | 管理员/开发者 |

### 7.2 两条独立展示链路

```
┌─────────────────────────────────────────────────────────────────────┐
│                        两条独立的展示链路                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  链路A: 监测项视图 → 读取持久化字段                          │   │
│  ├────────────────────────────────────────────────────────────┤   │
│  │  用户访问 / 首页                                             │   │
│  │     ↓ watchlist.index()                                     │   │
│  │  从datastore读取sorted_watches列表                           │   │
│  │     ↓ 模板循环渲染每个watch                                 │   │
│  │  watch.extra_notification_error_text()                      │   │
│  │     ↓ 读取 watch['last_notification_error']                │   │
│  │  生成带链接的错误HTML → 展示在Watch卡片                     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  链路B: 通知日志页 → 读取内存日志                           │   │
│  ├────────────────────────────────────────────────────────────┤   │
│  │  用户访问 /settings/notification-logs                      │   │
│  │     ↓ settings.notification_logs()                         │   │
│  │  from flask_app import notification_debug_log              │   │
│  │     ↓ (直接引用全局变量)                                    │   │
│  │  传给模板 notification-log.html                            │   │
│  │     ↓ Jinja2 reverse倒序                                   │   │
│  │  <ul><li> 循环输出每条日志 → 最新在最上面                   │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 日志页读取实现

**路由位置**: `changedetectionio/blueprint/settings/__init__.py:313-319`

```python
@settings_blueprint.route("/notification-logs", methods=['GET'])
@login_optionally_required
def notification_logs():
    """通知日志页 - 读取全局内存日志变量"""
    # ↓ 直接从全局命名空间import
    from changedetectionio.flask_app import notification_debug_log
    
    # ↓ 空列表处理
    logs_to_render = notification_debug_log if len(notification_debug_log) \
                     else ["Notification logs are empty - no notifications sent yet."]
    
    return render_template("notification-log.html", logs=logs_to_render)
```

**模板渲染**:
```html
<!-- notification-log.html:8-12 -->
<div id="notification-error-log">
    <ul style="font-size: 80%; margin: 0; padding: 0 0 0 7px;">
        {% for log in logs|reverse %}  {# ← 倒序: 最新在最上面 #}
        <li>{{ log }}</li>             {# ← 纯文本展示 #}
        {% endfor %}
    </ul>
</div>
```

### 7.4 边界清晰性总结

| 边界类型 | 清晰程度 | 说明 |
|---------|---------|------|
| **存储边界** | ✅ 完全清晰 | 一个持久化一个内存，物理分离 |
| **访问边界** | ✅ 完全清晰 | 两条独立代码路径，无交叉读取 |
| **展示边界** | ✅ 完全清晰 | 不同页面，不同UI风格 |
| **用户边界** | ✅ 基本清晰 | 面向不同用户群体 |
| **写入边界** | ⚠️ 注意 | 同一处代码同时写入两个存储 |

**关键设计决策**:
> 故意不让通知日志页读取Watch的 `last_notification_error` 字段
> 因为日志页的目标是展示**详细的历史记录**，而不是状态标记

---

## 8. 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| **并发配置** | | |
| NOTIFICATION_WORKERS启动 | `flask_app.py` | 1001-1010 |
| notification_runner主循环 | `flask_app.py` | 1053-1102 |
| notification_debug_log写入点1(异常) | `flask_app.py` | 1094 |
| notification_debug_log写入点2(发送记录) | `flask_app.py` | 1100 |
| notification_debug_log截断 | `flask_app.py` | 1102 |
| **持久化与落盘** | | |
| datastore.update_watch | `store/__init__.py` | - |
| watch.commit() | `model/__init__.py` | - |
| EntityPersistenceMixin._save_to_disk | `model/persistence.py` | 52-84 |
| save_json_atomic原子写入 | `store/file_saving_datastore.py` | 36-175 |
| **展示读取链路** | | |
| Watch.extra_notification_error_text | `model/Watch.py` | 1293-1299 |
| watchlist.index()路由 | `blueprint/watchlist/__init__.py` | 16-143 |
| notification_logs()路由 | `blueprint/settings/__init__.py` | 313-319 |
| 通知日志模板 | `notification-log.html` | 8-12 |
| **数据模型** | | |
| last_notification_error默认值 | `model/__init__.py` | 339-340 |
| notification_debug_log定义 | `flask_app.py` | - |

---

## 9. 设计评价与改进建议

### 9.1 现有设计优点

1. **双存储职责清晰**
   - 持久化字段: 状态标记，面向普通用户
   - 内存日志: 详细调试，面向管理员
   - 边界清晰，互不干扰

2. **Watch写入线程安全**
   - datastore.lock保护整个update_watch流程
   - 包括内存更新和磁盘写入
   - 无并发问题

3. **读取天然安全**
   - 两个数据源的读取都是只读操作
   - 无读-修改-写竞争
   - 性能优秀（内存访问）

4. **日志滚动窗口设计**
   - 限制100条防止内存泄漏
   - FIFO策略符合调试需求
   - 重启后自动清空避免过期日志干扰

### 9.2 已发现的问题

#### 问题1: notification_debug_log并发写入无锁保护

**严重程度**: 中（默认配置下安全，多线程配置下有风险）

**症状**:
- 日志条目丢失
- 条目顺序错乱
- 极端情况列表损坏

#### 问题2: 错误信息不一致

- `last_notification_error` 只有固定文本
- 详细错误只在内存日志
- 用户无法从监测项页面看到具体错误原因

### 9.3 具体改进建议

#### 改进A: 为notification_debug_log添加线程锁

**实施成本**: 极低
**预期收益**: 消除并发风险

```python
# 在flask_app.py添加全局锁
notification_debug_log_lock = threading.Lock()
notification_debug_log = []

# 在notification_runner中使用
def notification_runner(worker_id=0):
    global notification_debug_log, notification_debug_log_lock
    # ...
    except Exception as e:
        log_lines = str(e).splitlines()
        with notification_debug_log_lock:  # ← 添加锁
            notification_debug_log += log_lines
    
    # 写入发送记录
    with notification_debug_log_lock:  # ← 添加锁
        notification_debug_log += ["{} - SENDING - {}".format(...)]
        notification_debug_log = notification_debug_log[-100:]
```

#### 改进B: 使用collections.deque代替list

**实施成本**: 低
**预期收益**: 更高效的append操作，天然滚动窗口

```python
from collections import deque

# 初始化时指定maxlen自动滚动
notification_debug_log = deque(maxlen=100)

# 自动截断，不需要手动[-100:]
notification_debug_log.append("log message")  # 自动超出时弹出最早的
notification_debug_log.extend(log_lines)      # 批量追加
```

**为什么deque更好**:
- `append()` 和 `extend()` 是原子操作（C实现）
- `maxlen` 参数实现自动滚动窗口
- 性能更高（O(1) vs O(n)截断）

#### 改进C: last_notification_error存储更多信息

**实施成本**: 中
**预期收益**: 用户体验提升，不用跳转就能看到基本错误

```python
# notification_runner中写入更详细的信息
update_obj = {
    'last_notification_error': {
        'timestamp': int(time.time()),
        'message': str(e)[:100],  # 限制长度
        'hint': 'See notification logs for full stack trace'
    }
}
datastore.update_watch(uuid, update_obj)
```

#### 改进D: 按Watch UUID过滤日志页

**实施成本**: 中
**预期收益**: 调试体验大幅提升

```python
# 新增路由: /settings/notification-logs/<uuid>
@settings_blueprint.route("/notification-logs/<uuid>", methods=['GET'])
def notification_logs_for_watch(uuid):
    from changedetectionio.flask_app import notification_debug_log
    
    # 解析每条日志，过滤出该Watch的记录
    # 需要日志格式中包含uuid信息
    filtered = [line for line in notification_debug_log if uuid in line]
    
    return render_template("notification-log.html", 
                           logs=filtered if filtered else ["No logs for this watch"])
```

### 9.4 架构思考

**当前架构的合理性**:
- 对于绝大多数场景（`NOTIFICATION_WORKERS=1`），完全没有问题
- 双存储体系的设计非常经典：持久化状态 + 内存调试日志
- 边界清晰，职责分明

**演进方向**:
- 随着通知量增长，需要强化多线程安全性
- 随着企业用户增加，需要更友好的错误展示
- 随着SLA要求提高，需要持久化日志历史

---

## 总结

本修订版报告完整覆盖了:

1. **并发机制**: `NOTIFICATION_WORKERS` 配置下的多线程工作模型
2. **竞争分析**: `notification_debug_log` 并发写入的风险点与现有保护评估
3. **读取链路**: `last_notification_error` 从写入到Watch页面展示的完整路径
4. **边界划分**: 明确了日志页与监测项视图各自的数据源与边界

**关键发现**:
- ✅ Watch持久化写入有完整的锁保护，线程安全
- ⚠️ `notification_debug_log` 多线程写入无锁保护，高并发配置下有丢失风险
- ✅ 两个展示链路完全独立，各自读取不同数据源，边界清晰
- ✅ 现有设计在默认配置下工作良好，问题仅在显式配置多线程时暴露
