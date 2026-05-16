# Browser Steps 执行链路与失败处理分析报告

## 1. 整体架构概述

Browser Steps 是 changedetection.io 中的浏览器自动化功能，允许用户配置一系列操作步骤（如点击、输入文本、等待等），在页面抓取前执行这些步骤。整个流程分为：

1. **前端编辑**：用户通过可视化界面配置步骤
2. **数据序列化**：步骤配置保存为 JSON 格式
3. **抓取执行**：在实际抓取时按顺序执行每个步骤
4. **失败处理**：单步失败时的异常捕获与处理
5. **失败通知**：向用户发送失败告警

---

## 2. 前端编辑流程与数据结构

### 2.1 前端编辑界面

**文件位置**：`changedetectionio/static/js/browser-steps.js`

前端采用可视化选择器方式配置步骤：
- 用户点击页面元素，系统自动识别选择器
- 支持多种操作类型（点击、输入文本、等待等）
- 每个步骤包含三个核心字段：`operation`、`selector`、`optional_value`

### 2.2 步骤配置数据结构

每个浏览器步骤在前端是一个对象，结构如下：

```javascript
{
  "operation": "Click element",      // 操作类型
  "selector": "#submit-button",      // CSS/XPath 选择器
  "optional_value": ""               // 可选值（如输入文本、等待时间）
}
```

### 2.3 支持的操作类型

在 `browser_steps/browser_steps.py:26-59` 中定义：

| 操作类型 | 说明 | 是否需要 selector | 是否需要 optional_value |
|---------|------|------------------|------------------------|
| Click element | 点击元素 | 是 | 否 |
| Click element if exists | 点击元素（如存在） | 是 | 否 |
| Enter text in field | 在输入框输入文本 | 是 | 是 |
| Execute JS | 执行 JavaScript | 否 | 是 |
| Goto URL | 跳转到 URL | 否 | 是 |
| Wait for seconds | 等待指定秒数 | 否 | 是 |
| Wait for text | 等待文本出现 | 否 | 是 |
| Check checkbox | 勾选复选框 | 是 | 否 |
| 更多操作... | | | |

---

## 3. 步骤配置序列化过程

### 3.1 表单提交

在编辑页面 `blueprint/ui/templates/edit.html` 中，浏览器步骤通过表单字段提交：
- 表单字段命名：`steps-0-operation`, `steps-1-selector` 等
- 使用 WTForms 动态表单处理

### 3.2 后端接收与验证

**文件位置**：`changedetectionio/forms.py`

表单验证后，步骤数据被转换为字典列表格式：

```python
browser_steps = [
    {
        'operation': 'Click element',
        'selector': '#button',
        'optional_value': ''
    },
    # ... 更多步骤
]
```

### 3.3 数据持久化

**文件位置**：`changedetectionio/model/Watch.py`

步骤配置作为 Watch 对象的一个字段存储：
- 保存为 JSON 格式到数据目录的 `watch.json` 文件
- 通过 `EntityPersistenceMixin` 处理持久化

---

## 4. 抓取执行流程

### 4.1 执行入口

**文件位置**：`changedetectionio/content_fetchers/playwright.py:368-375`

在 `run()` 方法中，页面加载完成后调用 `iterate_browser_steps()` 执行所有步骤：

```python
if self.browser_steps:
    try:
        await self.iterate_browser_steps(start_url=url)
    except BrowserStepsStepException:
        # 异常向上抛出，由上层处理
        raise
```

### 4.2 步骤迭代执行

**文件位置**：`changedetectionio/content_fetchers/base.py:165-199`

`iterate_browser_steps()` 是核心执行函数：

```python
async def iterate_browser_steps(self, start_url=None):
    from changedetectionio.browser_steps.browser_steps import steppable_browser_interface, browser_steps_get_valid_steps
    
    interface = steppable_browser_interface(start_url=start_url)
    interface.page = self.page
    valid_steps = browser_steps_get_valid_steps(self.browser_steps)

    for step in valid_steps:
        step_n += 1  # 步骤序号（从1开始）
        
        # 执行前保存截图和HTML
        await self.screenshot_step("before-" + str(step_n))
        await self.save_step_html("before-" + str(step_n))

        try:
            # 支持 Jinja2 模板变量替换
            optional_value = step['optional_value']
            selector = step['selector']
            if '{%' in optional_value or '{{' in optional_value:
                optional_value = jinja_render(template_str=optional_value)
            
            # 调用具体操作
            await interface.call_action(
                action_name=step['operation'],
                selector=selector,
                optional_value=optional_value
            )
            
            # 执行后保存截图和HTML
            await self.screenshot_step(step_n)
            await self.save_step_html(step_n)
            
        except (Error, TimeoutError) as e:
            # 捕获 Playwright 异常，包装后抛出
            raise BrowserStepsStepException(step_n=step_n, original_e=e)
```

### 4.3 步骤执行器

**文件位置**：`changedetectionio/browser_steps/browser_steps.py:75-132`

`call_action()` 方法负责调度具体操作：

```python
async def call_action(self, action_name, selector=None, optional_value=None):
    call_action_name = re.sub('[^0-9a-zA-Z]+', '_', action_name.lower())
    
    # 支持 Jinja2 模板变量
    if selector and ('{%' in selector or '{{' in selector):
        selector = jinja_render(template_str=selector)
    
    # 动态调用对应的 action_* 方法
    action_handler = getattr(self, "action_" + call_action_name)
    await action_handler(selector, optional_value)
```

---

## 5. 单步失败处理逻辑

### 5.1 异常定义

**文件位置**：`changedetectionio/content_fetchers/exceptions/__init__.py`

```python
class BrowserStepsStepException(Exception):
    def __init__(self, step_n, original_e):
        self.step_n = step_n  # 失败的步骤序号（从0开始）
        self.original_e = original_e  # 原始异常
        super().__init__(f"Browser step {step_n + 1} failed: {str(original_e)}")
```

### 5.2 异常捕获点

**文件位置**：`changedetectionio/worker.py:323-354`

在 worker 的抓取流程中捕获步骤异常：

```python
try:
    await site_changed_check(...)
except content_fetchers_exceptions.BrowserStepsStepException as e:
    # 步骤执行失败
    error_step = e.step_n + 1  # 转换为从1开始的显示序号
    
    # 更新 Watch 的错误状态
    datastore.update_watch_error(
        uuid=uuid,
        error_text=f"Browser Step #{error_step} failed: {str(e.original_e)}"
    )
    
    # 增加连续失败计数
    consecutive_browser_step_errors = watch.get('consecutive_browser_step_errors', 0) + 1
    
    # 达到阈值时发送通知
    threshold = int(datastore.data['settings']['application'].get('filter_failure_notification_threshold_attempts', 0))
    if consecutive_browser_step_errors >= threshold:
        await send_step_failure_notification(
            watch_uuid=uuid, 
            step_n=e.step_n, 
            notification_q=notification_q, 
            datastore=datastore
        )
        consecutive_browser_step_errors = 0  # 重置计数
    
    # 保存连续失败计数
    datastore.update_watch(
        uuid=uuid, 
        update_obj={'consecutive_browser_step_errors': consecutive_browser_step_errors}
    )
```

### 5.3 关键设计要点

1. **连续失败计数**：使用 `consecutive_browser_step_errors` 字段记录连续失败次数
2. **阈值触发**：只有当连续失败达到配置阈值时才发送通知，避免频繁告警
3. **自动重置**：发送通知后重置计数器，避免重复发送相同告警
4. **错误信息持久化**：通过 `update_watch_error` 保存错误信息供用户查看

---

## 6. 失败通知内容来源

### 6.1 通知服务

**文件位置**：`changedetectionio/notification_service.py:480-524`

`send_step_failure_notification()` 方法构建通知内容：

```python
def send_step_failure_notification(self, watch_uuid, step_n):
    watch = self.datastore.data['watching'].get(watch_uuid, False)
    threshold = self.datastore.data['settings']['application'].get('filter_failure_notification_threshold_attempts')
    
    step = step_n + 1  # 显示序号从1开始
    
    # 通知标题
    notification_title = f"Changedetection.io - Alert - Browser step at position {step} could not be run"
    
    # 通知正文（硬编码模板）
    body = f"""Hello,

Your configured browser step at position {step} for the web page watch {{{{watch_url}}}} did not appear on the page after {threshold} attempts, did the page change layout?

The element may have moved and needs editing, or does it need a delay added?

Edit link: {{{{base_url}}}}/edit/{{{{watch_uuid}}}}

Thanks - Your omniscient changedetection.io installation.
"""

    # 构建通知对象
    n_object = NotificationContextData({
        'notification_title': notification_title,
        'notification_body': body,
        'notification_format': _check_cascading_vars(self.datastore, 'notification_format', watch),
    })
    
    # 获取通知接收地址（级联优先级）
    if len(watch['notification_urls']):
        n_object['notification_urls'] = watch['notification_urls']
    elif len(self.datastore.data['settings']['application']['notification_urls']):
        n_object['notification_urls'] = self.datastore.data['settings']['application']['notification_urls']
    
    # 添加额外变量
    n_object.update({
        'watch_url': watch['url'],
        'uuid': watch_uuid
    })
    
    # 放入通知队列
    self.notification_q.put(n_object)
```

### 6.2 通知内容构成

| 字段 | 来源 | 说明 |
|------|------|------|
| **notification_title** | 硬编码 | 包含步骤位置的固定文本 |
| **notification_body** | 硬编码模板 | 包含步骤位置、阈值、编辑链接等变量 |
| **notification_format** | 级联配置 | Watch → Tag → Global |
| **notification_urls** | 级联配置 | Watch 配置优先，否则使用全局配置 |
| **watch_url** | Watch 对象 | 被监控的页面URL |
| **watch_uuid** | Watch 对象 | 监控项唯一标识 |

### 6.3 级联变量解析机制

**文件位置**：`changedetectionio/notification_service.py:17-54`

`_check_cascading_vars()` 实现配置优先级：

```
Individual Watch Settings → Tag Settings → Global Settings
```

这意味着：
1. 首先检查 Watch 自身的配置
2. 如果 Watch 没有配置，检查其所属的 Tag
3. 如果 Tag 也没有配置，使用全局设置
4. 最后还有默认值兜底

---

## 7. 关键调用链总结

### 7.1 正常执行流程

```
前端编辑步骤
    ↓
表单提交 → WTForms 验证
    ↓
Watch 对象保存 (JSON持久化)
    ↓
抓取任务触发
    ↓
Playwright Fetcher.run()
    ↓
iterate_browser_steps()
    ↓
call_action() → 动态调用 action_* 方法
    ↓
步骤执行成功 → 继续下一个步骤
    ↓
所有步骤完成 → 执行页面内容抓取
```

### 7.2 失败处理流程

```
步骤执行异常 (TimeoutError / Error)
    ↓
iterate_browser_steps() 捕获
    ↓
包装为 BrowserStepsStepException 抛出
    ↓
worker.py 捕获异常
    ↓
├─ 更新 Watch 错误信息
├─ 增加连续失败计数
└─ 达到阈值 → 调用 send_step_failure_notification()
        ↓
        notification_service 构建通知内容
        ↓
        通知放入队列 → Apprise 异步发送
```

---

## 8. 代码优化建议

### 8.1 通知模板硬编码问题

**当前问题**：通知标题和正文硬编码在 Python 代码中，不利于维护和国际化。

**建议**：
- 将通知模板移到独立的 Jinja2 模板文件中
- 支持多语言配置

### 8.2 步骤执行前后截图命名不一致

**当前问题**：
- 执行前：`screenshot_step("before-" + str(step_n))`
- 执行后：`screenshot_step(step_n)`

**建议**：统一命名规范，如 `step-1-before.jpg` 和 `step-1-after.jpg`

### 8.3 异常信息丰富度

**当前问题**：通知中只包含步骤位置，没有具体错误原因。

**建议**：在通知中包含原始异常信息，帮助用户快速定位问题：
```python
body = f"""Hello,

Step {step} failed with error: {str(e.original_e)}

...
"""
```

---

## 9. 附录：关键文件清单

| 文件路径 | 说明 |
|---------|------|
| `changedetectionio/browser_steps/browser_steps.py` | 步骤执行核心逻辑 |
| `changedetectionio/static/js/browser-steps.js` | 前端编辑交互逻辑 |
| `changedetectionio/blueprint/browser_steps/__init__.py` | 实时预览后端API |
| `changedetectionio/content_fetchers/base.py` | 抓取器基类，包含 iterate_browser_steps |
| `changedetectionio/content_fetchers/playwright.py` | Playwright 抓取实现 |
| `changedetectionio/notification_service.py` | 通知服务，包含失败通知构建 |
| `changedetectionio/worker.py` | 任务调度与异常处理 |
| `changedetectionio/model/Watch.py` | Watch 数据模型 |
