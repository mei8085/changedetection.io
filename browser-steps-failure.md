# Browser Steps 单步失败链路分析报告

## 1. 异常对象定义与字段传递

### 1.1 异常类定义

**文件位置**：`changedetectionio/content_fetchers/exceptions/__init__.py:45-50`

```python
class BrowserStepsStepException(Exception):
    def __init__(self, step_n, original_e):
        self.step_n = step_n          # 步骤序号（从1开始）
        self.original_e = original_e  # Playwright原始异常对象
        logger.debug(f"Browser Steps exception at step {self.step_n} {str(original_e)}")
        return
```

### 1.2 异常抛出点与step_n传递

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

**步骤序号传递关系表**：
| 代码位置 | 变量 | 值（步骤1失败时） | 说明 |
|---------|------|------------------|------|
| base.py:169 | `step_n = 0` | 0 | 初始化 |
| base.py:177 | `step_n += 1` | 1 | 第一个步骤执行前变为1 |
| base.py:199 | `step_n=step_n` | 1 | 异常对象中存储的是1-based序号 |
| worker.py:327 | `error_step = e.step_n + 1` | 2 | **显示时再次+1** |
| worker.py:354 | `step_n=e.step_n` | 1 | 传递给通知服务的是原始值 |

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
        c = 0  # 达到阈值后重置计数
    
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

## 3. 阈值触发与两条重置路径

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
        c = 0  # 达到阈值后重置计数
```

### 3.2 重置路径1：达到阈值后重置

**触发场景**：连续失败次数达到配置阈值

**代码位置**：`worker.py:355`
```python
if threshold > 0 and c >= threshold:
    if not watch.get('notification_muted'):
        await send_step_failure_notification(...)
    c = 0  # 无论是否发送通知，只要达到阈值就重置
```

**特点**：
- 无论通知是否因`notification_muted`被跳过，计数器都会重置为0
- 仅在`threshold > 0`且达到阈值时触发

### 3.3 重置路径2：抓取成功后重置

**触发场景**：任何一次完整的成功抓取（包括Browser Steps全部成功）

**代码位置1**：`worker.py:153-154`（抓取开始前）
```python
# Clear last errors
datastore.data['watching'][uuid]['browser_steps_last_error_step'] = None
```

**代码位置2**：`worker.py:416-417`（抓取成功后，else分支）
```python
if not watch.get('ignore_status_codes'):
    update_obj['consecutive_filter_failures'] = 0
```

**关键发现（代码证据：worker.py:416-417）**：
- **抓取开始前**先清除`browser_steps_last_error_step`错误步骤标记
- **抓取成功后**（无异常抛出时），根据`ignore_status_codes`开关决定是否重置计数
  - ✅ 默认情况 `ignore_status_codes = False` → **重置**计数 = 0
  - ❌ 启用忽略 `ignore_status_codes = True` → **不重置**计数

---

## 4. 失败/成功/再次失败对照说明

### 4.1 场景对照（阈值=3，默认`ignore_status_codes=False`）

| 次数 | 场景 | 计数值 | 操作 | 说明 |
|-----|------|--------|------|------|
| 1 | 第一次失败 | 1 | 计数+1，不通知 | 未达阈值 |
| 2 | 第二次失败 | 2 | 计数+1，不通知 | 未达阈值 |
| 3 | 第三次失败 | 3 | 达到阈值→发送通知→重置计数=0 | 阈值触发重置 |
| 4 | （中间某一次）成功抓取 | 0 | 成功路径重置计数=0 | 仅默认情况生效 |
| 5 | 再次失败（成功后第一次） | 1 | 计数+1，不通知 | 重新开始累计 |

**注意**：以上为默认行为。若启用 `ignore_status_codes=True`，第4次成功抓取**不会重置**计数，失败累计将继续。

### 4.2 状态流转图

```
初始状态 (count=0)
    ↓ [第一次失败]
count=1 → 不通知
    ↓ [第二次失败]
count=2 → 不通知
    ↓ [第三次失败]
count=3 → 达到阈值 → 发送通知 → count=0
    ↓
重置状态 (count=0)
    ↓ [抓取成功]
count保持0 → 清除browser_steps_last_error_step
    ↓ [再次失败]
count=1 → 重新开始累计
```

### 4.3 对告警节奏的影响

**两条重置路径共同作用的效果（默认配置）**：

| 影响 | 说明 |
|------|------|
| **防抖动** | 间歇性故障恢复后，计数器重置，不会产生多余告警 |
| **告警间隔** | 两次告警之间至少需要`threshold`次连续失败 |
| **成功恢复** | 只要有一次HTTP 200的成功抓取，之前的失败累计就全部清零 |
| **状态码忽略影响** | 启用`ignore_status_codes=True`时，成功抓取不重置计数，可能更快触发告警 |

### 4.4 ignore_status_codes 影响总结

| `ignore_status_codes` 配置 | HTTP状态 | Browser Steps结果 | 计数重置行为 |
|----------------------------|----------|-------------------|-------------|
| **False（默认）** | 200 OK | 全部成功 | ✅ 重置计数 = 0 |
| **False（默认）** | 非200 | 异常抛出（失败） | 不进入else分支，不重置 |
| **True** | 任意状态码 | 全部成功 | ❌ **不重置**，计数保持累加 |

**代码证据**（worker.py:416-417）：
```python
# 只有未启用忽略状态码时才重置失败计数
if not watch.get('ignore_status_codes'):
    update_obj['consecutive_filter_failures'] = 0
```

---

## 5. 错误信息处理流程

### 5.1 原始错误信息提取

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

### 5.2 错误信息持久化字段

| 字段 | 值来源 | 示例值 | 说明 |
|------|--------|--------|------|
| `last_error` | 构造的`err_text` | "Browser step at position 2 could not run... Could not find the target." | 用户可见的完整错误提示 |
| `browser_steps_last_error_step` | `e.step_n + 1` | 2 | 高亮标记的步骤序号（加1后） |

**代码证据**（worker.py:342-344）：
```python
datastore.update_watch(uuid=uuid,
                     update_obj={'last_error': err_text,
                               'browser_steps_last_error_step': error_step})
```

---

## 6. 通知内容来源与构建

### 6.1 通知标题与正文

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

### 6.2 通知内容字段来源表

| 通知字段 | 数据来源 | 说明 |
|---------|----------|------|
| **`notification_title`** | 硬编码f-string | 包含`{step}`变量（step_n+1后的值） |
| **`notification_body`** | 硬编码多行f-string | 包含`{step}`、`{threshold}`变量 |
| **`notification_format`** | `_check_cascading_vars()` | 级联获取：Watch → Tag → Global |
| **`notification_urls`** | 级联获取 | 优先`watch['notification_urls']`，否则全局配置 |
| **`watch_url`** | `watch['url']` | 监控页面URL |
| **`uuid`/`watch_uuid`** | 传入参数 | 监控项唯一标识 |

### 6.3 通知调用链

```
worker.py:354 → send_step_failure_notification(watch_uuid, step_n=e.step_n, ...)
    ↓ (worker.py:782-792)
notification_service.send_step_failure_notification(watch_uuid, step_n)  # step_n=1
    ↓ (notification_service.py:489)
step = step_n + 1 → step=2
    ↓
构建 NotificationContextData → 放入 notification_q 队列
```

---

## 7. 步骤序号偏移问题分析

### 7.1 各环节口径对齐表

| 环节 | 实际 step_n | 预期口径 | 代码证据 | 说明 |
|------|------------|---------|---------|------|
| 异常对象 | 1-based | 1-based | base.py:169-199 | 循环中 step_n +=1，第一个步骤失败时 step_n = 1 |
| 通知服务内部 | step_n + 1 | 1-based（用户可见） | notification_service.py:489 + 单测68行 | **设计意图**：输入0-based → 输出1-based 展示给用户 |
| 前端 CSS 高亮 | browser_steps_last_error_step | 1-based | browser-steps.js:455 | CSS nth-child 是 1-based |
| 前端截图判断 | browser_steps_last_error_step | 1-based（i+1） | browser-steps.js:368 | 数组索引 i 是 0-based |
| 截图文件名 | step_n | 1-based | base.py:179,194 | step_1.jpeg，正确 |

### 7.2 加1操作分布与Bug诊断

| 位置 | 代码 | 输入值 | 输出值 | 预期行为 | 是否Bug | 说明 |
|------|------|--------|--------|---------|--------|------|
| **worker.py:327** | `error_step = e.step_n + 1` | 1 | 2 | 直接使用 1-based | **是Bug** | 异常对象已是1-based，无需再加1 |
| **worker.py:354** | `step_n=e.step_n` | 1 | 1 | 传递 0-based 给通知服务 | **是Bug** | 通知服务期望0-based输入 |
| **notification_service.py:489** | `step = step_n + 1` | 1 | 2 | 0-based → 1-based 转换 | **不是Bug** | 这是设计意图，单测验证了此行为 |

**单测证据**（test_step_failure_notification.py:63-68）：
```python
service.send_step_failure_notification(watch_uuid=watch_uuid, step_n=1)
assert 'position 2' in item['notification_title']  # 输入1 → 显示2
```
→ 证明通知服务内部 `+1` 是设计意图，不是Bug。

### 7.3 完整Bug证据链

```
base.py:169: step_n = 0
base.py:177: step_n += 1 → step_n = 1  ✓ 正确（1-based）
base.py:199: raise BrowserStepsStepException(step_n=1, ...)
↓
worker.py:327: error_step = e.step_n + 1 → error_step = 2  ← Bug 1：多余+1
worker.py:344: browser_steps_last_error_step = 2  ← 保存错误值
↓
worker.py:354: send_step_failure_notification(step_n=1)  ← Bug 2：应传0-based
notification_service.py:489: step = 1 + 1 → step = 2  ← 设计意图，但输入错
```

### 7.4 影响评估

| 影响点 | 严重程度 | 说明 |
|--------|---------|------|
| 前端高亮错误 | 中 | `browser_steps_last_error_step=2` 导致步骤1失败时高亮步骤2 |
| 通知序号偏移 | 中 | 通知邮件显示"position 2"，实际是步骤1失败 |
| 截图判断错误 | 低 | 错误的截图类型判断（before/after），但文件名本身正确 |
| 截图文件名 | 无 | `step_1.jpeg` 直接用 base.py 中的 step_n，正确 |

---

## 8. 其他设计问题

### 8.1 Exception父类未初始化

**问题**：
```python
class BrowserStepsStepException(Exception):
    def __init__(self, step_n, original_e):
        self.step_n = step_n
        self.original_e = original_e
        # 缺少 super().__init__(f"Step {step_n} failed: {original_e}")
        logger.debug(...)
        return
```

**影响**：`str(exception)`可能返回空字符串或非预期结果，不利于日志调试。

---

## 9. 附录：关键文件与行号索引

| 文件路径 | 关键代码行号 | 说明 |
|---------|-------------|------|
| `changedetectionio/content_fetchers/exceptions/__init__.py` | 45-50 | BrowserStepsStepException定义 |
| `changedetectionio/content_fetchers/base.py` | 165-199 | iterate_browser_steps异常抛出 |
| `changedetectionio/worker.py` | 153-154 | 抓取开始前清除错误步骤标记 |
| `changedetectionio/worker.py` | 323-359 | Browser Steps异常处理完整逻辑 |
| `changedetectionio/worker.py` | 346-357 | 连续失败计数与阈值判断 |
| `changedetectionio/worker.py` | 416-417 | 成功路径重置计数字段 |
| `changedetectionio/worker.py` | 782-792 | send_step_failure_notification包装函数 |
| `changedetectionio/notification_service.py` | 480-524 | 通知内容构建 |

---

## 10. 修复建议

| 问题 | 修复方案 |
|------|---------|
| 步骤序号偏移 | 移除worker.py:327和notification_service.py:489中的`+ 1`操作，确保异常中存储的step_n直接使用 |
| Exception未初始化 | 添加`super().__init__(f"Browser step {step_n} failed: {str(original_e)}")` |
| 硬编码通知模板 | 将通知标题和正文移至外部Jinja2模板文件，支持国际化 |
