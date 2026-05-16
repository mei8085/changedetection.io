# 变更检测系统架构分析报告（最终修订版）

## 概述

本文档基于**逐行代码证据**详细分析了changedetection.io项目的内容抓取、差异检测、数据格式化和预览功能实现机制。重点修正了所有抓取器的能力标志边界、静态标志与实际行为不一致的前端表现差异。

**分析依据**（精确代码位置）：
- `changedetectionio/content_fetchers/base.py:69-76` - Fetcher基类能力标志默认值
- `changedetectionio/content_fetchers/requests.py:8-207` - HTTP抓取器实现
- `changedetectionio/content_fetchers/playwright.py:153-172` - Playwright抓取器实现
- `changedetectionio/content_fetchers/puppeteer.py:171-186` - Puppeteer抓取器实现
- `changedetectionio/content_fetchers/webdriver_selenium.py:8-20` - Selenium抓取器实现
- `changedetectionio/pluggy_interface.py:444-496` - 能力标志获取逻辑
- `changedetectionio/blueprint/ui/preview.py:90-125` - 预览路由实现
- `changedetectionio/blueprint/ui/templates/preview.html:79-94` - 预览页面模板

---

## 一、各抓取器能力标志精确核对

### 1.1 能力标志定义机制

**获取逻辑证据** (`pluggy_interface.py:444-496`)：
```python
def get_fetcher_capabilities(watch, datastore):
    # 从fetcher类获取静态属性，不是实例属性
    return {
        'supports_browser_steps': getattr(fetcher_class, 'supports_browser_steps', False),
        'supports_screenshots': getattr(fetcher_class, 'supports_screenshots', False),
        'supports_xpath_element_data': getattr(fetcher_class, 'supports_xpath_element_data', False)
    }
```

**关键发现**：能力标志是**类级静态属性**，在初始化时确定，不随运行时状态改变。

### 1.2 各抓取器能力标志最终核对表

| 抓取器 | 截图能力 | 浏览器步骤 | 视觉选择器 | 代码位置 |
|-------|---------|-----------|-----------|---------|
| **Requests (html_requests)** | ✅ **False** | ✅ **False** | ✅ **False** | `requests.py` 基类继承 |
| **Playwright** | ✅ **True** | ✅ **True** | ✅ **True** | `playwright.py:170-172` |
| **Puppeteer** | ✅ **True** | ✅ **True** | ✅ **True** | `puppeteer.py:184-186` |
| **Selenium WebDriver** | ✅ **True** | ✅ **False** | ✅ **True** | `webdriver_selenium.py:18-20` |
| **基类 (Fetcher)** | False | False | False | `base.py:69-71` |

**修正说明**：
- ✅ Selenium的`supports_browser_steps`实际为`False`（之前错误标记为True）
- ✅ 所有能力标志均基于类静态属性，与运行时行为无关

---

## 二、有条件能力与静态标志不一致的前端表现差异

### 2.1 Requests抓取器的"隐形截图"问题

**运行时特殊行为证据** (`requests.py:203-207`)：
```python
# 如果内容是图像，将其设置为截图用于SSIM/视觉比较
content_type = r.headers.get('content-type', '').lower()
if 'image/' in content_type:
    self.screenshot = r.content
    logger.debug(f"检测到图像内容({content_type})，设置为截图用于比较")
```

**前端显示条件证据** (`preview.html:84-93`)：
```jinja2
{% if capabilities.supports_screenshots %}       {# 外层条件：静态标志 #}
    {% if screenshot %}                           {# 内层条件：实际数据 #}
        <img src="..." alt="当前截图">
    {% else %}
        暂无截图，请重新检查页面
    {% endif %}
{% else %}
    <strong>需要支持截图的Content Fetcher才能显示截图。</strong>
{% endif %}
```

**最终表现差异分析**：

| 场景 | 静态标志 | 实际有截图 | 前端表现 | 用户体验 |
|------|---------|-----------|---------|---------|
| 普通网页 (Requests) | False | False | 显示"需要支持截图的Fetcher" | ✅ 符合预期 |
| 图片URL (Requests) | False | True | **仍然显示"需要支持截图的Fetcher"** | ❌ 误导用户 |
| 任何页面 (Playwright) | True | True/False | 截图存在时显示，否则提示重新检查 | ✅ 符合预期 |

**问题本质**：双重判断机制导致"有截图但不显示"的悖论
1. 外层 `capabilities.supports_screenshots` 基于**类静态属性**
2. 内层 `screenshot` 基于**实际快照数据**
3. Requests在图片URL时实际保存了截图，但外层条件直接隐藏了整个截图区域

### 2.2 处理器自定义预览的绕过机制

**预览路由分发逻辑** (`preview.py:14-56`)：
```python
def preview_page(uuid):
    processor_name = watch.get('processor', 'text_json_diff')
    processor_module = get_processor_submodule(processor_name, 'preview')
    
    if processor_module and hasattr(processor_module, 'render'):
        # 处理器自定义预览 - 绕过fetcher能力标志检查
        return processor_module.render(...)
    
    # 默认文本预览 - 受fetcher能力标志限制
    return render_template("preview.html", ...)
```

**不同处理器的表现对比**：

| 处理器 | 自定义预览 | 截图显示方式 | 受静态标志限制？ |
|-------|-----------|-------------|----------------|
| `text_json_diff` | ❌ 无 | 默认模板双重判断 | ✅ 是 |
| `image_ssim_diff` | ✅ 有 | processor-asset路由直接流式传输 | ❌ 否 |
| `restock_diff` | ❌ 无 | 默认模板双重判断 | ✅ 是 |

**关键发现**：`image_ssim_diff`等带自定义预览的处理器完全绕过了`supports_screenshots`静态标志检查，即使使用Requests fetcher也能显示图像比较结果。

### 2.3 分块截图与高度限制参数确认

**常量定义证据** (`content_fetchers/__init__.py:13-26`)：
```python
# 最大截图高度 - 可通过环境变量 SCREENSHOT_MAX_HEIGHT 覆盖
SCREENSHOT_MAX_HEIGHT_DEFAULT = 20000
SCREENSHOT_MAX_TOTAL_HEIGHT = int(os.getenv("SCREENSHOT_MAX_HEIGHT", 20000))

# 分块阈值高度 - 超过此高度才启用分块拼接
# 可通过环境变量 SCREENSHOT_CHUNK_HEIGHT 覆盖
SCREENSHOT_SIZE_STITCH_THRESHOLD = int(os.getenv("SCREENSHOT_CHUNK_HEIGHT", 10000))

# 默认JPEG质量
SCREENSHOT_DEFAULT_QUALITY = 40
```

**最终参数确认表**：

| 参数 | 默认值 | 环境变量 | 说明 |
|------|--------|---------|------|
| 最大总高度 | **20000px** | `SCREENSHOT_MAX_HEIGHT` | 超过会被截断并显示警告 |
| 分块阈值高度 | **10000px** | `SCREENSHOT_CHUNK_HEIGHT` | 超过此高度才启用分块拼接 |
| 默认JPEG质量 | **40** | 无 | 截图压缩质量 |

**分块拼接触发条件**（双重AND条件）：
1. 页面实际高度 > `SCREENSHOT_SIZE_STITCH_THRESHOLD` (10000px)
2. 页面实际高度 > 当前视口高度

### 2.4 视口元素锁定功能状态

**代码证据** (`base.py:73-84`)：
```python
# Screenshot element locking - prevents layout shifts during screenshot capture
# Only needed for visual comparison (image_ssim_diff processor)
# Locks element dimensions in the first viewport to prevent headers/ads from resizing
lock_viewport_elements = False      # Default: disabled for performance

def __init__(self, proxy_override=None, custom_browser_connection_url=None, **kwargs):
    super().__init__()
    # 仅当kwargs明确传入时才启用，否则保持默认False
    if 'lock_viewport_elements' in kwargs:
        self.lock_viewport_elements = kwargs['lock_viewport_elements']
```

**当前功能状态**：
- ✅ 代码框架完整（注入/解锁JS脚本、条件分支）
- ✅ 没有任何调用路径将此参数设置为True
- ❌ **此功能实际上完全未启用**
- 注释提到"image_ssim_diff processor"会启用，但代码路径中找不到实现

---

## 三、内容抓取层向通知与存储模块传递结果的路径

### 3.1 数据流向完整链路

```
┌─────────────────────────────────────────────────────────────────────┐
│                          任务队列层                                  │
│                  PriorityQueue + 调度线程                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               ↓ watch_uuid
┌─────────────────────────────────────────────────────────────────────┐
│                          Worker处理层                                │
│   processor_module.perform_site_check() → call_browser()            │
└──────────────────────────────┬──────────────────────────────────────┘
                               ↓ Fetcher实例
┌─────────────────────────────────────────────────────────────────────┐
│                         内容抓取层 (Fetcher)                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  .content        → 处理后文本或哈希值                         │  │
│  │  .headers        → HTTP响应头字典                             │  │
│  │  .status_code    → HTTP状态码                                │  │
│  │  .screenshot     → 截图二进制(有条件设置)                     │  │
│  │  .xpath_data     → 视觉选择器数据JSON(浏览器模式)             │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               ↓ Fetcher属性
┌─────────────────────────────────────────────────────────────────────┐
│                       变更检测层 (Processor)                         │
│  run_changedetection() → (changed_detected, update_obj, contents)   │
└──────────────────────────────┬──────────────────────────────────────┘
                               ↓ update_obj
┌─────────────────────────────────────────────────────────────────────┐
│                         数据持久化层 (Watch)                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  save_history_text()       → 文本快照文件                      │  │
│  │  save_screenshot()         → 截图PNG/JPEG文件                 │  │
│  │  save_xpath_data()         → 视觉选择器JSON文件               │  │
│  │  save_error_snapshot()     → 错误截图/文本                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               ↓ 检测结果 + 内容
┌─────────────────────────────────────────────────────────────────────┐
│                          通知分发层                                  │
│  notification_q → Apprise → Email/Slack/Telegram等                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 关键传递节点的条件分支

**Fetcher层条件传递**：
```python
# Requests fetcher仅在特定条件下设置screenshot
if 'image/' in content_type:
    self.screenshot = r.content  # 仅图像URL时传递
    
# Playwright/Puppeteer始终传递截图（除非明确禁用）
if self.screenshot_enabled:
    self.screenshot = capture_page_screenshot()
    
# 视觉选择器仅在支持的浏览器模式下传递
if self.supports_xpath_element_data:
    self.xpath_data = extract_element_positions()
```

**Processor层条件传递**：
```python
# 仅当内容发生变化时才触发通知
if changed_detected and threshold_met:
    add_to_notification_queue()

# 仅当处理器自定义预览存在时才绕过默认模板
if processor_module and hasattr(processor_module, 'render'):
    use_custom_preview()
else:
    use_default_template()
```

---

## 四、差异数据格式化为前端可展示的形式

### 4.1 Placemarker标记机制

**核心常量与替换映射** (`diff/__init__.py:33-58`, `notification/handler.py:86-101`)：

| Placemarker标记 | 替换后HTML样式 | 用途 |
|----------------|---------------|------|
| `REMOVED_PLACEMARKER_OPEN/CLOSED` | `background: #fadad7; color: #b30000` | 删除内容高亮 |
| `ADDED_PLACEMARKER_OPEN/CLOSED` | `background: #eaf2c2; color: #406619` | 新增内容高亮 |
| `CHANGED_PLACEMARKER_OPEN/CLOSED` | 同删除样式 | 被替换的旧内容 |
| `CHANGED_INTO_PLACEMARKER_OPEN/CLOSED` | 同新增样式 | 替换后的新内容 |

**标记替换安全特性**：
1. 中间标记而非直接HTML，支持多格式输出
2. 仅在最终渲染阶段执行替换，避免转义问题
3. 文本与样式分离，便于主题定制

### 4.2 差异渲染管道

```
原始文本A vs 原始文本B
        ↓ difflib.SequenceMatcher 行级比较
    带标签的行序列 (+/-/=)
        ↓ render_inline_word_diff() 词级比较
    Placemarker标记插入完成
        ↓ build_diff_cell_visualizer() 单元格概览
    100个单元格的变更热图数据
        ↓ apply_html_color_to_body() 样式应用
    HTML格式化输出
        ↓ Jinja模板 + diff_unescape_difference_spans
    前端最终展示
```

**词级差异实现细节** (`render_inline_word_diff()`)：
- 使用 diff-match-patch 库的 `linesToChars` 技术
- 支持整行替换模式和内嵌变更模式
- 变更粒度自动调整（词级 vs 字符级）

---

## 五、架构设计总结与问题清单

### 5.1 设计优势

| 设计决策 | 优势 |
|---------|------|
| **能力标志静态化** | 避免实例化开销，UI判断快速 |
| **Placemarker中间标记** | 多格式复用，转义安全 |
| **处理器感知预览** | 灵活支持自定义预览需求 |
| **分块截图** | 超大页面内存友好 |
| **processor-asset路由** | 大二进制数据流式传输，避免HTML嵌入内存问题 |

### 5.2 已确认的设计问题

| 问题编号 | 问题描述 | 影响范围 | 严重程度 |
|---------|---------|---------|---------|
| **P001** | Requests fetcher在图像URL时实际有截图，但`supports_screenshots=False`导致预览不显示 | 图片URL监控场景 | 中等 |
| **P002** | `lock_viewport_elements` 代码完整但无启用路径，注释与实现不一致 | 图像处理器视觉稳定性 | 低 |
| **P003** | 能力标志是类静态属性，无法表达"有条件支持"场景 | 未来功能扩展 | 中 |

### 5.3 各抓取器能力总览（最终版）

| 特性 | Requests | Playwright | Puppeteer | Selenium |
|------|----------|------------|-----------|----------|
| **supports_screenshots** | ❌ False | ✅ True | ✅ True | ✅ True |
| **实际截图行为** | ⚠️ 仅图像URL | ✅ 始终 | ✅ 始终 | ✅ 始终 |
| **预览时截图可见** | ❌ 即使有也不可见 | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| **supports_browser_steps** | ❌ False | ✅ True | ✅ True | ❌ False |
| **supports_xpath_element_data** | ❌ False | ✅ True | ✅ True | ✅ True |
| **视口元素锁定** | ❌ 不适用 | ⚠️ 代码存在未启用 | ⚠️ 代码存在未启用 | ❌ 不支持 |
| **JS执行** | ❌ 不支持 | ✅ 完整支持 | ✅ 完整支持 | ✅ 支持 |
| **分块截图** | ❌ 不适用 | ✅ >10000px时分块 | ✅ >10000px时分块 | ❌ 不支持 |

---

**文档生成时间**：2026年5月16日  
**分析版本**：changedetection.io 当前主分支（逐行代码验证）  
**分析范围**：所有抓取器能力标志、预览条件逻辑、数据传递路径
