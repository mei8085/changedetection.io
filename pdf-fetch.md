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

## 7. 下游通知流程

当检测到变更后，通知流程与普通 HTML 完全相同：

### 7.1 历史记录保存

**位置**：`worker.py:526-528`

```python
watch.save_history_blob(contents=contents,
                        timestamp=int(fetch_start_time),
                        snapshot_id=update_obj.get('previous_md5', 'none'))
```

### 7.2 通知触发

**位置**：`worker.py:566-569`

```python
# Send notifications on second+ check
if watch.history_n >= 2:
    logger.info(f"Change detected in UUID {uuid} - {watch['url']}")
    if not watch.get('notification_muted'):
        await send_content_changed_notification(uuid, notification_q, datastore)
```

### 7.3 通知内容渲染

通知服务使用相同的 diff 渲染引擎：
- `FormattableDiff`：可格式化的差异字符串
- `FormattableExtract`：仅提取变更部分（`diff_changed_from` / `diff_changed_to`）
- 支持 Jinja2 模板自定义通知格式

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
