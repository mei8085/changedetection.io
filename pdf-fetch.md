# PDF 及非 HTML 页面抓取完整处理路径分析

## 1. 概述

当系统抓取到 PDF 或浏览器无法常规渲染的页面时，会触发一条特殊的处理链路。这条链路在内容类型识别、抓取器选择、内容预处理等关键点上与普通 HTML 抓取分叉，但在差异计算和通知阶段重新汇入主流程。

系统主要处理三类非标准页面：
1. **PDF 文档**：需要专用工具转换为 HTML
2. **source: 原始内容**：用户明确要求获取原始响应
3. **非 HTML 文本格式**：纯文本、CSV、YAML、XML、JSON 等
4. **空/无文本页面**：有响应但无有效文本内容

## 2. 内容类型识别机制

### 2.1 第一层识别：Watch 类型属性

#### 2.1.1 Watch.is_pdf 属性

**位置**：`changedetectionio/model/Watch.py:411-421`

```python
@property
def is_pdf(self):
    url = str(self.get("url") or "").lower()
    content_type = str(self.get("content-type") or "").lower()
    return (
            url.endswith(".pdf")
            or content_type.split(";")[0].strip() == "application/pdf"
    )
```

**识别逻辑**：
- 通过 URL 后缀判断：`.pdf` 结尾
- 通过 HTTP `Content-Type` 头判断：`application/pdf`

#### 2.1.2 Watch.is_source_type_url 属性

**位置**：`changedetectionio/model/Watch.py:353-354`

```python
@property
def is_source_type_url(self):
    return self.get('url', '').startswith('source:')
```

**触发场景**：
- 用户在 URL 前手动添加 `source:` 前缀，明确要求获取原始响应
- 典型用法：`source:https://api.example.com/data.json`
- 系统会跳过 HTML → 文本转换，直接对原始内容进行变更检测

#### 2.1.3 浏览器抓取失败信号

浏览器抓取器（Playwright/Puppeteer/Selenium）可能在以下情况下抛出异常，表明页面无法被浏览器常规渲染：

| 异常类型 | 触发条件 | 代码位置 |
|---------|---------|----------|
| `EmptyReply` | 页面内容完全为空（`await page.content()` 返回空字符串） | `playwright.py:359-363` |
| `PageUnloadable` | 页面加载超时、JS 执行致命错误、导航失败 | `playwright.py:328-332` |
| `Non200ErrorCodeReceived` | HTTP 状态码非 200（403/404/500 等） | `playwright.py:354-357` |
| `BrowserStepsStepException` | 浏览器自动化步骤执行失败 | `playwright.py:369-373` |

**重要**：浏览器抓取失败后**没有自动回退机制**。抓取器在 `call_browser()` 阶段被选定后就不会改变。用户必须手动将抓取器切换为 `html_requests` 才能获取原始响应内容。

### 2.2 第二层识别：guess_stream_type 智能检测

**位置**：`changedetectionio/processors/magic.py`

当 HTTP 头缺失或不可靠时，系统会使用 `guess_stream_type` 类进行更深入的检测：

1. **puremagic 库分析**：对文件前 200 字节进行 MIME 类型检测
2. **内容模式匹配**：检查 `%PDF-1` 魔数签名
3. **HTTP 头信任链**：优先信任 HTTP 头，其次是 magic 检测，最后是内容模式

**PDF 检测关键点** (`magic.py:110-111, 128-129`)：
```python
elif 'pdf' in magic_content_header:
    self.is_pdf = True
# ...
elif '%pdf-1' in test_content:
    self.is_pdf = True
```

## 3. 抓取器选择与分叉点

### 3.1 普通 HTML 抓取路径

普通 HTML 页面根据配置选择抓取器：
- `html_requests`：快速 HTTP 客户端
- `html_webdriver`：Playwright/Puppeteer/Selenium 浏览器渲染

### 3.2 PDF 抓取路径分叉

**分叉点 1：强制使用 html_requests 抓取器**

**位置**：`changedetectionio/processors/base.py:157-158`

```python
# PDF should be html_requests because playwright will serve it up (so far) in a embedded page
if self.watch.is_pdf:
    prefer_fetch_backend = "html_requests"
```

**原因**：浏览器渲染器会将 PDF 嵌入到 HTML 页面中（使用 `<embed>` 或 `<iframe>`），导致无法直接获取 PDF 原始字节。

**分叉点 2：启用二进制模式**

**位置**：`changedetectionio/processors/base.py:239`

```python
# Requests for PDF's, images etc should be passwd the is_binary flag
is_binary = self.watch.is_pdf
```

**二进制模式的影响** (`content_fetchers/requests.py:195-197`)：
```python
if is_binary:
    # Binary files just return their checksum until we add something smarter
    self.content = hashlib.md5(r.content).hexdigest()
else:
    self.content = r.text
self.raw_content = r.content  # 原始字节始终保存
```

**注意**：虽然 `self.content` 被设置为 MD5 哈希，但 `self.raw_content` 保存了完整的原始字节，供后续 PDF 转 HTML 使用。

### 3.3 source: 原始内容路径

**分叉点：跳过 HTML → 文本转换**

**位置**：`changedetectionio/processors/text_json_diff/processor.py:510-511`

```python
if watch.is_source_type_url:
    # For source URLs, keep raw content
    stripped_text = html_content
```

**特性**：
- 使用用户配置的抓取器（可以是 html_requests 或 html_webdriver）
- 不进行 HTML 到文本的转换，直接使用原始响应内容
- 过滤器仍然生效（subtractive_selectors、include_filters）
- 适用于监控 API 响应、原始数据文件等

### 3.4 非 HTML 文本格式路径

对于纯文本、CSV、YAML、XML 等格式：
- 使用 `html_requests` 抓取器（大多数情况下）
- 不进行 HTML 混淆 workaround 处理
- 文本提取阶段直接保留原始格式（`is_plaintext` 标志）
- JSON 会进行格式化和排序，避免因键顺序变化导致的误报

### 3.5 空/无文本页面的回退机制

**配置选项**：`empty_pages_are_a_change`（全局设置）

| 配置值 | 行为 | 适用场景 |
|-------|------|---------|
| `False`（默认） | 抛出 `ReplyWithContentButNoText` 异常，不记录变更 | 正常网页监控，避免误报 |
| `True` | 将空内容视为有效内容，继续进行变更检测 | 监控页面是否消失、API 是否返回空响应 |

**异常处理流程** (`worker.py:214-236`)：
1. 捕获 `ReplyWithContentButNoText` 异常
2. 检查过滤器是否仅匹配到图片等非文本元素
3. 更新 watch 的 `last_error` 字段，提供用户友好的错误提示
4. 保存截图和 XPath 数据（如果有），帮助用户调试
5. 跳过本次变更检测

### 3.6 浏览器无法常规渲染但非 PDF 的完整处理链路

#### 3.6.1 触发该分支的判定信号

**定义**：用户配置了浏览器抓取器（`html_webdriver`/Playwright），但目标 URL 返回的内容无法被浏览器正常渲染为有意义的文本页面。

**判定信号矩阵**：

| 判定阶段 | 判定信号 | 代码位置 |
|---------|---------|----------|
| 抓取前预判 | 不是 PDF（`watch.is_pdf == False`） | `Watch.py:411-421` |
| 抓取前预判 | 用户配置为浏览器抓取器（`fetch_backend` 为 `html_webdriver` 或 `system` 且全局默认是浏览器） | `base.py:133-141` |
| 内容类型检测 | `guess_stream_type` 检测结果为非 HTML：<br>- `is_plaintext = True`<br>- `is_json = True`<br>- `is_csv = True`<br>- `is_yaml = True`<br>- `is_xml = True` | `magic.py:50-138` |
| 文本提取后 | 过滤和提取后文本为空（`ReplyWithContentButNoText`） | `processor.py:542-550` |

**典型场景**：
1. **纯文本文件**：`.txt`、`.log`、`.md` 等
2. **结构化数据**：JSON API、CSV 数据、YAML 配置、XML 文档
3. **二进制文件**：图片、音频、视频等（无文本内容）
4. **SPA 渲染失败**：Angular/React 应用未正确渲染，仅返回空骨架

#### 3.6.2 抓取器选择：无自动回退，按用户配置执行

**重要说明**：系统**没有**"浏览器抓取失败后自动回退到 html_requests"的机制。抓取器在抓取前就已确定。

**抓取器选择逻辑** (`base.py:133-174`)：

```python
# 1. 获取用户配置的抓取器
prefer_fetch_backend = self.watch.get('fetch_backend', 'system')

# 2. 如果是 'system'，使用全局默认
if not prefer_fetch_backend or prefer_fetch_backend == 'system':
    prefer_fetch_backend = self.datastore.data['settings']['application'].get('fetch_backend')

# 3. 仅 PDF 会强制切换到 html_requests
if self.watch.is_pdf:
    prefer_fetch_backend = "html_requests"

# 4. 其他情况按配置执行
if hasattr(content_fetchers, prefer_fetch_backend):
    fetcher_obj = getattr(content_fetchers, prefer_fetch_backend)
else:
    # 抓取器不存在时的默认回退
    fetcher_obj = getattr(content_fetchers, "html_requests")
```

**抓取器执行结果**：

| 内容类型 | 使用 html_webdriver 结果 | 使用 html_requests 结果 |
|---------|-------------------------|------------------------|
| 纯文本/CSV/YAML | 浏览器将文本包裹在 HTML 中（`<html><body>文本</body></html>`），需额外处理 | 直接获取原始文本，效率更高 |
| JSON | 浏览器可能渲染为交互式 JSON 查看器，文本提取不稳定 | 直接获取原始 JSON，可预测性强 |
| XML | 浏览器可能渲染为 XML 树视图 | 直接获取原始 XML |
| 二进制（图片/视频） | 浏览器显示媒体播放器，无可用文本 | 直接获取二进制数据，`is_binary=True` 时返回 MD5 |

**用户可选的回退方案**：
- 手动将该 watch 的 `fetch_backend` 改为 `html_requests`
- 使用 `source:` 前缀强制获取原始内容
- 配置 `empty_pages_are_a_change = True` 允许空内容进入检测流程

#### 3.6.3 文本抽取与过滤的实际执行方式

**位置**：`processor.py:461-521`

当 `guess_stream_type` 检测到非 HTML 内容时，文本抽取流程会跳过 HTML→文本转换：

```python
# === TEXT EXTRACTION ===
if watch.is_source_type_url:
    # For source URLs, keep raw content
    stripped_text = html_content
elif stream_content_type.is_plaintext:
    # For plaintext, keep as-is without HTML-to-text conversion
    stripped_text = html_content
else:
    # Extract text from HTML/RSS content (not generic XML)
    if stream_content_type.is_html or stream_content_type.is_rss:
        stripped_text = content_processor.extract_text_from_html(html_content, stream_content_type)
    else:
        stripped_text = html_content
```

**各类型的具体执行路径**：

##### 路径 A：纯文本 (`is_plaintext = True`)

**触发条件** (`magic.py:98-99, 130-137`)：
- `Content-Type: text/plain`
- 或 `puremagic` 检测为 `text/plain` 且无 HTML 标签
- 或其他 `text/*` 类型但不是 HTML

**执行流程**：
1. **跳过 HTML 混淆 workaround** (`processor.py:487-488`)
   ```python
   if stream_content_type.is_html:
       content = html_tools.workarounds_for_obfuscations(content)
   ```
2. **过滤器应用** (`processor.py:502-507`)
   - 减法选择器：如果是纯文本，CSS 选择器不生效
   - 包含过滤器：CSS 选择器不生效，但 XPath 和正则提取可能仍有用
3. **跳过 HTML→文本转换** (`processor.py:513-515`)
   ```python
   elif stream_content_type.is_plaintext:
       stripped_text = html_content
   ```
4. **文本转换** (`processor.py:523-584`)
   - 空白修剪、去重、排序、行过滤、正则提取等全部正常执行

##### 路径 B：JSON (`is_json = True`)

**触发条件** (`magic.py:102-109, 119-120`)：
- `Content-Type: application/json` 等 JSON 类型
- 且不是 JSONP（`cb({...})` 格式）

**执行流程**：
1. **JSON 预处理** (`processor.py:481-483`)
   ```python
   if stream_content_type.is_json:
       if not filter_config.has_include_json_filters:
           content = content_processor.preprocess_json(raw_content=content)
   ```
   - 键排序：避免因键顺序变化导致的误报
   - 格式化：`json.dumps(..., sort_keys=True, indent=2)`
   - 如果用户配置了 `json:`/`jq:` 过滤器，则跳过此步骤
2. **跳过 HTML 混淆 workaround**
3. **过滤器应用**
   - `json:`/`jq:`/`jqraw:` 过滤器正常生效
   - CSS 选择器不生效
4. **跳过 HTML→文本转换** (`processor.py:520-521`)
   - 因为 `is_html = False` 且 `is_plaintext = False`
   - 直接使用 JSON 内容
5. **文本转换**：全部正常执行

##### 路径 C：空页面 (`ReplyWithContentButNoText`)

**触发条件** (`processor.py:542-550`)：
```python
if not stream_content_type.is_json and not empty_pages_are_a_change and len(stripped_text.strip()) == 0:
    raise content_fetchers.exceptions.ReplyWithContentButNoText(...)
```

**执行流程**：
1. 抛出异常，携带：URL、状态码、截图、是否有过滤器、HTML 内容、XPath 数据
2. `worker.py:214-236` 捕获异常
3. 生成用户友好的错误提示：
   ```python
   datastore.update_watch(uuid=uuid, update_obj={
       'last_error': f"Got HTML content but no text found (With {e.status_code} reply code){extra_help}"
   })
   ```
4. 保存截图和 XPath 数据作为错误快照
5. `process_changedetection_results = False`，跳过变更检测

**配置 `empty_pages_are_a_change = True` 时**：
- 不抛出异常
- 空文本继续进入校验和计算流程
- 如果与上一次内容不同（例如从有文本变为空），则检测为变更

#### 3.6.4 回接到 Diff 计算与通知触发

无论经过哪条路径，只要没有抛出异常，最终都会进入相同的变更检测流程：

**位置**：`processor.py:586-650`

```python
# === CHECKSUM CALCULATION ===
if text_for_checksuming is None:
    text_for_checksuming = stripped_text

# Calculate checksum
ignore_whitespace = self.datastore.data['settings']['application'].get('ignore_whitespace', False)
fetched_md5 = ChecksumCalculator.calculate(text_for_checksuming, ignore_whitespace=ignore_whitespace)

# === BLOCKING RULES EVALUATION ===
blocked = False
# Check trigger_text
if rule_engine.evaluate_trigger_text(text_for_checksuming, filter_config.trigger_text):
    blocked = True
# Check text_should_not_be_present
if rule_engine.evaluate_text_should_not_be_present(stripped_text, filter_config.text_should_not_be_present):
    blocked = True

# === CHANGE DETECTION ===
if watch.get('previous_md5') != fetched_md5 and not blocked:
    changed_detected = True
    update_obj['previous_md5'] = fetched_md5
else:
    changed_detected = False
    update_obj['previous_md5'] = fetched_md5
```

**通知触发** (`worker.py:566-569`)：
```python
if watch.history_n >= 2:
    logger.info(f"Change detected in UUID {uuid} - {watch['url']}")
    if not watch.get('notification_muted'):
        await send_content_changed_notification(uuid, notification_q, datastore)
```

## 4. 备用渲染策略：PDF 转 HTML

### 4.1 预处理触发条件

**位置**：`changedetectionio/processors/text_json_diff/processor.py:475-477`

```python
# PDF preprocessing
if watch.is_pdf or stream_content_type.is_pdf:
    content = content_processor.preprocess_pdf(raw_content=self.fetcher.raw_content)
    stream_content_type.is_html = True
```

**双保险触发**：
- `watch.is_pdf`：基于 URL/Content-Type 的预判
- `stream_content_type.is_pdf`：基于内容的智能检测

### 4.2 PDF 转 HTML 实现

**位置**：`changedetectionio/processors/text_json_diff/processor.py:292-318`

```python
def preprocess_pdf(self, raw_content):
    """Convert PDF to HTML using external tool."""
    from shutil import which
    tool = os.getenv("PDF_TO_HTML_TOOL", "pdftohtml")
    if not which(tool):
        raise PDFToHTMLToolNotFound(
            f"Command-line `{tool}` tool was not found in system PATH"
        )

    import subprocess
    proc = subprocess.Popen(
        [tool, '-stdout', '-', '-s', 'out.pdf', '-i'],
        stdout=subprocess.PIPE,
        stdin=subprocess.PIPE
    )
    proc.stdin.write(raw_content)
    proc.stdin.close()
    html_content = proc.stdout.read().decode('utf-8')
    proc.wait(timeout=60)

    # Add metadata for change detection
    metadata = (
        f"<p>Added by changedetection.io: Document checksum - "
        f"{hashlib.md5(raw_content).hexdigest().upper()} "
        f"Original file size - {len(raw_content)} bytes</p>"
    )
    return html_content.replace('</body>', metadata + '</body>')
```

**关键特性**：
- 使用外部工具 `pdftohtml`（可通过 `PDF_TO_HTML_TOOL` 环境变量自定义）
- 注入元数据：文档 MD5 校验和 + 文件大小，确保即使文本提取相同，文件本身变化也能被检测
- 转换后设置 `stream_content_type.is_html = True`，使后续流程视为普通 HTML 处理

## 5. 文本抽取流程（与 HTML 汇合）

PDF 转 HTML 后，处理流程与普通 HTML 完全一致：

### 5.1 过滤器应用

1. **减法选择器** (`subtractive_selectors`)：先移除不需要的元素
2. **包含过滤器** (`include_filters`)：CSS/XPath/JSON 过滤器提取目标内容
3. **特殊过滤器**：
   - XPath 过滤器：`/` 或 `xpath:` 前缀
   - JSON 过滤器：`json:` / `jq:` / `jqraw:` 前缀
   - CSS 选择器：默认

### 5.2 HTML 转文本

**位置**：`processor.py:386-394`

```python
def extract_text_from_html(self, html_content, stream_content_type):
    """Convert HTML to plain text."""
    do_anchor = self.datastore.data["settings"]["application"].get("render_anchor_tag_content", False)
    return html_tools.html_to_text(
        html_content=html_content,
        render_anchor_tag_content=do_anchor,
        is_rss=stream_content_type.is_rss
    )
```

### 5.3 文本转换操作

- 空白字符修剪 (`trim_text_whitespace`)
- 重复行删除 (`remove_duplicate_lines`)
- 字母排序 (`sort_text_alphabetically`)
- 行过滤 (`extract_lines_containing`)
- 正则提取 (`extract_text`)

## 6. Diff 计算与变更检测

### 6.1 校验和计算

**位置**：`processor.py:597-599`

```python
ignore_whitespace = self.datastore.data['settings']['application'].get('ignore_whitespace', False)
fetched_md5 = ChecksumCalculator.calculate(text_for_checksuming, ignore_whitespace=ignore_whitespace)
```

### 6.2 变更判定

```python
if watch.get('previous_md5') != fetched_md5:
    changed_detected = True
```

### 6.3 阻塞规则评估

在判定变更前，会评估一系列阻塞规则：
1. **触发文本** (`trigger_text`)：只有包含触发文本才报告变更
2. **禁止文本** (`text_should_not_be_present`)：包含禁止文本则不报告
3. **条件规则** (`conditions`)：自定义条件插件评估

## 7. 下游通知流程（所有分支汇合）

无论哪种内容类型，当检测到变更后，通知流程完全相同：

### 7.1 历史记录保存

**位置**：`worker.py:526-528`

```python
watch.save_history_blob(contents=contents,
                        timestamp=int(fetch_start_time),
                        snapshot_id=update_obj.get('previous_md5', 'none'))
```

**各分支的历史内容**：
- **普通 HTML**：提取后的纯文本
- **PDF 文档**：pdftohtml 转换后提取的纯文本
- **source: / 纯文本**：原始内容（经过过滤器和文本转换）
- **JSON**：格式化排序后的 JSON 文本

### 7.2 最后抓取的 HTML 保存

**位置**：`worker.py:557-559`

```python
empty_pages_are_a_change = datastore.data['settings']['application'].get('empty_pages_are_a_change', False)
if update_handler.fetcher.content or (not update_handler.fetcher.content and empty_pages_are_a_change):
    watch.save_last_fetched_html(contents=update_handler.fetcher.content, timestamp=int(fetch_start_time))
```

### 7.3 通知触发

**位置**：`worker.py:566-569`

```python
# Send notifications on second+ check
if watch.history_n >= 2:
    logger.info(f"Change detected in UUID {uuid} - {watch['url']}")
    if not watch.get('notification_muted'):
        await send_content_changed_notification(uuid, notification_q, datastore)
```

### 7.4 通知内容渲染

通知服务使用相同的 diff 渲染引擎，对所有内容类型一视同仁：
- `FormattableDiff`：可格式化的差异字符串（支持 `lines`、`added_only`、`removed_only` 等参数）
- `FormattableExtract`：仅提取变更部分（`diff_changed_from` / `diff_changed_to`）
- 支持 Jinja2 模板自定义通知格式
- 支持通过 Apprise 发送到 70+ 通知渠道

## 8. 处理路径总览图

```
URL 进入
  │
  ▼
┌─────────────────────────────────────────────────────┐
│ Watch.is_pdf 检测 (URL 后缀 / Content-Type)          │
└───────────────────┬─────────────────────────────────┘
                    │
           ┌────────┴────────┐
           │ 是 PDF?         │ 否
           ▼                 ▼
┌────────────────────┐  ┌─────────────────────┐
│ 强制 html_requests │  │ 按配置选择抓取器     │
│ is_binary = True   │  │ html_requests 或    │
│                    │  │ html_webdriver      │
└─────────┬──────────┘  └──────────┬──────────┘
          │                        │
          └──────────┬─────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│ 抓取内容 (self.raw_content 保存原始字节)             │
└───────────────────┬─────────────────────────────────┘
                    │
           ┌────────┴────────┐
           │ 是 PDF?         │ 否
           ▼                 ▼
┌────────────────────┐  ┌─────────────────────┐
│ preprocess_pdf()   │  │ 正常 HTML 处理       │
│ pdftohtml 转换     │  │                      │
│ 注入元数据         │  │                      │
└─────────┬──────────┘  └──────────┬──────────┘
          │                        │
          └──────────┬─────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│ 过滤器应用 (subtractive → include)                  │
│ HTML → 文本转换                                     │
│ 文本转换 (修剪/去重/排序/提取)                       │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│ 校验和计算 → 变更检测 → 阻塞规则评估                 │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
          ┌─────────┴─────────┐
          │ 检测到变更?       │ 否
          ▼                   ▼
┌────────────────────┐    ┌─────────────┐
│ 保存历史快照       │    │ 结束流程    │
│ 发送通知           │    │             │
└────────────────────┘    └─────────────┘
```

## 9. 与普通 HTML 抓取的关键分叉点对比

| 环节 | 普通 HTML | PDF/非 HTML | 代码位置 |
|------|-----------|-------------|----------|
| 抓取器选择 | 按配置选择 html_requests 或 html_webdriver | 强制使用 html_requests | `base.py:157-158` |
| is_binary 标志 | False | True | `base.py:239` |
| fetcher.content | 解码后的文本 | MD5 哈希（但 raw_content 保存原始字节） | `requests.py:195-197` |
| 预处理 | HTML 混淆 workaround | PDF → HTML 转换 + 元数据注入 | `processor.py:475-477` |
| stream_content_type | 根据检测结果设置 | 转换后强制 is_html=True | `processor.py:477` |

## 10. 关键设计考量

### 10.1 为什么强制 html_requests?
- Playwright 等浏览器会将 PDF 渲染为嵌入页面，无法获取原始字节
- HTTP 客户端可以直接获取 PDF 原始内容，供 pdftohtml 转换

### 10.2 为什么注入元数据?
- 确保即使 PDF 文本内容完全相同，文件本身的变化（如元数据修改）也能被检测
- 提供调试信息：校验和、文件大小

### 10.3 为什么使用外部 pdftohtml?
- 利用成熟工具的 PDF 解析能力，避免重复造轮子
- 可通过环境变量自定义转换工具，灵活适配不同环境

### 10.4 为什么双路径 PDF 检测?
- `watch.is_pdf`：早期预判，影响抓取器选择
- `stream_content_type.is_pdf`：内容确认，确保预处理执行
- 双重保险：即使 HTTP 头错误，也能通过内容检测正确处理
