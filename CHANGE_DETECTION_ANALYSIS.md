# 变更检测系统架构分析报告

## 概述

本文档分析了 changedetection.io 项目中内容抓取、差异检测、数据格式化和预览功能的实现机制。重点关注数据流向、模块交互和不同抓取模式下的行为差异。

---

## 一、内容抓取层向通知与存储模块传递结果的路径

### 1.1 整体数据流程架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   任务队列      │────▶│  Worker 处理层  │────▶│  处理器层      │
│ (PriorityQueue) │     │ (async worker)  │     │ (Processor)    │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   通知系统      │◀────│  数据存储层     │◀────│  内容抓取器    │
│ (Apprise)       │     │ (Watch Model)   │     │ (Fetcher)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### 1.2 任务队列与Worker处理流程

**队列机制** (`changedetectionio/queue_handlers.py`):
- 使用 `RecheckPriorityQueue` 优先级队列
- 支持异步/同步双接口
- 任务基于 `PrioritizedItem` 进行优先级排序
- 关键方法: `async_get()`, `async_put()` 使用线程池执行器

**Worker处理流程** (`changedetectionio/worker.py:46-500`):

1. **获取任务**: 从队列中获取待检查的Watch UUID
2. **初始化处理器**: 根据Watch配置选择处理器
   ```python
   processor = watch.get('processor', 'text_json_diff')
   processor_module = get_processor_module(processor)
   update_handler = processor_module.perform_site_check(datastore, watch_uuid)
   ```
3. **内容抓取**: 调用 `await update_handler.call_browser()` 异步抓取
4. **变更检测**: 在线程池中运行 `run_changedetection()` 避免阻塞事件循环
5. **结果处理**: 
   - `changed_detected` 布尔值标记是否有变更
   - `update_obj` 包含更新元数据
   - `contents` 为处理后的文本内容

### 1.3 结果传递路径详解

```
内容抓取 → 处理器层 → Watch模型 → 存储/通知
    │          │            │         │
    │          │            │         └── 发送通知到队列
    │          │            └───────────── 保存历史快照
    │          └────────────────────────── 执行差异检测
    └───────────────────────────────────── 获取页面内容
```

**关键传递点**:

1. **从Fetcher到Processor**:
   - `call_browser()` 调用具体fetcher执行抓取
   - Fetcher设置 `self.content`, `self.headers`, `self.status_code` 等属性
   - 异常通过自定义异常类向上传递 (`Non200ErrorCodeReceived`, `EmptyReply` 等)

2. **从Processor到Watch模型**:
   - `run_changedetection()` 返回 `(changed_detected, update_obj, contents)`
   - 调用 `datastore.update_watch(uuid, update_obj)` 更新元数据
   - 调用 `watch.save_history_text()` 保存历史快照

3. **从Watch到Notification**:
   - 变更检测通过后将通知任务放入 `notification_q`
   - 通知处理器异步发送到配置的通知渠道

---

## 二、差异数据格式化为前端可展示的形式

### 2.1 差异渲染核心流程

```
历史快照对比 → Placemarker标记 → 样式应用 → 前端渲染
    │              │               │            │
    │              │               │            └── diff.html模板
    │              │               └────────────── apply_html_color_to_body()
    │              └────────────────────────────── render_diff()
    └──────────────────────────────────────────── get_history_snapshot()
```

### 2.2 Placemarker标记机制

**核心标记常量** (`changedetectionio/diff/__init__.py:33-43`):

| 标记常量 | 用途 |
|---------|------|
| `REMOVED_PLACEMARKER_OPEN/CLOSED` | 标记删除内容 |
| `ADDED_PLACEMARKER_OPEN/CLOSED` | 标记新增内容 |
| `CHANGED_PLACEMARKER_OPEN/CLOSED` | 标记被替换的旧内容 |
| `CHANGED_INTO_PLACEMARKER_OPEN/CLOSED` | 标记替换后的新内容 |

**标记替换逻辑** (`changedetectionio/notification/handler.py:86-101`):

```python
def apply_html_color_to_body(n_body: str):
    n_body = n_body.replace(REMOVED_PLACEMARKER_OPEN,
                            f'<span style="{HTML_REMOVED_STYLE}" role="deletion" aria-label="Removed text" title="Removed text">')
    n_body = n_body.replace(REMOVED_PLACEMARKER_CLOSED, f'</span>')
    # ... 其他标记替换
    return n_body
```

### 2.3 差异渲染引擎

**核心函数** `render_diff()` (`changedetectionio/diff/__init__.py:424-507`):

**参数配置**:
- `include_equal`: 是否包含未变更行
- `include_removed/added/replaced`: 控制显示变更类型
- `word_diff`: 是否启用词级差异检测
- `ignore_junk`: 是否忽略空白字符变化
- `context_lines`: 上下文行数

**实现机制**:
1. 使用 `difflib.SequenceMatcher` 进行行级对比
2. 启用 `word_diff` 时使用 `diff-match-patch` 库进行词级对比
3. 通过 `render_inline_word_diff()` 实现内嵌高亮
4. Placemarker标记而非直接HTML，支持多格式输出

**词级差异渲染** (`render_inline_word_diff()`):
- 使用 `diff-match-patch` 的 `diff_linesToChars` 技术
- 将词映射为字符，执行高效差异检测
- 支持整行替换和内嵌变更两种模式

### 2.4 前端可视化增强

**差异单元格可视化** (`changedetectionio/processors/text_json_diff/difference.py:24-91`):

- 构建100个单元格的概览网格
- 基于字符位置计算变更所在单元格
- 支持三种状态: `deletion`, `insertion`, `mixed`
- 为长文档提供快速视觉导航

**渲染管道**:
```
原始文本 → diff.render_diff() → Placemarker标记 →
    build_diff_cell_visualizer() → 单元格数据 →
        apply_html_color_to_body() → HTML格式化 →
            Jinja模板渲染 → 前端显示
```

---

## 三、预览功能在不同抓取模式下的行为差异

### 3.1 抓取器(Fetcher)能力对比

**Fetcher基类能力标志** (`changedetectionio/content_fetchers/base.py:69-71`):

| 能力标志 | 说明 |
|---------|------|
| `supports_browser_steps` | 支持浏览器自动化步骤 |
| `supports_screenshots` | 支持页面截图 |
| `supports_xpath_element_data` | 支持视觉选择器数据 |

**主要Fetcher实现对比**:

| 特性 | Requests Fetcher | Playwright Fetcher |
|------|------------------|-------------------|
| **基础信息** | `requests.py:16` | `playwright.py` |
| **描述** | Basic fast Plaintext/HTTP Client | 全功能Chrome浏览器 |
| **截图支持** | ❌ 不支持（除非响应是图片） | ✅ 完整支持 |
| **浏览器步骤** | ❌ 不支持（抛出异常） | ✅ 完整支持 |
| **XPath数据** | ❌ 不支持 | ✅ 完整支持 |
| **JS执行** | ❌ 不支持 | ✅ 完整支持 |
| **代理支持** | ✅ 支持 | ✅ 支持 |
| **异步执行** | ✅ 线程池包装 | ✅ 原生async |
| **性能** | ⚡ 快速 | 🐢 较慢但功能完整 |

### 3.2 预览功能实现机制

**处理器感知的预览路由** (`changedetectionio/blueprint/ui/preview.py:14-56`):

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

**Processor资产服务路由** (`preview.py:127-186`):
- 支持处理器特定二进制资源（截图、文件等）
- 通过 `/preview/<uuid>/processor-asset/<asset_name>` 访问
- 解决大二进制数据内存问题，单独HTTP响应流式传输

### 3.3 不同抓取模式下预览行为差异

#### 3.3.1 Requests Fetcher (Basic HTTP Client)

**行为特征**:
1. **仅文本预览**: 仅显示处理后的纯文本内容
2. **无截图支持**: `is_html_webdriver = False`，截图区域隐藏
3. **快速响应**: 直接读取历史快照文件，无需浏览器操作
4. **过滤高亮**: 支持触发文本、忽略文本的行级高亮
5. **无浏览器步骤**: 不支持浏览器步骤相关的预览功能

**适用场景**:
- 纯JSON/XML API监控
- 静态HTML页面
- 高频率监控需求

#### 3.3.2 Playwright Fetcher (Chrome Browser)

**行为特征**:
1. **完整截图支持**: 
   - 全页面截图捕获（分块拼接技术）
   - 截图质量可配置: JPEG/PNG，质量参数
   - 高度限制: `SCREENSHOT_MAX_TOTAL_HEIGHT`
   
2. **视觉选择器数据**:
   - 捕获元素位置和尺寸数据
   - 支持可视化CSS选择器构建
   - `xpath_data` 存储元素元数据

3. **浏览器步骤支持**:
   - 步骤执行过程截图
   - 错误步骤截图保留
   - 步骤HTML快照保存

4. **视口元素锁定**:
   - 截图时锁定固定元素尺寸（header/ad等）
   - 防止布局偏移影响视觉对比
   - 仅在 `lock_viewport_elements=True` 时启用

5. **错误可视化**:
   - 错误截图独立保存 (`as_error=True`)
   - 错误文本快照
   - XPath错误数据

#### 3.3.3 处理器特定预览行为

**文本/JSON处理器** (`text_json_diff`):
- 默认预览实现
- 文本语法高亮
- 过滤/触发标记
- 行号显示

**图像对比处理器** (`image_ssim_diff`):
- 自定义预览模块
- 图像差异可视化
- 基于SSIM算法的像素级对比
- 通过processor-asset路由提供原始图像

**库存监控处理器** (`restock_diff`):
- 价格趋势图表
- 库存状态时间线
- 产品元数据展示

### 3.4 截图捕获技术细节

**Playwright分块截图** (`playwright.py:16-150`):

1. **分块策略**:
   - 单块高度: `SCREENSHOT_SIZE_STITCH_THRESHOLD` (8192px)
   - 最大总高度: `SCREENSHOT_MAX_TOTAL_HEIGHT` (16384px)
   - 动态调整视口大小

2. **内存管理**:
   - 使用独立进程(`spawn`)进行图像拼接
   - PIL内存泄漏问题通过子进程退出解决
   - 分块捕获降低单张内存压力

3. **元素锁定**:
   - `res/lock-elements-sizing.js` 注入固定尺寸
   - 防止滚动/重渲染导致布局变化
   - 截图完成后 `unlock-elements-sizing.js` 恢复

---

## 四、关键数据结构与设计模式

### 4.1 Watch模型数据流转

**历史快照存储**:
```python
# Watch模型核心方法
watch.get_history_snapshot(timestamp)  # 获取指定时间快照
watch.save_history_text(contents)      # 保存文本快照
watch.save_screenshot(screenshot)      # 保存截图
watch.save_xpath_data(data)            # 保存XPath元数据
```

**快照压缩策略** (`Watch.py:54-133`):
- Brotli流式压缩: 大于 `SNAPSHOT_BROTLI_COMPRESSION_THRESHOLD` (20KB)
- 分块处理降低内存峰值
- C级内存强制回收

### 4.2 异常处理层次

```
Fetcher层异常 → Processor层捕获 → Watch状态更新
    │              │                     │
    │              │                     └── last_error字段
    │              └────────────────────────── 错误截图保存
    └────────────────────────────────────────── HTTP状态、重试逻辑
```

**自定义异常类型** (`content_fetchers/exceptions.py`):
- `Non200ErrorCodeReceived`: HTTP错误码
- `EmptyReply`: 空响应
- `BrowserStepsStepException`: 浏览器步骤失败
- `FilterNotFoundInResponse`: 过滤器未找到
- `ScreenshotUnavailable`: 截图不可用

### 4.3 处理器插件架构

**核心接口** (`changedetectionio/processors/__init__.py`):
- `get_processor_module()`: 获取处理器主模块
- `get_processor_submodule()`: 获取子模块(preview, difference)
- 动态导入支持内置和插件处理器

**子模块约定**:
- `processor.py`: 核心变更检测逻辑
- `preview.py`: 预览页面渲染
- `difference.py`: 历史对比页面
- `forms.py`: 配置表单定义

---

## 五、性能优化与可扩展性

### 5.1 性能优化点

1. **异步架构**:
   - Worker异步处理，避免阻塞事件循环
   - CPU密集型操作在线程池中执行
   - 队列支持高并发场景

2. **内存管理**:
   - 截图拼接使用独立进程防止内存泄漏
   - Brotli压缩减少存储占用
   - 显式GC和C级内存回收

3. **差异算法**:
   - diff-match-patch词级对比效率
   - Placemarker标记避免重复HTML转义
   - 上下文行数配置控制输出大小

### 5.2 扩展点

1. **自定义Fetcher**: 继承 `Fetcher` 基类
2. **自定义Processor**: 实现处理器接口
3. **通知插件**: Apprise框架支持
4. **条件插件**: 自定义变更过滤规则

---

## 六、总结

### 核心发现

1. **数据流清晰**: 从抓取到通知经过明确的模块边界，异常处理分层完善
2. **Placemarker设计**: 中间标记格式支持多输出场景，解耦检测和渲染
3. **处理器感知UI**: 预览和差异页面支持处理器自定义，架构灵活
4. **Fetcher能力分层**: 不同抓取器明确能力边界，通过标志位控制UI行为

### 关键设计优势

- **异步优先**: 全链路异步设计，支持高并发
- **内存敏感**: 大对象处理特别关注内存泄漏问题
- **插件友好**: 处理器、抓取器、通知系统均支持扩展
- **视觉增强**: 差异可视化、单元格概览等提升用户体验

### 未来改进方向

1. 更多处理器特定预览实现
2. 差异渲染性能优化（大文档）
3. 截图对比的增量存储策略
4. 实时变更推送的进一步优化

---

**文档生成时间**: 2026年  
**分析版本**: changedetection.io 当前主分支  
**分析范围**: 内容抓取、差异检测、预览功能
