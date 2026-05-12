# ChangeDetection.io 从抓取到通知的完整流程

## 目录
1. [整体架构](#1-整体架构)
2. [处理器分流机制](#2-处理器分流机制)
3. [处理器详细对比](#3-处理器详细对比)
4. [通知模板处理](#4-通知模板处理)
5. [底层发送通道](#5-底层发送通道)
6. [错误处理与重试](#6-错误处理与重试)
7. [完整流程图](#7-完整流程图)

---

## 1. 整体架构

### 1.1 核心模块

系统采用模块化设计，主要包含以下核心模块：

| 模块 | 职责 | 主要文件 |
|------|------|----------|
| **内容抓取器** (Content Fetchers) | 负责通过不同方式获取网页内容 | `content_fetchers/requests.py`, `playwright.py`, `puppeteer.py` |
| **处理器** (Processors) | 对抓取的内容进行差异检测 | `processors/text_json_diff/`, `restock_diff/`, `image_ssim_diff/` |
| **通知服务** (Notification Service) | 管理通知的模板渲染和队列 | `notification_service.py` |
| **通知处理器** (Notification Handler) | 处理通知的具体发送 | `notification/handler.py` |
| **异步工作器** (Async Worker) | 驱动整个检查流程 | `worker.py` |

### 1.2 流程概述

```
网页/内容源
    ↓
[内容抓取器] ← 根据配置选择 (requests/playwright/puppeteer)
    ↓
[处理器分流] ← 根据 watch.processor 字段选择
    ↓
[差异检测] ← 不同处理器产生不同的差异输出格式
    ↓
[变更决策] ← 判断是否需要通知
    ↓
[通知服务] ← 级联优先级: Watch > Tag > Global
    ↓
[模板渲染] ← Jinja2 模板 + 上下文变量
    ↓
[通知队列] ← NotificationQueue
    ↓
[通知处理器] ← Apprise 集成
    ↓
[发送通道] ← 邮件/Slack/Discord/Telegram/Webhook 等
```

---

## 2. 处理器分流机制

### 2.1 处理器发现机制

处理器的发现和注册通过 `changedetectionio/processors/__init__.py` 中的 `find_processors()` 函数实现：

```python
# 1. 扫描内置处理器包
sub_packages = find_sub_packages("changedetectionio.processors")
# 2. 加载每个子包中的 processor.py
for sub_package in sub_packages:
    module = importlib.import_module(f"{package_name}.{sub_package}.processor")
# 3. 识别继承自 difference_detection_processor 的类
# 4. 通过 pluggy 加载插件处理器
plugin_results = plugin_manager.hook.register_processor()
```

### 2.2 处理器选择优先级

在 `worker.py` 中，处理器的选择逻辑：

```python
processor = watch.get('processor', 'text_json_diff')  # 默认 text_json_diff
processor_module = get_processor_module(processor)
update_handler = processor_module.perform_site_check(datastore=datastore, watch_uuid=uuid)
```

处理器的权重排序（`processor_weight`）：
- `text_json_diff`: -100 (默认，最高优先级)
- `restock_diff`: 1
- `image_ssim_diff`: 2

### 2.3 环境变量控制

可通过 `DISABLED_PROCESSORS` 环境变量禁用特定处理器：
```bash
DISABLED_PROCESSORS=image_ssim_diff,restock_diff
```

---

## 3. 处理器详细对比

### 3.1 Text/JSON/HTML 处理器 (`text_json_diff`)

**位置**: `processors/text_json_diff/processor.py`

#### 功能特点
- 支持 HTML、JSON、PDF、RSS 等多种格式
- 支持 CSS 选择器、XPath、JSONPath 过滤
- 丰富的文本转换和规则引擎

#### 输出格式
- **快照内容**: 纯文本 (UTF-8 字符串)
- **差异计算**: 基于 MD5 校验和
- **触发条件**: `watch.previous_md5 != fetched_md5`

#### 处理流程
```
┌─────────────────────────────────────────────────────────┐
│ 内容预处理                                               │
│  ├── RSS: CDATA 解析 / 格式化 RSS items                  │
│  ├── PDF: pdftohtml 转换为 HTML                         │
│  ├── JSON: 排序并格式化                                  │
│  └── HTML: 混淆代码清理                                  │
├─────────────────────────────────────────────────────────┤
│ 过滤应用                                                 │
│  ├── Include Filters (CSS/XPath/JSON)                    │
│  └── Subtractive Selectors (元素移除)                    │
├─────────────────────────────────────────────────────────┤
│ 文本提取                                                 │
│  ├── html_to_text 转换                                   │
│  └── 保留原始格式 (source URL 模式)                       │
├─────────────────────────────────────────────────────────┤
│ 文本转换                                                 │
│  ├── 空白修剪 (trim_text_whitespace)                     │
│  ├── 去重行 (remove_duplicate_lines)                     │
│  └── 字母排序 (sort_text_alphabetically)                 │
├─────────────────────────────────────────────────────────┤
│ 规则引擎                                                 │
│  ├── Trigger Text (触发文本)                             │
│  ├── Text Should Not Be Present (禁止文本)               │
│  └── Custom Conditions (自定义条件)                      │
├─────────────────────────────────────────────────────────┤
│ 校验和计算                                               │
│  └── MD5(处理后的文本)                                    │
└─────────────────────────────────────────────────────────┘
```

#### 关键类结构

| 类 | 职责 |
|----|------|
| `FilterConfig` | 合并 Watch/Tag/Global 的过滤配置 |
| `ContentTransformer` | 文本修剪、去重、排序、提取 |
| `RuleEngine` | 触发规则、条件评估 |
| `ContentProcessor` | 内容预处理、过滤、文本提取 |
| `ChecksumCalculator` | MD5 校验和计算 |

### 3.2 补货/价格处理器 (`restock_diff`)

**位置**: `processors/restock_diff/processor.py`

#### 功能特点
- 专门用于电商产品库存和价格监控
- 支持 JSON-LD、Microdata、OpenGraph 元数据解析
- 价格变化阈值和库存状态检测

#### 输出格式
- **快照内容**: 格式化字符串 `"In Stock: True - Price: 99.99"`
- **检测数据**: `Restock` 对象 (dict-like)
  - `price`: 价格
  - `currency`: 货币
  - `availability`: 可用性状态
  - `in_stock`: 布尔值

#### 处理流程
```
┌─────────────────────────────────────────────────────────┐
│ 元数据提取 (两种方式)                                     │
│                                                          │
│ 方式 1: 纯 Python 提取 (优先，无 lxml 内存泄漏)            │
│  └── regex 提取 JSON-LD、Meta 标签                        │
│                                                          │
│ 方式 2: extruct 解析 (Linux 下使用子进程隔离)              │
│  └── multiprocessing spawn 模式避免内存泄漏               │
├─────────────────────────────────────────────────────────┤
│ 插件兜底                                                 │
│  └── LLM/自定义插件可覆盖提取结果                          │
├─────────────────────────────────────────────────────────┤
│ 检测逻辑                                                 │
│  ├── 库存变化: out_of_stock → in_stock                    │
│  ├── 价格变化: previous_price != current_price            │
│  ├── 价格阈值: % 变化超过阈值                              │
│  └── 价格区间: min/max 限制                               │
├─────────────────────────────────────────────────────────┤
│ 校验和计算                                               │
│  └── MD5("In Stock: {in_stock} - Price: {price}")        │
└─────────────────────────────────────────────────────────┘
```

#### 配置选项 (`restock_diff.json`)
```json
{
  "restock_diff": {
    "follow_price_changes": true,
    "in_stock_processing": "in_stock_only",
    "price_change_threshold_percent": 5,
    "price_change_min": 10,
    "price_change_max": 1000
  }
}
```

### 3.3 视觉截图处理器 (`image_ssim_diff`)

**位置**: `processors/image_ssim_diff/processor.py`

#### 功能特点
- 基于 OpenCV 的像素级差异检测
- 支持区域裁剪 (bounding box / visual selector)
- 子进程隔离确保内存管理

#### 输出格式
- **快照内容**: PNG 截图字节流 (bytes)
- **差异分数**: 变化像素百分比 (0-100)
- **触发条件**: `change_score > min_change_percentage`

#### 处理流程
```
┌─────────────────────────────────────────────────────────┐
│ 前置检查                                                 │
│  ├── 确保使用浏览器 fetcher (playwright/webdriver)        │
│  └── MD5 快速检查 (图片完全相同则跳过)                     │
├─────────────────────────────────────────────────────────┤
│ 区域裁剪 (可选)                                          │
│  ├── Bounding Box: "x,y,width,height"                    │
│  └── Visual Selector: 从 xpath_data 提取元素位置          │
├─────────────────────────────────────────────────────────┤
│ 图像比较 (子进程隔离)                                     │
│  └── isolated_opencv.compare_images_isolated()           │
│      ├── 高斯模糊 (sigma=OPENCV_BLUR_SIGMA)               │
│      ├── 像素差异阈值 (0-255)                             │
│      └── 输出: 变化像素百分比                              │
├─────────────────────────────────────────────────────────┤
│ 阈值判断                                                 │
│  └── change_score > min_change_percentage (默认 1%)      │
└─────────────────────────────────────────────────────────┘
```

#### 配置选项
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `pixel_difference_threshold_sensitivity` | 30 | 像素差异阈值 (0-255) |
| `min_change_percentage` | 1 | 最小变化百分比触发 |
| `bounding_box` | None | 裁剪区域 |
| `auto_track_region` | False | 模板匹配跟踪 (暂未启用) |

### 3.4 处理器输出对比总结

| 特性 | text_json_diff | restock_diff | image_ssim_diff |
|------|----------------|--------------|-----------------|
| **快照格式** | UTF-8 字符串 | UTF-8 字符串 | PNG 字节流 |
| **差异检测** | MD5 校验和 | MD5 校验和 | 像素变化百分比 |
| **适用场景** | 通用文本/JSON | 电商产品监控 | 视觉变化监控 |
| **历史对比** | 文本行级别 | 价格/库存状态 | 图像像素级别 |
| **输出到通知** | `{{diff}}` 文本差异 | `{{watch_title}}` 等 | 截图附件 |
| **特殊功能** | 过滤/规则/LLM | 价格阈值/库存检测 | 区域裁剪/像素阈值 |

---

## 4. 通知模板处理

### 4.1 通知服务架构

#### 通知服务初始化
```python
# notification_service.py
class NotificationService:
    def __init__(self, datastore, notification_q):
        self.datastore = datastore
        self.notification_q = notification_q
```

#### 级联优先级 (_check_cascading_vars)
通知配置采用三级优先级：

```python
# 1. Watch 级别 (最高优先级)
v = watch.get(var_name)
if v and not watch.get('notification_muted'):
    return v

# 2. Tag 级别
for tag_uuid, tag in tags.items():
    v = tag.get(var_name)
    if v and not tag.get('notification_muted'):
        return v

# 3. Global 级别 (最低优先级)
if datastore.data['settings']['application'].get(var_name):
    return datastore.data['settings']['application'].get(var_name)
```

### 4.2 通知上下文数据

#### NotificationContextData 类
定义了所有可用的模板变量：

```python
class NotificationContextData(dict):
    def __init__(self, initial_data=None, **kwargs):
        super().__init__({
            # 基本信息
            'base_url': None,
            'watch_url': 'https://WATCH-PLACE-HOLDER/',
            'watch_uuid': 'XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX',
            'watch_title': None,
            'watch_tag': None,
            'watch_mime_type': None,
            
            # 时间戳
            'change_datetime': FormattableTimestamp(time.time()),
            'notification_timestamp': time.time(),
            
            # 快照数据
            'prev_snapshot': None,
            'current_snapshot': None,
            
            # 差异数据 (多种格式)
            'diff': FormattableDiff('', ''),
            'diff_clean': FormattableDiff('', '', include_change_type_prefix=False),
            'diff_added': FormattableDiff('', '', include_removed=False),
            'diff_removed': FormattableDiff('', '', include_added=False),
            'diff_full': FormattableDiff('', '', include_equal=True),
            'diff_patch': FormattableDiff('', '', patch_format=True),
            'diff_changed_from': FormattableExtract('', '', extract_fn=extract_changed_from),
            'diff_changed_to': FormattableExtract('', '', extract_fn=extract_changed_to),
            
            # LLM 相关
            'llm_summary': None,
            'llm_intent': None,
            
            # 链接
            'diff_url': None,
            'preview_url': None,
            
            # 附件
            'screenshot': None,
            
            # 触发文本
            'triggered_text': None,
        })
```

#### 可调用的格式变量

`FormattableDiff` 和 `FormattableTimestamp` 支持在模板中调用参数：

```jinja2
{# 基本用法 #}
{{ diff }}

{# 限制行数 #}
{{ diff(lines=5) }}

{# 只显示新增/删除 #}
{{ diff(added_only=true) }}
{{ diff(removed_only=true) }}

{# 上下文行数 #}
{{ diff(context=3) }}

{# 行级别差异 #}
{{ diff(word_diff=false) }}

{# 时间戳格式 #}
{{ change_datetime }}
{{ change_datetime(format='%Y-%m-%d') }}
{{ change_datetime(format='%A, %B %d, %Y') }}
```

### 4.3 差异渲染优化

#### 按需渲染 (`add_rendered_diff_to_notification_vars`)
为了性能优化，只渲染模板中实际使用的差异变量：

```python
# 扫描模板文本
for key in NotificationContextData().keys():
    if not key.startswith('diff'):
        continue
    pattern = rf"(?<![A-Za-z0-9_]){re.escape(key)}(?![A-Za-z0-9_])"
    if not re.search(pattern, notification_scan_text, re.IGNORECASE):
        continue
    # 仅渲染实际使用的 diff 变量
    ret[key] = FormattableDiff(prev_snapshot, current_snapshot, **diff_specs[key])
```

#### LLM 摘要覆盖
如果启用了 LLM 摘要，`{{diff}}` 会被 AI 摘要替换：

```python
_llm_change_summary = (n_object.get('_llm_change_summary') or '').strip()
_override_diff = datastore.data['settings']['application'].get('llm_override_diff_with_summary', True)
if _llm_change_summary and _override_diff:
    n_object['diff'] = _llm_change_summary
n_object['raw_diff'] = n_object.get('diff', '')  # 保留原始差异
```

### 4.4 模板渲染流程

```
┌─────────────────────────────────────────────────────────┐
│ 1. 获取通知配置 (级联优先级)                              │
│    notification_urls / title / body / format             │
├─────────────────────────────────────────────────────────┤
│ 2. 构建通知上下文                                        │
│    - set_basic_notification_vars()                       │
│    - 填充 prev/current_snapshot                          │
│    - 添加 watch.extra_notification_token_values()        │
├─────────────────────────────────────────────────────────┤
│ 3. 按需渲染差异                                          │
│    - add_rendered_diff_to_notification_vars()            │
│    - 仅渲染模板中使用的 {{diff_*}} 变量                   │
├─────────────────────────────────────────────────────────┤
│ 4. Jinja2 渲染                                           │
│    - jinja_render(template_str=n_object['notification_body'], **params)│
│    - 支持自定义 Jinja 扩展 (regex, time 等)               │
├─────────────────────────────────────────────────────────┤
│ 5. 服务特定调整                                          │
│    - apply_service_tweaks()                              │
│    - Discord: 转换为 Markdown 或 embeds                  │
│    - Telegram: 有限 HTML 子集                             │
│    - Email: 等宽字体包装                                 │
└─────────────────────────────────────────────────────────┘
```

### 4.5 通知格式类型

| 格式 | 说明 | 差异标记 |
|------|------|----------|
| `text` | 纯文本 | `(added)`, `(removed)`, `(changed)` |
| `html` | HTML | `<span style="...">` 彩色标记 |
| `htmlcolor` | 彩色 HTML | 同 HTML，带颜色样式 |
| `markdown` | Markdown | `**added**`, `<del>removed</del>` |

---

## 5. 底层发送通道

### 5.1 Apprise 集成

系统使用 **Apprise** 作为通知发送的底层库：

```python
# notification/handler.py
import apprise
from apprise import NotifyFormat

apobj = apprise.Apprise(debug=True, asset=apprise_asset)
apobj.add(url)
apobj.notify(
    title=n_title,
    body=n_body,
    body_format=apprise_input_format,
    attach=n_object.get('screenshot', None)
)
```

### 5.2 自定义插件

#### Discord 自定义插件
```python
# 覆盖默认 Discord 插件以支持彩色 embeds
apprise.plugins.N_MGR.remove('discord')
apprise.plugins.N_MGR.add(NotifyDiscordCustom, schemas='discord')
```

#### HTTP 自定义处理器
```python
from .apprise_plugin.custom_handlers import apprise_http_custom_handler
```

### 5.3 支持的通知类型

通过 Apprise 支持的通道包括（但不限于）：

| 类别 | 通道 | URL 模式 |
|------|------|----------|
| **邮件** | SMTP | `mailtos://`, `mail://` |
| **即时通讯** | Slack | `slack://` |
| | Discord | `discord://`, `https://discord.com/api/...` |
| | Telegram | `tgram://` |
| | Teams | `msteams://` |
| | Mattermost | `mmost://` |
| **Webhook** | 通用 HTTP | `json://`, `forms://`, `get://`, `post://` |
| **消息队列** | Redis | `redis://` |
| | MQTT | `mqtt://` |
| **移动端** | Pushbullet | `pbul://` |
| | Pushover | `pover://` |
| **短信** | Twilio | `twilio://` |
| **DevOps** | OpsGenie | `opsgenie://` |
| | PagerDuty | `pagerduty://` |

### 5.4 服务特定调整 (`apply_service_tweaks`)

#### Telegram 调整
```python
if url.startswith('tgram://'):
    # 仅支持 <s>, <b>, <i>, <code>, <pre>
    # 移除 <br> 并替换为 \n
    n_body = n_body.replace('<br>', '\n')
    # 限制: 标题 4096 字符，正文 3600 字符
```

#### Discord 调整
```python
if url.startswith('discord://'):
    # 转换为 Discord Markdown:
    # ~~strikethrough~~ (删除), **bold** (新增)
    # htmlcolor 模式: 使用自定义 embeds (6000 字符限制)
    # html 模式: 纯文本 (2000 字符限制)
```

#### Email 调整
```python
if url.startswith('mail') and 'html' in requested_output_format:
    # 使用等宽字体包装
    n_body = as_monospaced_html_email(content=n_body, title=n_title)
```

---

## 6. 错误处理与重试

### 6.1 处理器级别错误

#### 常见异常类型
```python
# content_fetchers/exceptions/__init__.py (概念)

checksumFromPreviousCheckWasTheSame      # 内容未变化，跳过
ReplyWithContentButNoText                 # 有内容但无文本
Non200ErrorCodeReceived                   # 非 200 状态码
BrowserConnectError                       # 浏览器连接失败
BrowserFetchTimedOut                      # 浏览器超时
BrowserStepsStepException                 # 浏览器步骤失败
EmptyReply                                # 空响应
ScreenshotUnavailable                     # 截图不可用
JSActionExceptions                        # JS 执行异常
PageUnloadable                            # 页面无法加载
BrowserStepsInUnsupportedFetcher          # 不支持的 fetcher

# processors/exceptions.py
ProcessorException                        # 通用处理器异常

# processors/text_json_diff/processor.py
FilterNotFoundInResponse                  # 过滤器未找到
PDFToHTMLToolNotFound                     # PDF 工具未找到
```

#### 错误处理流程 (worker.py)

```
异常捕获
    ↓
┌─────────────────────────────────────────────────────────┐
│ FilterNotFoundInResponse                                 │
│  ├── 保存截图和 xpath_data                               │
│  ├── 递增 consecutive_filter_failures                    │
│  └── 达到阈值时发送 filter_failure_notification          │
├─────────────────────────────────────────────────────────┤
│ BrowserStepsStepException                                │
│  ├── 记录 browser_steps_last_error_step                  │
│  ├── 递增 consecutive_filter_failures                    │
│  └── 达到阈值时发送 step_failure_notification            │
├─────────────────────────────────────────────────────────┤
│ ProcessorException                                       │
│  ├── 保存截图和 xpath_data                               │
│  └── 记录 last_error                                     │
├─────────────────────────────────────────────────────────┤
│ 其他异常                                                 │
│  ├── 清理错误工件 (last-error-screenshot.png)            │
│  ├── 记录到 watch.last_error                             │
│  └── process_changedetection_results = False             │
└─────────────────────────────────────────────────────────┘
```

### 6.2 通知级别错误

#### 发送时错误捕获
```python
# notification/handler.py: process_notification()
with apprise.LogCapture(level=apprise.logging.DEBUG) as logs:
    # ... 发送逻辑 ...
    apobj.notify(...)
    
    log_value = logs.getvalue()
    if log_value and ('WARNING' in log_value or 'ERROR' in log_value):
        logger.critical(log_value)
        raise Exception(log_value)
```

#### 通知错误记录
```python
# worker.py 中初始化
update_obj = {'last_notification_error': False, 'last_error': False}
```

#### 过滤器失败通知
```python
# notification_service.py: send_filter_failure_notification()
n_object = NotificationContextData({
    'notification_title': 'Alert - CSS/xPath filter was not present',
    'notification_body': f"Your filters '{filter_list}' did not appear...",
})
# 通知 URL 优先级: Watch > Global
if len(watch['notification_urls']):
    n_object['notification_urls'] = watch['notification_urls']
elif len(self.datastore.data['settings']['application']['notification_urls']):
    n_object['notification_urls'] = self.datastore.data['settings']['application']['notification_urls']
```

### 6.3 重试机制

#### 隐式重试 (调度器)
系统没有显式的通知重试队列，但通过以下机制实现间接重试：

1. **Watch 调度循环**
   - 每个 watch 按 `check_interval` 定期检查
   - 如果检查失败，下次调度时会自动重试

2. **失败通知**
   - 过滤器/步骤失败达到阈值时发送失败通知
   - 通知内容包含修复建议

3. **连续失败计数**
   ```python
   # Filter/Step 失败
   c = watch.get('consecutive_filter_failures', 0)
   c += 1
   if c >= threshold:
       send_filter_failure_notification()
       c = 0  # 重置计数
   ```

#### 工作器重启策略
```python
# worker.py
max_jobs = int(os.getenv("WORKER_MAX_JOBS", "10"))
max_runtime_seconds = int(os.getenv("WORKER_MAX_RUNTIME", "3600"))  # 1小时

# 触发重启条件:
# 1. 处理 job 数达到 max_jobs
# 2. 运行时间达到 max_runtime_seconds
return "restart"
```

### 6.4 内存安全与资源清理

#### 处理器级清理
```python
# text_json_diff/processor.py: 显式删除大对象
del content
if 'html_content' in locals() and html_content is not stripped_text:
    del html_content
# ...
import gc
gc.collect()
```

#### 子进程隔离 (restock_diff, image_ssim_diff)
```python
# restock_diff: 使用 multiprocessing spawn 模式
ctx = multiprocessing.get_context('spawn')
p = ctx.Process(target=_extract_itemprop_availability_worker, args=(child_conn,))
# 子进程退出时 OS 回收所有内存 (包括 lxml C 级分配)

# image_ssim_diff: 独立线程 + 子进程
thread = threading.Thread(target=thread_target, daemon=True, name="ImageDiff-Processor")
# 内部调用 isolated_opencv.compare_images_isolated()
```

#### 工作器 finally 块
```python
finally:
    # 1. 退出浏览器
    await update_handler.fetcher.quit(watch=watch)
    
    # 2. 清空内容
    update_handler.fetcher.clear_content()
    
    # 3. 删除引用
    if update_handler:
        del update_handler
    
    # 4. 强制 GC
    import gc
    gc.collect()
    
    # 5. 释放 UUID
    worker_pool.release_uuid_from_processing(uuid, worker_id=worker_id)
```

---

## 7. 完整流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        调度器 (Scheduler)                              │
│  按 check_interval 将 watch 加入 RecheckPriorityQueue                  │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│                      异步工作器 (Async Worker)                         │
│                                                                       │
│  1. 从队列获取 watch UUID                                              │
│  2. claim_uuid_for_processing() 防止重复处理                           │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ 内容抓取阶段                                                      │  │
│  │                                                                 │  │
│  │  call_browser()                                                  │  │
│  │   ├── SSRF 检查 (IANA 受限地址)                                  │  │
│  │   ├── 选择 Fetcher: requests / playwright / puppeteer           │  │
│  │   │     (基于 watch.fetch_backend 和 datastore 设置)             │  │
│  │   ├── 应用代理、请求头、Cookie、超时                               │  │
│  │   ├── 执行浏览器步骤 (如有)                                       │  │
│  │   └── 获取: content, screenshot, xpath_data, headers             │  │
│  └─────────────────────────────────────────────────────────────────┘  │
                              │                                          │
                              ↓                                          │
  ┌─────────────────────────────────────────────────────────────────┐  │
  │ 处理器分流 & 差异检测                                              │  │
  │                                                                 │  │
  │  processor = watch.get('processor', 'text_json_diff')            │  │
  │                                                                 │  │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐       │  │
  │  │ text_json_   │  │ restock_     │  │ image_ssim_      │       │  │
  │  │ diff         │  │ diff         │  │ diff             │       │  │
  │  │              │  │              │  │                  │       │  │
  │  │ 文本/JSON    │  │ 库存/价格    │  │ 截图比较         │       │  │
  │  │ CSS/XPath    │  │ JSON-LD      │  │ OpenCV 像素     │       │  │
  │  │ 过滤规则     │  │ 元数据       │  │ 区域裁剪         │       │  │
  │  │              │  │              │  │                  │       │  │
  │  │ 输出: 文本   │  │ 输出: 状态   │  │ 输出: 图片字节   │       │  │
  │  └──────────────┘  └──────────────┘  └──────────────────┘       │  │
  │                          │                                         │  │
  │                          ↓                                         │  │
  │               changed_detected?                                    │  │
  │                    │                                               │  │
  │              Yes ──┴── No                                          │  │
  │                │        │                                          │  │
  │                ↓        │                                          │  │
  │  ┌──────────────────┐   │                                          │  │
  │  │ LLM 评估 (可选)  │   │                                          │  │
  │  │                  │   │                                          │  │
  │  │ 1. Intent 过滤   │   │                                          │  │
  │  │    (是否重要变更)│   │                                          │  │
  │  │                  │   │                                          │  │
  │  │ 2. Change 摘要  │   │                                          │  │
  │  │    (AI 总结)    │   │                                          │  │
  │  └──────────────────┘   │                                          │  │
  │           │             │                                          │  │
  │           ↓             │                                          │  │
  │  ┌──────────────────┐   │                                          │  │
  │  │ 保存快照历史     │   │                                          │  │
  │  │ save_history_    │   │                                          │  │
  │  │ blob()           │   │                                          │  │
  │  └──────────────────┘   │                                          │  │
  │           │             │                                          │  │
  │           ↓             │                                          │  │
  │  ┌────────────────────────────────────────────────────────────┐   │  │
  │  │ 通知决策: history_n >= 2 AND not notification_muted       │   │  │
  │  └────────────────────────────┬───────────────────────────────┘   │  │
  │                               │                                   │  │
  │                               ↓                                   │  │
  └───────────────────────────────┼───────────────────────────────────┘  │
                                  │                                      │
                                  ↓                                      │
┌──────────────────────────────────────────────────────────────────────┐
│                      通知服务 (Notification Service)                   │
│                                                                       │
│  1. 级联获取配置                                                       │
│     Watch → Tag → Global                                              │
│     notification_urls, title, body, format                            │
│                                                                       │
│  2. 构建通知上下文                                                     │
│     NotificationContextData()                                         │
│     ├── prev/current_snapshot                                         │
│     ├── diff, diff_added, diff_removed...                             │
│     ├── llm_summary, llm_intent                                       │
│     └── watch_url, diff_url, screenshot                               │
│                                                                       │
│  3. 加入 NotificationQueue                                             │
│     notification_q.put(n_object)                                      │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│                    通知处理器 (Notification Handler)                   │
│                                                                       │
│  process_notification(n_object, datastore)                            │
│                                                                       │
│  1. 按需渲染差异变量                                                   │
│     add_rendered_diff_to_notification_vars()                          │
│                                                                       │
│  2. Jinja2 模板渲染                                                    │
│     jinja_render(template_str=body, **params)                         │
│                                                                       │
│  3. 服务特定调整                                                       │
│     apply_service_tweaks()                                            │
│     ├── Discord: Markdown / embeds                                   │
│     ├── Telegram: 有限 HTML + 长度限制                                │
│     ├── Email: 等宽字体包装                                           │
│     └── 通用: 差异标记替换                                             │
│                                                                       │
│  4. Apprise 发送                                                       │
│     apobj = apprise.Apprise()                                         │
│     apobj.add(url)                                                    │
│     apobj.notify(title, body, body_format, attach=screenshot)         │
│                                                                       │
│  5. 错误捕获                                                           │
│     LogCapture(level=DEBUG)                                           │
│     if WARNING/ERROR in logs: raise Exception()                       │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ↓
                ┌─────────────┴─────────────┐
                │                           │
                ↓                           ↓
        ┌───────────────┐           ┌───────────────┐
        │  发送成功     │           │  发送失败     │
        │               │           │               │
        │  记录到日志   │           │  raise       │
        │  (logger)    │           │  Exception   │
        └───────────────┘           └───────┬───────┘
                                            │
                                            ↓
                            ┌───────────────────────────────┐
                            │  工作器异常捕获               │
                            │  logger.error()              │
                            │  datastore.update_watch(     │
                            │    {'last_error': ...}       │
                            │  )                            │
                            │                               │
                            │  下次调度自动重试             │
                            │  (隐式重试机制)               │
                            └───────────────────────────────┘
```

---

## 8. 关键设计决策

### 8.1 处理器设计
- **插件化**: 通过 `pluggy` 支持第三方处理器
- **继承式**: 所有处理器继承 `difference_detection_processor` 基类
- **输出统一**: 均返回 `(changed_detected, update_obj, contents)` 三元组

### 8.2 通知设计
- **级联配置**: Watch > Tag > Global 三级优先级
- **按需渲染**: 只计算模板中实际使用的差异变量
- **服务适配**: 每个通知通道有独立的格式转换逻辑
- **安全优先**: HTML 通知时对页面内容进行 HTML 转义防止 XSS

### 8.3 错误处理设计
- **细粒度异常**: 每个失败场景有特定异常类型
- **资源隔离**: 大内存操作使用子进程/独立线程
- **隐式重试**: 通过调度循环实现自动重试
- **快速失败**: 过滤器/步骤失败累积计数，达到阈值立即通知

### 8.4 性能优化
- **MD5 快速跳过**: 原始内容不变时跳过复杂处理
- **子进程隔离**: 避免 C 扩展内存泄漏
- **按需渲染**: 惰性计算差异变量
- **配置哈希**: 过滤器配置变化时自动重新处理

---

## 9. 关键文件索引

| 模块 | 路径 | 关键类/函数 |
|------|------|------------|
| 处理器注册 | `processors/__init__.py` | `find_processors()`, `available_processors()` |
| 处理器基类 | `processors/base.py` | `difference_detection_processor`, `call_browser()` |
| 文本处理器 | `processors/text_json_diff/processor.py` | `perform_site_check`, `FilterConfig`, `ContentTransformer` |
| 补货处理器 | `processors/restock_diff/processor.py` | `perform_site_check`, `extract_itemprop_availability_safe()` |
| 图像处理器 | `processors/image_ssim_diff/processor.py` | `perform_site_check` |
| 通知服务 | `notification_service.py` | `NotificationService`, `NotificationContextData` |
| 通知发送 | `notification/handler.py` | `process_notification()`, `apply_service_tweaks()` |
| 工作器 | `worker.py` | `async_update_worker()`, `send_content_changed_notification()` |
| 队列 | `queue_handlers.py` | `RecheckPriorityQueue`, `NotificationQueue` |

---

*报告生成时间: 2026-05-12*
*基于代码库: changedetection.io*
