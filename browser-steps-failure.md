# Browser Steps 单步失败链路分析报告

## 1. 异常对象定义与字段传递

### 1.1 异常类定义

**文件位置**：`changedetectionio/content_fetchers/exceptions/__init__.py:45-50`

```python
class BrowserStepsStepException(Exception):
    def __init__(self, step_n, original_e):
        self.step_n = step_n          # 步骤序号（从1开始）
        self.original_e = original_e  # 原始Playwright异常对象
        logger.debug(f"Browser Steps exception at step {self.step_n} {str(original_e)}")
        return
```

**关键代码证据**：
- 异常类**未调用父类 `Exception.__init__()`**，直接初始化两个实例属性
- `step_n`：整数，表示失败的步骤序号
- `original_e`：Playwright 原始异常对象（`TimeoutError` 或 `Error`）

### 1.2 异常抛出点与 step_n 传递

**文件位置**：`changedetectionio/content_fetchers/base.py:165-199`

```python
async def iterate_browser_steps(self, start_url=None):
    step_n = 0  # 初始化为0

    for step in valid_steps:
        step_n += 1  # 进入循环立即+1，第一个步骤step_n=1
        logger.debug(f">> Iterating check - browser Step n {step_n} - {step['operation']}...")
        
        try:
            await interface.call_action(...)
        except (Error, TimeoutError) as e:
            # 抛出时step_n已是从1开始的序号
            raise BrowserStepsStepException(step_n=step_n, original_e=e)
```

**步骤序号传递关系**：
| 代码位置 | 变量 | 值含义 |
|---------|------|--------|
| base.py:169 | `step_n = 0` | 初始化 |
| base.py:177 | `step_n += 1` | 第一个步骤执行前变为 1 |
| base.py:199 | `step_n=step_n` | 异常对象中存储的是 1-based 序号 |
| worker.py:327 | `error_step = e.step_n + 1` | **显示时再次+1，存在潜在Bug** |

---

## 2. 连续失败计数机制

### 2.1 字段名与初始值

**文件位置**：`changedetectionio/worker.py:346-357`

```python
if watch.get('filter_failure_notification_send', False):
    # 读取连续失败计数，默认0
    c = watch.get('consecutive_filter_failures', 0)
    c += 1  # 计数+1
    
    threshold = datastore.data['settings']['application'].get(
        'filter_failure_notification_threshold_attempts', 0
    )
    
    if threshold > 0 and c >= threshold:
        if not watch.get('notification_muted'):
            await send_step_failure_notification(...)
        c = 0  # 发送通知后重置计数
    
    datastore.update_watch(uuid=uuid, update_obj={'consecutive_filter_failures': c})
```

### 2.2 准确字段名对照表

| 字段名 | 位置 | 类型 | 默认值 | 说明 |
|--------|------|------|--------|------|
| **`consecutive_filter_failures`** | Watch对象 | int | 0 | **实际使用的连续失败计数字段** |
| **`filter_failure_notification_send`** | Watch对象 | bool | False | **通知功能总开关** |
| **`filter_failure_notification_threshold_attempts`** | 全局设置 | int | 0 | 触发通知的阈值 |
| **`notification_muted`** | Watch对象 | bool | False | 单个监控的通知静音开关 |

---

## 3. 阈值触发与重置规则

### 3.1 触发通知的完整条件链

**所有条件必须同时满足**：

```
条件1: watch['filter_failure_notification_send'] == True
    AND
条件2: threshold > 0
    AND
条件3: consecutive_filter_failures >= threshold
    AND
条件4: watch['notification_muted'] == False
```

**代码证据**（worker.py:346-355）：
```python
if watch.get('filter_failure_notification_send', False):  # 条件1
    c = watch.get('consecutive_filter_failures', 0)
    c += 1
    threshold = datastore.data['settings']['application'].get(
        'filter_failure_notification_threshold_attempts', 0
    )
    if threshold > 0 and c >= threshold:  # 条件2 + 条件3
        if not watch.get('notification_muted'):  # 条件4
            await send_step_failure_notification(...)
        c = 0  # 发送后重置计数
```

### 3.2 计数重置规则

**重置时机**：
- 仅在 `threshold > 0 and c >= threshold` 条件满足后重置
- 无论通知是否因 `notification_muted` 被跳过，计数器都会重置为 0

**不重置场景**：
- `filter_failure_notification_send == False`：计数不更新
- `threshold == 0`：计数持续累加但不触发通知
- 未达到阈值：计数保留，下次失败时继续累加

---

## 4. 错误信息处理流程

### 4.1 原始错误信息提取

**文件位置**：`changedetectionio/worker.py:328-338`

```python
from playwright._impl._errors import TimeoutError, Error

# 默认错误提示
err_text = f"Browser step at position {error_step} could not run, check the watch, add a delay if necessary, view Browser Steps to see screenshot at that step."

# 根据异常类型追加信息
if e.original_e.name == "TimeoutError":
    err_text += " Could not find the target."  # TimeoutError只追加固定提示
else:
    # 其他Error类型取异常信息第一行
    err_text += " " + str(e.original_e).splitlines()[0]
```

### 4.2 错误信息持久化字段

| 字段 | 值来源 | 示例 |
|------|--------|------|
| `last_error` | 构造的 `err_text` | "Browser step at position 2 could not run... Could not find the target." |
| `browser_steps_last_error_step` | `e.step_n + 1` | 2 |

**代码证据**（worker.py:342-344）：
```python
datastore.update_watch(uuid=uuid,
                     update_obj={'last_error': err_text,
                               'browser_steps_last_error_step': error_step})
```

---

## 5. 通知内容来源与构建

### 5.1 通知标题与正文

**文件位置**：`changedetectionio/notification_service.py:480-524`

```python
def send_step_failure_notification(self, watch_uuid, step_n):
    watch = self.datastore.data['watching'].get(watch_uuid, False)
    threshold = self.datastore.data['settings']['application'].get(
        'filter_failure_notification_threshold_attempts'
    )
    
    step = step_n + 1  # 显示序号再次+1
    
    # 硬编码标题
    notification_title = f"Changedetection.io - Alert - Browser step at position {step} could not be run"
    
    # 硬编码正文模板
    body = f"""Hello,

Your configured browser step at position {step} for the web page watch {{{{watch_url}}}} did not appear on the page after {threshold} attempts, did the page change layout?

The element may have moved and needs editing, or does it need a delay added?

Edit link: {{{{base_url}}}}/edit/{{{{watch_uuid}}}}

Thanks - Your omniscient changedetection.io installation.
"""
```

### 5.2 通知内容字段来源表

| 通知字段 | 数据来源 | 说明 |
|---------|----------|------|
| **`notification_title`** | 硬编码 f-string | 包含 `{step}` 变量 |
| **`notification_body`** | 硬编码多行f-string | 包含 `{step}`、`{threshold}` 变量 |
| **`notification_format`** | `_check_cascading_vars()` | 级联获取：Watch → Tag → Global |
| **`notification_urls`** | 级联获取 | 优先 `watch['notification_urls']`，否则全局配置 |
| **`watch_url`** | `watch['url']` | 监控页面URL |
| **`uuid` / `watch_uuid`** | 传入参数 | 监控项唯一标识 |

### 5.3 通知调用链

```
worker.py:354 → send_step_failure_notification(watch_uuid, step_n, ...)
    ↓ (worker.py:782-792)
notification_service.send_step_failure_notification(watch_uuid, step_n)
    ↓ (notification_service.py:480-524)
构建 NotificationContextData → 放入 notification_q 队列
```

---

## 6. 关键调用链与数据流向

### 6.1 完整失败处理流

```
Playwright抛出TimeoutError/Error
    ↓ [base.py:196-199]
iterate_browser_steps捕获 → 包装BrowserStepsStepException
    ↓ 抛出时携带: step_n(1-based), original_e
    ↓ [worker.py:323-359]
worker捕获异常
    ├─ 计算 error_step = e.step_n + 1  ← 潜在Bug:序号多加1
    ├─ 根据 original_e.name 构造 err_text
    ├─ 更新 last_error 和 browser_steps_last_error_step
    │
    └─ filter_failure_notification_send == True?
        ├─ 是 → consecutive_filter_failures += 1
        │       └─ threshold > 0 AND 计数 >= threshold?
        │           ├─ 是 → notification_muted == False?
        │           │       ├─ 是 → 发送通知
        │           │       └─ 否 → 跳过通知
        │           └─ 无论是否发送通知: 重置计数 = 0
        │
        └─ 否 → 不更新计数，直接结束
```

### 6.2 screenshot_step 命名规则

**文件位置**：`changedetectionio/content_fetchers/base.py:179-180,194-195`

| 时机 | 调用代码 | 生成文件名 |
|------|---------|-----------|
| 步骤执行前 | `screenshot_step("before-" + str(step_n))` | `step_before-1.jpeg` |
| 步骤执行后 | `screenshot_step(step_n)` | `step_1.jpeg` |

---

## 7. 代码问题与Bug确认

### 7.1 Bug: 步骤序号显示不一致

**问题**：
- 异常中 `step_n` 已是 1-based（第一个步骤失败时 `step_n=1`）
- `worker.py:327` 再次 `+1` → `error_step = 2`
- 导致用户看到的步骤序号比实际配置的序号大1

**证据链**：
```
base.py:169: step_n = 0
base.py:177: step_n += 1 → step_n = 1
base.py:199: raise BrowserStepsStepException(step_n=1, ...)
worker.py:327: error_step = e.step_n + 1 → error_step = 2  ← 错误
```

### 7.2 Bug: Exception父类未初始化

**问题**：
- `BrowserStepsStepException.__init__` 未调用 `super().__init__()`
- 导致 `str(exception)` 可能返回空字符串或非预期结果

**证据**（exceptions/__init__.py:46-50）：
```python
def __init__(self, step_n, original_e):
    self.step_n = step_n
    self.original_e = original_e
    # 缺少 super().__init__(f"Step {step_n} failed: {original_e}")
    logger.debug(...)
    return
```

---

## 8. 附录：关键文件与行号索引

| 文件路径 | 关键代码行号 | 说明 |
|---------|-------------|------|
| `changedetectionio/content_fetchers/exceptions/__init__.py` | 45-50 | BrowserStepsStepException定义 |
| `changedetectionio/content_fetchers/base.py` | 165-199 | iterate_browser_steps异常抛出 |
| `changedetectionio/worker.py` | 323-359 | Browser Steps异常处理完整逻辑 |
| `changedetectionio/worker.py` | 346-357 | 连续失败计数与阈值判断 |
| `changedetectionio/worker.py` | 782-792 | send_step_failure_notification包装函数 |
| `changedetectionio/notification_service.py` | 480-524 | 通知内容构建 |
