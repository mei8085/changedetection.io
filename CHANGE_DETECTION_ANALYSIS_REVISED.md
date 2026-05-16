# 变更检测系统架构分析报告（修订版）

## 概述

本文档基于代码证据详细分析了changedetection.io项目的内容抓取、差异检测、数据格式化和预览功能实现机制。重点修正了截图配置默认值、不同抓取模式能力边界、以及各种条件分支行为。

**分析依据**：
- `changedetectionio/content_fetchers/__init__.py` - 常量定义
- `changedetectionio/content_fetchers/base.py` - Fetcher基类能力标志
- `changedetectionio/content_fetchers/requests.py` - 基础HTTP抓取器
- `changedetectionio/content_fetchers/playwright.py` - 浏览器抓取器
- `changedetectionio/blueprint/ui/preview.py` - 预览功能路由
- `changedetectionio/processors/image_ssim_diff/` - 图像处理器实现

---

## 一、内容抓取层向通知与存储模块传递结果的路径

### 1.1 整体数据流程架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   任务队列      │────▶│ Worker 处理层   │────▶│ 处理器层        │
│ RecheckPriority │     │ (async worker)  │     │ Processor       │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   通知系统      │◀────│ 数据存储层      │◀────│ 内容抓取器      │
│ Apprise/Queue   │     │ Watch Model     │     │ Fetcher         │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### 1.2 关键数据传递路径

**Worker处理流程** (`changedetectionio/worker.py:46-500`)：

1. **获取任务**：从 `RecheckPriorityQueue` 队列获取待检查Watch UUID
2. **处理器初始化**：根据Watch配置选择处理器类型
   ```python
   processor = watch.get('processor', 'text_json_diff')  # 默认文本处理器
   processor_module = get_processor_module(processor)
   update_handler = processor_module.perform_site_check(datastore, watch_uuid)
   ```
3. **内容抓取**：调用 `await update_handler.call_browser()` 异步执行抓取
4. **变更检测**：在线程池中执行 `run_changedetection()` 避免阻塞事件循环
5. **结果返回**：元组 `(changed_detected, update_obj, contents)`

**Fetcher到Processor的传递** (`processors/base.py:117-260`)：

- `self.fetcher` 实例在 `call_browser()` 中创建
- Fetcher执行后设置属性：`.content`, `.headers`, `.status_code`, `.screenshot`, `.xpath_data`
- 异常通过自定义异常类向上传递：`Non200ErrorCodeReceived`, `EmptyReply`, `checksumFromPreviousCheckWasTheSame` 等

**Processor到Watch的传递**：
- `run_changedetection()` 返回 `(changed_detected, update_obj, contents)`
- 调用 `datastore.update_watch(uuid, update_obj)` 更新元数据
- 调用 `watch.save_history_text()` / `watch.save_screenshot()` 持久化历史

**Watch到Notification的传递**：
- 变更检测通过后，通知任务存入通知队列
- 通知处理器异步发送到配置的Apprise通知渠道

---

## 二、差异数据格式化为前端可展示的形式

### 2.1 Placemarker标记机制

**核心常量定义** (`diff/__init__.py:33-58`)：

| Placemarker常量 | 替换后HTML | 用途 |
|----------------|-----------|------|
| `REMOVED_PLACEMARKER_OPEN/CLOSED` | `<span style="background-color: #fadad7; color: #b30000;">` | 标记删除内容 |
| `ADDED_PLACEMARKER_OPEN/CLOSED` | `<span style="background-color: #eaf2c2; color: #406619;">` | 标记新增内容 |
| `CHANGED_PLACEMARKER_OPEN/CLOSED` | 同上removed样式 | 标记被替换的旧内容 |
| `CHANGED_INTO_PLACEMARKER_OPEN/CLOSED` | 同上added样式 | 标记替换后的新内容 |

**标记替换实现** (`notification/handler.py:86-101`)：

```python
def apply_html_color_to_body(n_body: str):
    # 执行占位符到HTML样式的替换
    n_body = n_body.replace(REMOVED_PLACEMARKER_OPEN, '<span style="...">')
    # ... 其他标记替换
    return n_body
```

### 2.2 词级与行级差异渲染

**词级差异模式** (`render_inline_word_diff()`):
- 使用 diff-match-patch 库的 `linesToChars` 技术
- 支持整行替换模式和内嵌变更模式
- Placemarker标记而非直接HTML，支持多格式输出

**渲染管道流程**:
```
原始文本比较 → difflib.SequenceMatcher → 
    Placemarker标记插入 → build_diff_cell_visualizer() →
        apply_html_color_to_body() → Jinja模板渲染 → 前端展示
```

**差异单元格可视化** (`processors/text_json_diff/difference.py:24-91`):
- 构建100个单元格的文档概览
- 基于字符位置计算变更所在单元格
- 单元格状态：删除、新增、混合、无变更
- 为长文档提供快速视觉导航

### 2.3 前端渲染的最终输出

**HTML输出结构**：
```html
<!-- 已删除内容 -->
<span style="background-color: #fadad7; color: #b30000;" role="deletion">
    已删除的文本内容
</span>

<!-- 新增内容 -->
<span style="background-color: #eaf2c2; color: #406619;" role="insertion">
    新增的文本内容
</span>
```

---

## 三、预览功能在不同抓取模式下的行为差异

### 3.1 Fetcher能力标志定义（证据）

**Fetcher基类默认值** (`content_fetchers/base.py:69-76`)：
```python
class Fetcher():
    supports_browser_steps = False      # 默认不支持浏览器步骤
    supports_screenshots = False        # 默认不支持页面截图
    supports_xpath_element_data = False # 默认不支持视觉选择器数据
    lock_viewport_elements = False      # 默认禁用视口元素锁定
```

**各Fetcher实际能力标志**：

| 能力标志 | Requests | Playwright | Puppeteer | Selenium |
|---------|----------|------------|-----------|----------|
| `supports_screenshots` | ❌ `False` | ✅ `True` | ✅ `True` | ✅ `True` |
| `supports_browser_steps` | ❌ `False` | ✅ `True` | ✅ `True` | ❌ `False` |
| `supports_xpath_element_data` | ❌ `False` | ✅ `True` | ✅ `True` | ❌ `False` |
| **代码位置** | `requests.py` | `playwright.py:170-172` | `puppeteer.py:184-185` | `webdriver_selenium.py:18-19` |

### 3.2 Requests Fetcher（基础HTTP客户端）的预览行为

**核心代码证据** (`content_fetchers/requests.py:203-207`)：
```python
# 如果内容是图像，将其设置为截图用于SSIM/视觉比较
content_type = r.headers.get('content-type', '').lower()
if 'image/' in content_type:
    self.screenshot = r.content
    logger.debug(f"检测到图像内容({content_type})，设置为截图用于比较")
```

**行为特征**（证据支持）：
1. **文本模式为主**：默认只显示处理后的纯文本内容
2. **有条件的图像支持**：仅当HTTP响应 `Content-Type` 包含 `image/` 时才设置 `screenshot` 属性
3. **无截图能力标志**：`supports_screenshots` 仍为 `False`（此能力标志仅表示浏览器级截图能力）
4. **快速响应**：直接读取历史快照文件，无需浏览器操作
5. **过滤高亮**：支持触发文本、忽略文本的行级高亮
6. **无浏览器步骤支持**：配置浏览器步骤会抛出 `BrowserStepsInUnsupportedFetcher` 异常

**适用场景**：
- 纯JSON/XML API监控
- 静态HTML页面
- 直接图片URL监控（特殊路径）
- 高频率监控需求

### 3.3 Playwright/Puppeteer Fetcher（浏览器客户端）的预览行为

**核心能力特征**（证据支持）：
1. **完整截图支持**：
   - 支持全页面截图捕获
   - JPEG/PNG格式可配置
   - 分块拼接机制处理超高页面

2. **视觉选择器数据**：
   - 捕获元素位置和尺寸数据（`xpath_data`）
   - 支持可视化CSS选择器构建
   - 数据以JSON格式存储

3. **浏览器步骤支持**：
   - 步骤执行过程截图
   - 错误步骤截图保留
   - 步骤HTML快照保存

4. **视口元素锁定机制**：
   - **默认状态**：`lock_viewport_elements = False`（禁用）
   - **触发条件**：需要通过kwargs显式传入才能启用
   - **代码证据**：`base.py:83-84` 仅当kwargs包含此参数时才设置
   - **重要发现**：截至当前代码，**没有任何地方将此参数设置为True**
   - 虽然注释提到 "Only enabled for image_ssim_diff processor"，但实际代码路径中未启用

5. **错误可视化**：
   - 错误截图独立保存（`as_error=True`）
   - 错误文本快照
   - XPath错误数据

### 3.4 截图分块与高度限制配置（精确值）

**常量定义证据** (`content_fetchers/__init__.py:13-26`)：
```python
# 最大截图高度 - 可通过环境变量 SCREENSHOT_MAX_HEIGHT 覆盖
SCREENSHOT_MAX_HEIGHT_DEFAULT = 20000
SCREENSHOT_MAX_TOTAL_HEIGHT = int(os.getenv("SCREENSHOT_MAX_HEIGHT", 20000))

# 分块阈值 - 超过此高度才启用分块拼接模式
# 可通过环境变量 SCREENSHOT_CHUNK_HEIGHT 覆盖
SCREENSHOT_SIZE_STITCH_THRESHOLD = int(os.getenv("SCREENSHOT_CHUNK_HEIGHT", 10000))

# 默认JPEG质量
SCREENSHOT_DEFAULT_QUALITY = 40
```

**配置值总结表**：

| 配置项 | 默认值 | 环境变量覆盖 | 说明 |
|-------|--------|-------------|------|
| 最大截图总高度 | **20000px** | `SCREENSHOT_MAX_HEIGHT` | 超过此高度的页面会被截断 |
| 分块阈值高度 | **10000px** | `SCREENSHOT_CHUNK_HEIGHT` | 超过此高度才启用分块拼接 |
| 默认JPEG质量 | **40** | 无 | 截图压缩质量 |

**分块拼接触发条件**（双重条件）：
1. 页面高度 > `SCREENSHOT_SIZE_STITCH_THRESHOLD` (10000px) **且**
2. 页面高度 > 视口高度

**分块实现细节** (`playwright.py:16-149`)：
- 分块高度使用 `min(SCREENSHOT_SIZE_STITCH_THRESHOLD, SCREENSHOT_MAX_TOTAL_HEIGHT)`
- 分块通过独立进程 `spawn` 进行PIL拼接，防止内存泄漏
- 超过最大高度时会在截图顶部添加警告文本

### 3.5 处理器特定的预览行为差异

**预览路由分发逻辑** (`blueprint/ui/preview.py:14-56`)：
```python
def preview_page(uuid):
    processor_name = watch.get('processor', 'text_json_diff')
    processor_module = get_processor_submodule(processor_name, 'preview')
    
    if processor_module and hasattr(processor_module, 'render'):
        return processor_module.render(...)  # 处理器自定义预览
    
    # 降级到默认文本预览
    content = watch.get_history_snapshot(timestamp)
    return render_template("preview.html", ...)
```

**不同处理器的预览实现对比**：

| 处理器类型 | 自定义预览模块 | 预览内容类型 | 资源服务方式 |
|-----------|---------------|-------------|------------|
| `text_json_diff` | ❌ 无 | 纯文本高亮 + 条件截图 | 默认模板 |
| `image_ssim_diff` | ✅ 有 | 图像像素比较 + 差异热图 | 自定义asset路由 |
| `restock_diff` | ❌ 无 | 价格库存数据表格 | 默认模板 |

**图像处理器预览的特殊行为**：
- 自定义 `preview.py` 渲染器
- 提供像素级差异可视化和差异热图
- 通过 `processor-asset` 子路由流式传输二进制图像数据
- 避免大二进制数据嵌入HTML导致内存问题

### 3.6 前端预览条件分支逻辑

**截图显示条件** (`preview.html:84`):
```html
{% if capabilities.supports_screenshots %}
    <!-- 截图显示区域 -->
    <div class="snapshot-age">{{ watch.snapshot_screenshot_ctime|format_timestamp_timeago }}</div>
    <img style="max-width: 80%" id="screenshot-img" alt="当前截图">
{% else %}
    <strong>需要支持截图的Content Fetcher（如浏览器模式）才能显示截图。</strong>
{% endif %}
```

**关键条件分支**：
- 基于 `capabilities.supports_screenshots` 标志决定是否显示截图区域
- 即使Requests fetcher在图像URL时实际有截图，也不会通过此条件显示
- 图像处理器的自定义预览绕过了此条件检查

---

## 四、关键设计发现与待澄清点

### 4.1 已证实的设计特征

1. **Requests Fetcher的图像特殊处理**：
   - 仅当HTTP Content-Type是 `image/*` 时才设置screenshot属性
   - 这是为了支持直接图像URL的视觉比较
   - 但 `supports_screenshots` 能力标志始终为False

2. **lock_viewport_elements功能状态**：
   - 代码框架完整（注入/解锁JS脚本、条件分支）
   - 但**没有任何实际调用路径将此参数设置为True**
   - 目前此功能实际上是**未启用状态**
   - 可能是预留功能或需要手动配置启用

3. **能力标志与实际行为的不一致**：
   - `supports_screenshots` 是静态类属性
   - Requests fetcher在特定条件下实际有截图能力，但能力标志为False
   - 这可能导致预览页面在图像URL时无法显示截图（取决于前端判断逻辑）

### 4.2 潜在设计问题

1. **能力标志粒度不足**：
   - 静态布尔标志无法表达"有条件支持"的场景
   - 图像URL场景是Requests fetcher的例外情况

2. **代码注释与实际实现的差距**：
   - 注释提到"image_ssim_diff processor"会启用元素锁定
   - 但代码路径中找不到此设置的实际实现

---

## 五、总结与关键数据结构

### 5.1 Fetcher能力对比总表

| 特性 | Requests | Playwright | Puppeteer | Selenium |
|------|----------|------------|-----------|----------|
| 页面截图 | ⚠️ 仅图像URL | ✅ 完整支持 | ✅ 完整支持 | ✅ 完整支持 |
| 浏览器步骤 | ❌ 不支持 | ✅ 完整支持 | ✅ 完整支持 | ❌ 不支持 |
| 视觉选择器 | ❌ 不支持 | ✅ 完整支持 | ✅ 完整支持 | ❌ 不支持 |
| 视口元素锁定 | ❌ 不支持 | ⚠️ 代码存在但未启用 | ⚠️ 代码存在但未启用 | ❌ 不支持 |
| JS执行 | ❌ 不支持 | ✅ 完整支持 | ✅ 完整支持 | ✅ 支持 |
| 分块截图 | ❌ 不适用 | ✅ >10000px时分块 | ✅ >10000px时分块 | ❌ 不支持 |
| 异步执行 | ✅ 线程池包装 | ✅ 原生async | ✅ 原生async | ✅ 支持 |

### 5.2 截图配置参数总表

| 参数 | 默认值 | 环境变量 | 说明 |
|------|--------|---------|------|
| 最大总高度 | **20000px** | `SCREENSHOT_MAX_HEIGHT` | 超过会被截断并显示警告 |
| 分块阈值 | **10000px** | `SCREENSHOT_CHUNK_HEIGHT` | 超过此高度才启用分块 |
| 默认质量 | **40** | 无 | JPEG压缩质量 |

### 5.3 数据流向关键节点

```
抓取开始 → Fetcher.run()
    ↓
    ├─ 设置 .content (文本/哈希)
    ├─ 设置 .headers (HTTP头)
    ├─ 设置 .status_code (HTTP状态)
    ├─ [条件] 设置 .screenshot (图像响应/浏览器截图)
    └─ [条件] 设置 .xpath_data (视觉选择器)
    ↓
Processor.run_changedetection()
    ↓
    ├─ 变更检测算法执行
    ├─ 生成 update_obj 元数据字典
    └─ 准备通知内容
    ↓
Watch模型持久化
    ↓
    ├─ save_history_text() - 文本快照
    ├─ save_screenshot() - 截图文件
    └─ save_xpath_data() - 视觉选择器JSON
    ↓
通知队列 → Apprise发送
```

---

**文档生成时间**：2026年5月16日  
**分析版本**：changedetection.io 当前主分支（基于代码证据）  
**分析范围**：内容抓取、差异检测、预览功能、Fetcher能力边界
