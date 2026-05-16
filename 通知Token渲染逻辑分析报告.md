# 通知消息 Token 渲染逻辑分析报告

## 1. 系统架构概览

通知系统采用分层架构设计，主要分为以下几个层次：

- **UI 层**：用户交互界面，提供通知配置和测试功能
- **API 层**：提供通知配置的 REST API 接口
- **通知服务层**：核心业务逻辑，处理通知队列、变量渲染、模板处理
- **内容抓取层**：获取网页内容，为通知渲染提供原始数据
- **Apprise 分发层**：通过 Apprise 库将消息发送到各个通知渠道

---

## 2. Token 渲染逻辑 - UI 层实现

### 2.1 核心文件
`changedetectionio/blueprint/ui/notification.py`

### 2.2 主要功能

UI 层主要提供 **测试通知发送** 的 AJAX 端点，实现以下功能：

#### 2.2.1 通知测试端点 (`/notification/send-test`)
- 支持全局设置、标签组设置、单个监控设置三种模式
- 验证 Apprise URL 的有效性
- 使用 `NotificationContextData` 生成随机占位符数据进行验证
- 调用 `jinja_render()` 进行 URL 模板渲染验证
- 最终调用 `process_notification()` 发送实际通知

#### 2.2.2 关键代码流程
```python
# 1. 创建通知上下文数据对象
generic_notification_context_data = NotificationContextData()
# 2. 填充随机验证数据
generic_notification_context_data.set_random_for_validation()
# 3. 渲染通知 URL 模板
n_url = jinja_render(template_str=n_url, **generic_notification_context_data).strip()
# 4. 验证 Apprise URL 有效性
if not apobj.add(n_url):
    return f'Error:  {n_url} is not a valid AppRise URL.'
# 5. 调用通知处理函数发送实际通知
sent_obj = process_notification(n_object, datastore)
```

#### 2.2.3 数据准备逻辑
- 当没有足够的历史快照时，使用硬编码的示例文本进行测试
- 当有历史数据时，使用真实的 `prev_snapshot` 和 `current_snapshot`
- 通过 `set_basic_notification_vars()` 函数设置基本通知变量

---

## 3. Token 渲染逻辑 - API 层实现

### 3.1 核心文件
`changedetectionio/api/Notifications.py`

### 3.2 REST API 端点

API 层提供标准的 CRUD 操作来管理通知 URL：

| 方法 | 端点 | 功能 |
|------|------|------|
| GET | `/api/v1/notifications` | 获取通知 URL 列表 |
| POST | `/api/v1/notifications` | 添加通知 URL |
| PUT | `/api/v1/notifications` | 替换通知 URL 列表 |
| DELETE | `/api/v1/notifications` | 删除通知 URL |

### 3.3 验证机制

API 层使用 `ValidateAppRiseServers` 表单验证器来验证通知 URL 的有效性：

```python
def validate_notification_urls(notification_urls):
    from changedetectionio.forms import ValidateAppRiseServers
    validator = ValidateAppRiseServers()
    # ... 执行验证
```

> **注意**：API 层主要负责通知配置管理，**不直接参与 Token 渲染**，Token 渲染主要在通知服务层完成。

---

## 4. 内容抓取层：为渲染提供原始数据

### 4.1 核心文件
- `changedetectionio/content_fetchers/base.py` - 抽象基类
- `changedetectionio/content_fetchers/requests.py` - 简单 HTTP 请求抓取器
- `changedetectionio/content_fetchers/playwright.py` - Playwright 浏览器渲染抓取器
- `changedetectionio/content_fetchers/puppeteer.py` - Puppeteer 抓取器

### 4.2 抓取器基类设计

`Fetcher` 抽象基类定义了所有抓取器的通用接口：

```python
class Fetcher():
    # 输出数据
    content = None           # 抓取到的页面内容（字符串）
    error = None             # 错误信息
    status_code = None       # HTTP 状态码
    headers = {}             # 响应头
    screenshot = None        # 截图二进制数据
    xpath_data = None        # XPath 元素数据（用于可视化选择器）
    
    # 能力标志
    supports_browser_steps = False      # 是否支持浏览器步骤
    supports_screenshots = False        # 是否支持截图
    supports_xpath_element_data = False # 是否支持 XPath 元素提取

    @abstractmethod
    async def run(self, url, ...):
        # 执行实际抓取，设置 self.content 等属性
        pass
```

### 4.3 内容抓取流程

1. **抓取器选择**：根据监控配置选择合适的抓取器（requests / playwright / puppeteer）
2. **执行抓取**：调用 `run()` 方法获取页面内容
3. **内容后处理**：应用 CSS/XPath 过滤器、文本替换规则等
4. **内容存储**：将处理后的内容保存为快照（snapshot）
5. **差异检测**：与前一版本快照比较，检测变化
6. **通知触发**：当检测到变化时，将快照数据传递给通知服务层

### 4.4 为 Token 渲染提供的数据

内容抓取层最终为通知渲染提供以下核心数据：

| 数据项 | 说明 | 对应 Token |
|--------|------|-----------|
| `current_snapshot` | 当前版本的页面内容快照 | `{{current_snapshot}}` |
| `prev_snapshot` | 前一版本的页面内容快照 | `{{prev_snapshot}}` |
| 差异计算结果 | 通过 `diff.render_diff()` 计算的差异 | `{{diff}}`, `{{diff_added}}`, 等 |
| `triggered_text` | 触发告警的匹配文本 | `{{triggered_text}}` |
| `screenshot` | 页面截图（二进制） | 作为邮件附件 |

---

## 5. 通知服务层：Token 渲染核心逻辑

### 5.1 核心文件
- `changedetectionio/notification_service.py` - 通知服务和上下文数据定义
- `changedetectionio/notification/handler.py` - 通知处理和 Apprise 集成

### 5.2 NotificationContextData：Token 数据容器

`NotificationContextData` 类是所有通知 Token 的数据容器，定义了所有可用的模板变量：

```python
class NotificationContextData(dict):
    def __init__(self, initial_data=None, **kwargs):
        super().__init__({
            # 基础信息
            'base_url': None,                    # 系统基础 URL
            'change_datetime': FormattableTimestamp(...),  # 变化时间（可格式化）
            'uuid': '...',                       # 监控 UUID
            'watch_url': 'https://...',          # 监控的 URL
            'watch_uuid': '...',                 # 监控 UUID（重复）
            'watch_title': None,                 # 监控标题
            'watch_tag': None,                   # 监控标签
            'watch_mime_type': None,             # 内容 MIME 类型
            
            # 快照数据
            'current_snapshot': None,            # 当前快照内容
            'prev_snapshot': None,               # 前一快照内容
            'triggered_text': None,              # 触发的文本
            
            # 差异数据（多种格式）
            'diff': FormattableDiff('', ''),     # 标准差异
            'diff_clean': FormattableDiff(...), # 无前缀差异
            'diff_added': FormattableDiff(...),  # 仅显示新增
            'diff_removed': FormattableDiff(...),# 仅显示删除
            'diff_full': FormattableDiff(...),   # 完整上下文
            'diff_patch': FormattableDiff(...),  # Patch 格式
            'diff_changed_from': FormattableExtract(...),  # 变化前的值
            'diff_changed_to': FormattableExtract(...),    # 变化后的值
            
            # URL 链接
            'diff_url': None,                    # 差异页面链接
            'preview_url': None,                 # 预览页面链接
            
            # AI 相关
            'llm_summary': None,                 # AI 生成的变更摘要
            'llm_intent': None,                  # AI 评估意图
            
            # 其他
            'screenshot': None,                  # 截图
            'notification_timestamp': time.time(), # 通知时间戳
        })
```

### 5.3 可格式化数据类型

#### 5.3.1 FormattableTimestamp
支持在 Jinja2 模板中自定义时间格式：
```jinja2
{{ change_datetime }}                      # 默认格式
{{ change_datetime(format='%Y-%m-%d') }}   # 仅日期
{{ change_datetime(format='%H:%M:%S') }}   # 仅时间
```

#### 5.3.2 FormattableDiff
支持在模板中动态调整差异显示方式：
```jinja2
{{ diff }}                                  # 默认显示
{{ diff(lines=5) }}                         # 仅显示前5行
{{ diff(added_only=true) }}                 # 仅显示新增
{{ diff(removed_only=true) }}               # 仅显示删除
{{ diff(context=3) }}                       # 显示3行上下文
{{ diff(word_diff=false) }}                 # 行级差异而非词级
```

#### 5.3.3 FormattableExtract
提取具体的变更值（用于价格监控等场景）：
```jinja2
{{ diff_changed_from }}   # 变化前的值，如 "$99.99"
{{ diff_changed_to }}     # 变化后的值，如 "$109.99"
```

### 5.4 延迟渲染优化

`add_rendered_diff_to_notification_vars()` 函数实现了**按需渲染**优化：

```python
def add_rendered_diff_to_notification_vars(notification_scan_text, prev_snapshot, current_snapshot, word_diff):
    # 1. 扫描通知模板中实际使用了哪些 diff Token
    # 2. 只渲染实际用到的 diff 变体，避免不必要的计算
    # 3. 返回只包含已渲染 diff 的字典
```

**优化策略**：
- 扫描 `notification_title` 和 `notification_body` 中的 Token 使用
- 使用正则表达式匹配 `{{diff}}`, `{{diff_added}}` 等模式
- 只渲染匹配到的 diff 变体
- 显著提升性能（避免渲染 8 种未使用的 diff 格式）

### 5.5 级联变量优先级

通知配置采用三级级联优先级：

```python
def _check_cascading_vars(datastore, var_name, watch):
    # 优先级：单个监控设置 > 标签设置 > 全局设置
    v = watch.get(var_name)
    if v and not watch.get('notification_muted'):
        return v
    
    tags = datastore.get_all_tags_for_watch(uuid=watch.get('uuid'))
    for tag_uuid, tag in tags.items():
        v = tag.get(var_name)
        if v and not tag.get('notification_muted'):
            return v
    
    if datastore.data['settings']['application'].get(var_name):
        return datastore.data['settings']['application'].get(var_name)
    
    return default_value
```

**优先级顺序**：
1. **监控级别**（最高）：单个监控的通知配置
2. **标签级别**：监控所属标签的通知配置
3. **系统级别**（最低）：全局默认通知配置

---

## 6. Jinja2 安全渲染引擎

### 6.1 核心文件
`changedetectionio/jinja2_custom/safe_jinja.py`

### 6.2 安全沙箱环境

使用 `jinja2.sandbox.ImmutableSandboxedEnvironment` 创建安全的渲染环境：

```python
def create_jinja_env(extensions=None, **kwargs):
    jinja2_env = jinja2.sandbox.ImmutableSandboxedEnvironment(
        extensions=[TimeExtension],
        **kwargs
    )
    # 注册自定义过滤器
    jinja2_env.filters['regex_replace'] = regex_replace
    return jinja2_env
```

### 6.3 安全特性

1. **不可变沙箱**：防止模板修改系统状态
2. **负载大小限制**：`JINJA2_MAX_RETURN_PAYLOAD_SIZE = 10MB`（可配置）
3. **自定义扩展白名单**：仅允许 `TimeExtension` 等已注册扩展
4. **过滤器白名单**：仅允许 `regex_replace` 等安全过滤器
5. **HTML 转义**：在 HTML 格式通知中自动转义变量内容

### 6.4 HTML 安全防护

针对 HTML 格式通知，实施额外的 XSS 防护：

```python
# 在 process_notification() 中
if 'html' in requested_output_format:
    from markupsafe import escape as html_escape
    for key in [k for k in notification_parameters if k.startswith('diff') or k in _page_content_keys]:
        if notification_parameters.get(key):
            notification_parameters[key] = str(html_escape(str(notification_parameters[key])))
```

**防护范围**：
- 所有 `diff_*` 变量
- `raw_diff`, `current_snapshot`, `prev_snapshot`, `triggered_text`
- 防止被监控页面注入恶意 HTML/JavaScript

---

## 7. Apprise 消息分发流程

### 7.1 核心处理函数

`process_notification()` 函数是通知分发的核心入口：

```python
def process_notification(n_object: NotificationContextData, datastore):
    # 步骤 1: 创建通知参数
    notification_parameters = create_notification_parameters(n_object, datastore)
    
    # 步骤 2: 确定输出格式
    requested_output_format = n_object.get('notification_format', default_notification_format)
    
    # 步骤 3: 按需渲染 diff 变量
    n_object.update(add_rendered_diff_to_notification_vars(
        notification_scan_text=n_object.get('notification_body', '') + n_object.get('notification_title', ''),
        current_snapshot=n_object.get('current_snapshot'),
        prev_snapshot=n_object.get('prev_snapshot'),
        word_diff=False if requested_output_format_original == 'text' else True,
    ))
    
    # 步骤 4: AI 摘要处理
    _llm_change_summary = (n_object.get('_llm_change_summary') or '').strip()
    if _llm_change_summary and _override_diff:
        n_object['diff'] = _llm_change_summary
    
    # 步骤 5: HTML 安全转义（如需要）
    if 'html' in requested_output_format:
        # ... 执行 HTML 转义
    
    # 步骤 6: 遍历每个通知 URL
    for url in n_object['notification_urls']:
        # 6.1 渲染标题和正文模板
        n_body = jinja_render(template_str=n_object.get('notification_body', ''), **notification_parameters)
        n_title = jinja_render(template_str=n_object.get('notification_title', ''), **notification_parameters)
        
        # 6.2 渲染通知 URL 本身（支持 Token）
        url = jinja_render(template_str=url, **notification_parameters)
        
        # 6.3 应用服务特定的格式调整
        (url, n_body, n_title) = apply_service_tweaks(url, n_body, n_title, requested_output_format)
        
        # 6.4 添加到 Apprise
        if not url.startswith('null://'):
            apobj.add(url)
    
    # 步骤 7: 实际发送通知
    if not url.startswith('null://'):
        apobj.notify(
            title=n_title,
            body=n_body,
            body_format=apprise_input_format,
            attach=n_object.get('screenshot', None)
        )
```

### 7.2 服务特定格式调整 (`apply_service_tweaks`)

针对不同的通知渠道，应用不同的格式优化：

| 服务类型 | 优化措施 |
|---------|---------|
| **Telegram** | - 将 `<br>` 替换为 `\n`<br>- 限制内容长度<br>- 使用 HTML 标签标记差异 |
| **Discord** | - 自定义插件使用 Embed 富消息<br>- 将占位符转换为彩色 Embed<br>- 特殊的 Markdown 处理 |
| **HTML 邮件** | - 添加 CSS 样式标记差异<br>- 保持空格格式<br>- 转换换行符为 `<br>` |
| **纯文本** | - 使用 `(added)` / `(removed)` 文字标记 |
| **Markdown** | - 使用 `**` 标记新增<br>- 使用 `~~` 标记删除 |

### 7.3 差异占位符系统

使用统一的占位符系统，然后根据输出格式替换：

```python
# 占位符定义
REMOVED_PLACEMARKER_OPEN / CLOSED
ADDED_PLACEMARKER_OPEN / CLOSED
CHANGED_PLACEMARKER_OPEN / CLOSED
CHANGED_INTO_PLACEMARKER_OPEN / CLOSED

# 根据格式替换
if format == 'htmlcolor':
    text = text.replace(REMOVED_PLACEMARKER_OPEN, '<span style="color:red">')
elif format == 'markdown':
    text = text.replace(REMOVED_PLACEMARKER_OPEN, '~~')
elif format == 'telegram':
    text = text.replace(REMOVED_PLACEMARKER_OPEN, '<s>')
else:
    text = text.replace(REMOVED_PLACEMARKER_OPEN, '(removed) ')
```

### 7.4 自定义 Apprise 插件

项目扩展了自定义的 Apprise 插件：

1. **Discord 自定义插件** (`notification/apprise_plugin/discord.py`)
   - 支持使用 Discord Embed 富消息格式
   - 为新增/删除内容使用不同颜色的侧边栏
   - 突破纯文本 2000 字符限制

2. **HTTP 自定义处理器** (`notification/apprise_plugin/custom_handlers.py`)
   - 处理自定义 Webhook 请求
   - 支持自定义 HTTP 头和请求体

3. **自定义资源配置** (`notification/apprise_plugin/assets.py`)
   - 自定义通知图标
   - 自定义应用名称和配置

---

## 8. 完整数据流总结

### 8.1 端到端流程

```
┌─────────────────┐
│  内容抓取层     │  1. 抓取页面内容 (Fetcher.run())
│  (Playwright)   │  2. 处理/过滤内容
└────────┬────────┘
         │
         ▼  current_snapshot, prev_snapshot
┌─────────────────┐
│  差异检测引擎   │  3. 计算差异 (diff.render_diff())
└────────┬────────┘
         │
         ▼  触发通知事件
┌─────────────────┐
│  Notification   │  4. 构建 NotificationContextData
│    Service      │  5. 级联配置获取 (监控→标签→全局)
└────────┬────────┘  6. 级联配置获取 (监控→标签→全局)  6. add_rendered_diff_to_notification_vars()
         │
         ▼  notification_parameters
┌─────────────────┐
│  Jinja2 渲染    │  7. jinja_render(title_template)
│    引擎         │  8. jinja_render(body_template)
└────────┬────────┘  9. jinja_render(notification_url)
         │
         ▼  n_title, n_body, rendered_url
┌─────────────────┐
│  格式适配层     │  10. apply_service_tweaks()
│  (Handler)      │  11. 占位符替换为目标格式
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Apprise      │  12. apobj.add(url)
│   分发层        │  13. apobj.notify(title, body)
└─────────────────┘
```

### 8.2 关键设计决策总结

| 决策 | 优点 | 代价 |
|-----|------|------|
| **沙箱化 Jinja2** | 防止模板注入攻击，限制恶意代码执行 | 轻微性能开销 |
| **按需渲染 diff** | 仅渲染模板中用到的 diff 变体，提升性能 | 增加了代码复杂度 |
| **三级级联配置** | 灵活的通知策略，支持全局/标签/个体配置 | 配置查找逻辑复杂 |
| **统一占位符系统** | 一次 diff 渲染，适配多种输出格式 | 需要维护多套替换逻辑 |
| **内容抓取抽象** | 支持多种抓取后端（requests/playwright/puppeteer） | 接口设计复杂 |
| **异步队列处理** | 不阻塞主检测流程，提升吞吐量 | 需要维护队列状态 |

---

## 9. 核心类和函数索引

### 通知服务层
- `NotificationService` - 通知服务主类
- `NotificationContextData` - 通知 Token 数据容器
- `FormattableTimestamp` - 可格式化时间戳类型
- `FormattableDiff` - 可格式化差异类型
- `FormattableExtract` - 可格式化变更提取类型
- `add_rendered_diff_to_notification_vars()` - 按需渲染 diff 变量
- `create_notification_parameters()` - 创建通知参数字典
- `set_basic_notification_vars()` - 设置基本通知变量

### 通知处理层
- `process_notification()` - 通知处理主函数
- `apply_service_tweaks()` - 应用服务特定格式调整
- `replace_placemarkers_in_text()` - 替换差异占位符
- `notification_format_align_with_apprise()` - 格式对齐转换

### Jinja2 渲染层
- `render()` - 安全的 Jinja2 渲染函数
- `create_jinja_env()` - 创建沙箱化 Jinja2 环境

### 内容抓取层
- `Fetcher` - 抓取器抽象基类
- `Requests` - 简单 HTTP 抓取器
- `Playwright` - 浏览器渲染抓取器

### API/UI 层
- `Notifications` (API) - 通知配置 REST API
- `ajax_callback_send_notification_test()` (UI) - 测试通知发送端点
