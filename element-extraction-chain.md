# 页面元素抽取链路设计文档

## 1. 概述

本报告详细分析 Changedetection.io 页面元素抽取系统的完整链路，从用户配置选择器开始，经过内容抓取、元素定位、数据提取，最终到变化检测和结果呈现。

## 2. 架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         页面元素抽取完整链路                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────┐  │
│  │ 用户选择器配置│───▶│  抓取器执行  │───▶│  抽取器定位  │───▶│ 数据监控  │  │
│  │ (edit.html)  │    │(content_    │    │(html_tools. │    │(Watch.py) │  │
│  │              │    │ fetchers)   │    │  py)        │    │           │  │
│  └──────────────┘    └──────────────┘    └──────────────┘    └───────────┘  │
│         │                   │                   │                  │        │
│         ▼                   ▼                   ▼                  ▼        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────┐  │
│  │ 表单验证     │    │ 内容类型识别 │    │ XPath/CSS    │    │ 历史存储  │  │
│  │ (forms.py)   │    │ (magic.py)   │    │ 汇合点       │    │           │  │
│  └──────────────┘    └──────────────┘    └──────────────┘    └───────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3. 阶段一：用户选择器配置

### 3.1 配置入口

**文件位置：** `changedetectionio/forms.py:816-894`

用户在编辑页面的 "Filters & Triggers" 标签页配置选择器，主要涉及以下字段：

```python
class processor_text_json_diff_form(commonSettingsForm):
    # 包含过滤器 - 选择要监控的元素
    include_filters = StringListField(
        _l('CSS/JSONPath/JQ/XPath Filters'), 
        [ValidateCSSJSONXPATHInput()], 
        default=''
    )
    
    # 去除选择器 - 从监控中排除的元素
    subtractive_selectors = StringListField(
        _l('Remove elements'), 
        [ValidateCSSJSONXPATHInput(allow_json=False)]
    )
    
    # 提取包含特定文本的行
    extract_lines_containing = StringListField(
        _l('Extract lines containing'), 
        [validators.Optional()]
    )
    
    # 正则表达式提取
    extract_text = StringListField(
        _l('Extract text'), 
        [ValidateListRegex()]
    )
```

### 3.2 选择器语法类型

系统支持多种选择器语法，通过前缀自动识别：

| 语法类型 | 前缀/模式 | 示例 | 底层实现 |
|---------|----------|------|---------|
| CSS 选择器 | 无前缀（默认） | `.product-title`, `#price` | BeautifulSoup |
| XPath | `//` 或 `xpath:` | `//div[@class="price"]` | elementpath (XPath 3.0) |
| XPath 1.0 | `xpath1:` | `xpath1://*[local-name()="item"]` | lxml 原生 |
| JSONPath | `json:` | `json:$..price` | jsonpath_ng |
| jq | `jq:` / `jqraw:` | `jq:.items[].price` | jq (可选) |

### 3.3 表单验证

**验证器：** `changedetectionio/forms.py:618-716`

`ValidateCSSJSONXPATHInput` 验证器在表单提交时进行语法检查：

```python
class ValidateCSSJSONXPATHInput(object):
    def __call__(self, form, field):
        for line in data:
            # XPath 验证
            if line.strip()[0] == '/' or line.strip().startswith('xpath:'):
                from elementpath import select, SafeXPath3Parser
                tree = html.fromstring("<html></html>")
                try:
                    elementpath.select(tree, line.strip(), parser=SafeXPath3Parser)
                except elementpath.ElementPathError as e:
                    raise ValidationError(
                        f"'{expression}' is not a valid XPath expression. ({error})"
                    )
            
            # XPath1 验证
            if line.strip().startswith('xpath1:'):
                tree.xpath(line.strip())
            
            # JSONPath 验证
            if 'json:' in line:
                parse(input)
            
            # jq 验证
            if 'jq:' in line:
                jq.compile(input)
```

### 3.4 配置存储

配置通过 `Watch` 对象保存到 `url-watches.json`：

```python
# edit.py:225-226
datastore.data['watching'][uuid].update(form.data)
datastore.data['watching'][uuid].update(extra_update_obj)
datastore.data['watching'][uuid].commit()
```

## 4. 阶段二：抓取器获取页面内容

### 4.1 抓取器架构

**目录：** `changedetectionio/content_fetchers/`

| 抓取器 | 文件 | 适用场景 | 截图支持 |
|-------|------|---------|---------|
| html_requests | `requests.py` | 静态页面 | 否 |
| html_webdriver | `playwright.py` | JavaScript 渲染页面 | 是 |
| html_webdriver | `puppeteer.py` | JavaScript 渲染页面（备选） | 是 |
| html_webdriver | `webdriver_selenium.py` | Selenium 浏览器 | 是 |

### 4.2 抓取执行流程

**执行入口：** `changedetectionio/worker.py:165-175`

```python
# Worker 调度抓取
async def async_update_worker(worker_id, q, notification_q, app, datastore, executor):
    # ...
    # 调用浏览器抓取
    await update_handler.call_browser()
    
    # 在执行器中运行变更检测（避免阻塞事件循环）
    changed_detected, update_obj, contents = await loop.run_in_executor(
        executor,
        lambda: update_handler.run_changedetection(watch=watch)
    )
```

### 4.3 内容类型识别

**识别模块：** `changedetectionio/processors/magic.py`

```python
# processor.py:444-445
ctype_header = self.fetcher.get_all_headers().get('content-type', ...)
stream_content_type = guess_stream_type(
    http_content_header=ctype_header, 
    content=self.fetcher.content
)

# 返回对象包含属性
stream_content_type.is_html      # HTML 页面
stream_content_type.is_rss       # RSS/XML 订阅
stream_content_type.is_json      # JSON 数据
stream_content_type.is_pdf       # PDF 文档
stream_content_type.is_plaintext # 纯文本
```

### 4.4 内容预处理

**预处理阶段：** `changedetectionio/processors/text_json_diff/processor.py:461-489`

```python
# 1. RSS 预处理
if stream_content_type.is_rss:
    content = content_processor.preprocess_rss(content)
    # 选项：RSS 阅读器模式 → 转换为 HTML

# 2. PDF 预处理
if watch.is_pdf or stream_content_type.is_pdf:
    content = content_processor.preprocess_pdf(raw_content=self.fetcher.raw_content)
    # 使用 pdftohtml 转换为 HTML

# 3. JSON 预处理
if stream_content_type.is_json:
    if not filter_config.has_include_json_filters:
        content = content_processor.preprocess_json(raw_content=content)
        # 格式化并排序 JSON

# 4. HTML 混淆处理
if stream_content_type.is_html:
    content = html_tools.workarounds_for_obfuscations(content)
    # 处理特殊的混淆技术（如 HomeDepot 的注释注入）
```

### 4.5 抓取器输出

抓取器完成后，`update_handler` 包含：

| 属性 | 类型 | 说明 |
|-----|------|------|
| `content` | str | 抓取的页面内容（文本） |
| `raw_content` | bytes | 原始二进制内容 |
| `screenshot` | bytes \| None | 页面截图（PNG） |
| `xpath_data` | dict \| None | 元素定位数据（用于可视化选择器） |
| `last_status_code` | int | HTTP 状态码 |

## 5. 阶段三：抽取器解析定位 - XPath 与 CSS 汇合点

### 5.1 过滤器配置合并

**配置合并类：** `changedetectionio/processors/text_json_diff/processor.py:46-143`

```python
class FilterConfig:
    """合并 Watch、Tag、Global 三层配置"""
    
    @property
    def include_filters(self):
        # 1. Watch 配置
        filters = self._get_merged_rules('include_filters')
        # 2. 注入 LD+JSON 价格追踪
        if self.watch.get('track_ldjson_price_data'):
            filters += html_tools.LD_JSON_PRODUCT_OFFER_SELECTORS
        return filters
    
    @property
    def subtractive_selectors(self):
        # 合并顺序：Tag → Watch → Global
        return [*tag_selectors, *watch_selectors, *global_selectors]
```

**合并优先级：**
```
Tag 配置（覆盖） > Watch 配置 > Global 配置（默认）
```

### 5.2 核心汇合点 - apply_include_filters

**关键方法：** `changedetectionio/processors/text_json_diff/processor.py:334-380`

这是 XPath 与 CSS 语法的统一入口：

```python
class ContentProcessor:
    def apply_include_filters(self, content, stream_content_type):
        """应用包含过滤器 - XPath 与 CSS 在此汇合"""
        filtered_content = ""

        for filter_rule in self.filter_config.include_filters:
            # ═══════════════════════════════════════════════════════
            # 分支 1: XPath 过滤器
            # 识别条件: 以 '/' 或 'xpath:' 开头
            # ═══════════════════════════════════════════════════════
            if filter_rule[0] == '/' or filter_rule.startswith('xpath:'):
                filtered_content += html_tools.xpath_filter(
                    xpath_filter=filter_rule.replace('xpath:', ''),
                    html_content=content,
                    append_pretty_line_formatting=not self.watch.is_source_type_url,
                    is_xml=stream_content_type.is_rss or stream_content_type.is_xml
                )

            # ═══════════════════════════════════════════════════════
            # 分支 2: XPath1 过滤器（lxml 原生）
            # 识别条件: 以 'xpath1:' 开头
            # ═══════════════════════════════════════════════════════
            elif filter_rule.startswith('xpath1:'):
                filtered_content += html_tools.xpath1_filter(
                    xpath_filter=filter_rule.replace('xpath1:', ''),
                    html_content=content,
                    append_pretty_line_formatting=not self.watch.is_source_type_url,
                    is_xml=stream_content_type.is_rss or stream_content_type.is_xml
                )

            # ═══════════════════════════════════════════════════════
            # 分支 3: JSON 过滤器
            # 识别条件: 以 'json:', 'jq:', 'jqraw:' 开头
            # ═══════════════════════════════════════════════════════
            elif any(filter_rule.startswith(prefix) 
                     for prefix in JSON_FILTER_PREFIXES):
                filtered_content += html_tools.extract_json_as_string(
                    content=content,
                    json_filter=filter_rule
                )

            # ═══════════════════════════════════════════════════════
            # 分支 4: CSS 选择器（默认）
            # 识别条件: 不匹配以上任何模式
            # ═══════════════════════════════════════════════════════
            else:
                filtered_content += html_tools.include_filters(
                    include_filters=filter_rule,
                    html_content=content,
                    append_pretty_line_formatting=not self.watch.is_source_type_url
                )

        # ═══════════════════════════════════════════════════════
        # 结果检查：空内容抛出异常
        # ═══════════════════════════════════════════════════════
        if not filtered_content.strip():
            raise FilterNotFoundInResponse(
                msg=self.filter_config.include_filters,
                screenshot=self.fetcher.screenshot,
                xpath_data=self.fetcher.xpath_data
            )

        return filtered_content
```

### 5.3 CSS 选择器实现

**实现文件：** `changedetectionio/html_tools.py:136-152`

```python
def include_filters(include_filters, html_content, append_pretty_line_formatting=False):
    """CSS 选择器提取 - 使用 BeautifulSoup"""
    from bs4 import BeautifulSoup
    soup = BeautifulSoup(html_content, "html.parser")
    html_block = ""
    
    # 使用 BeautifulSoup 的 CSS 选择器
    r = soup.select(include_filters, separator="")

    for element in r:
        # 多结果时添加换行分隔符
        if append_pretty_line_formatting and len(html_block) \
           and not element.name in (['br', 'hr', 'div', 'p']):
            html_block += TEXT_FILTER_LIST_LINE_SUFFIX  # "<br>"
        html_block += str(element)

    return html_block
```

### 5.4 XPath 选择器实现

#### 5.4.1 XPath 3.0（推荐）

**实现文件：** `changedetectionio/html_tools.py:268-338`

```python
def xpath_filter(xpath_filter, html_content, 
                 append_pretty_line_formatting=False, 
                 is_xml=False):
    """XPath 3.0 提取 - 使用 elementpath 库"""
    from lxml import etree, html
    import elementpath

    # 解析器选择
    if is_xml:
        # XML/RSS 模式：保留 CDATA，禁用网络
        parser = etree.XMLParser(
            strip_cdata=False, 
            resolve_entities=False, 
            no_network=True
        )
        tree = etree.fromstring(html_content, parser=parser)
    else:
        # HTML 模式
        tree = html.fromstring(html_content, parser=etree.HTMLParser())

    # 命名空间处理
    namespaces = {'re': 'http://exslt.org/regular-expressions'}
    
    # 默认命名空间支持（RSS/Atom 常见）
    if hasattr(tree, 'nsmap') and tree.nsmap and None in tree.nsmap:
        namespaces[''] = tree.nsmap[None]

    # 使用安全解析器执行 XPath
    r = elementpath.select(
        tree, 
        xpath_filter.strip(), 
        namespaces=namespaces, 
        parser=SafeXPath3Parser
    )

    # 结果转换为字符串
    html_block = ""
    if type(r) != list:
        r = [r]

    for element in r:
        # ... 格式化处理
        if type(element) == str:
            html_block += element
        elif issubclass(type(element), etree._Element):
            html_block += etree.tostring(element, ...)
        else:
            html_block += elementpath_tostring(element)

    return html_block
```

#### 5.4.2 XPath 1.0（兼容）

**实现文件：** `changedetectionio/html_tools.py:341-393`

```python
def xpath1_filter(xpath_filter, html_content, 
                  append_pretty_line_formatting=False, 
                  is_xml=False):
    """XPath 1.0 提取 - 使用 lxml 原生 xpath()"""
    from lxml import etree, html

    # ... 解析逻辑同上 ...
    
    # NOTE: lxml 原生 xpath() 不支持默认命名空间的空前缀
    # 对于默认命名空间文档，需使用 local-name() 函数:
    #   //*[local-name()='title']/text()
    
    r = tree.xpath(xpath_filter.strip(), namespaces=namespaces)
    
    # ... 结果格式化 ...
```

#### 5.4.3 XPath 安全限制

**安全解析器：** `changedetectionio/html_tools.py:67-117`

为防止安全风险，系统构建了 `SafeXPath3Parser`，移除危险函数：

```python
_DEFAULT_UNSAFE_XPATH3_FUNCTIONS = [
    'unparsed-text',           # 文件读取
    'unparsed-text-lines',     # 文件读取
    'unparsed-text-available', # 文件读取
    'doc',                     # URI 资源获取
    'doc-available',           # URI 资源获取
    'json-doc',                # JSON 文档获取
    'collection',              # XML 节点集合加载
    'uri-collection',          # URI 集合枚举
    'transform',               # XSLT 转换
    'load-xquery-module',      # XQuery 模块加载
    'environment-variable',    # 环境变量泄露
    'available-environment-variables',  # 环境变量泄露
]

def _build_safe_xpath3_parser():
    """创建安全的 XPath3Parser 子类"""
    class SafeXPath3Parser(XPath3Parser):
        pass
    
    # 从符号表中移除危险函数
    for _fn in blocked:
        SafeXPath3Parser.symbol_table.pop(_fn, None)
    
    return SafeXPath3Parser

SafeXPath3Parser = _build_safe_xpath3_parser()  # 模块级单例
```

### 5.5 去除选择器（Subtractive Selectors）

**实现文件：** `changedetectionio/html_tools.py:198-224`

```python
def element_removal(selectors: List[str], html_content):
    """移除匹配选择器的元素"""
    modified_html = html_content
    css_selectors = []
    xpath_selectors = []

    # ═══════════════════════════════════════════════════════
    # 第一步：分类选择器
    # ═══════════════════════════════════════════════════════
    for selector in selectors:
        if selector.strip().startswith(('xpath:', 'xpath1:', '//')):
            # XPath 选择器
            xpath_selector = selector.removeprefix('xpath:').removeprefix('xpath1:')
            xpath_selectors.append(xpath_selector)
        else:
            # CSS 选择器
            css_selectors.append(selector.strip().strip(","))

    # ═══════════════════════════════════════════════════════
    # 第二步：应用 XPath 去除
    # ═══════════════════════════════════════════════════════
    if xpath_selectors:
        modified_html = subtractive_xpath_selector(xpath_selectors, modified_html)

    # ═══════════════════════════════════════════════════════
    # 第三步：应用 CSS 去除（优化：合并去重）
    # ═══════════════════════════════════════════════════════
    if css_selectors:
        # 去重后合并为一个 CSS 选择器（逗号分隔）
        # 防止元素索引偏移问题
        unique_selectors = list(set(css_selectors))
        combined_css_selector = " , ".join(unique_selectors)
        modified_html = subtractive_css_selector(
            combined_css_selector, 
            modified_html
        )

    return modified_html
```

### 5.6 多重选择器叠加语义

#### 5.6.1 Include Filters（包含）语义

**语义：并集（OR）**

```python
# processor.py:338-370
for filter_rule in self.filter_config.include_filters:
    # 每个规则独立执行，结果字符串拼接
    filtered_content += ...
```

**示例：**
```yaml
include_filters:
  - .product-title      # CSS: 选择所有产品标题
  - xpath://span[@class="price"]  # XPath: 选择所有价格
```

**结果：** 所有匹配 `.product-title` 的元素 + 所有匹配 XPath 的元素

#### 5.6.2 Subtractive Selectors（去除）语义

**语义：累积移除**

```python
# html_tools.py:213-221
# 1. XPath: 依次匹配并移除
# 2. CSS: 合并后一次性移除（优化）
```

**示例：**
```yaml
subtractive_selectors:
  - .advertisement      # CSS: 移除广告
  - xpath://script      # XPath: 移除脚本
```

**结果：** 先移除所有广告，再移除所有脚本

#### 5.6.3 执行顺序

```
┌─────────────────────────────────────────────────────────────────┐
│  过滤器执行顺序                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始内容                                                        │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 应用 include_filters（提取感兴趣的部分）               │   │
│  │    并集语义：选择器 1 ∪ 选择器 2 ∪ ...                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 应用 subtractive_selectors（从结果中移除）            │   │
│  │    累积语义：从包含结果中逐个移除                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼                                                          │
│  最终监控内容                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 6. 阶段四：抽取失败诊断

### 6.1 异常定义

**文件位置：** `changedetectionio/processors/text_json_diff/processor.py:34-38`

```python
class FilterNotFoundInResponse(ValueError):
    """过滤器未找到异常"""
    def __init__(self, msg, screenshot=None, xpath_data=None):
        self.screenshot = screenshot    # 页面截图
        self.xpath_data = xpath_data    # 元素定位数据
        ValueError.__init__(self, msg)
```

### 6.2 触发条件

**触发点：** `changedetectionio/processors/text_json_diff/processor.py:372-378`

```python
# 所有过滤器都返回空内容时触发
if not filtered_content.strip():
    raise FilterNotFoundInResponse(
        msg=self.filter_config.include_filters,  # 失败的过滤器列表
        screenshot=self.fetcher.screenshot,      # 截图供调试
        xpath_data=self.fetcher.xpath_data       # 元素数据
    )
```

### 6.3 异常处理流程

**处理入口：** `changedetectionio/worker.py:241-278`

```python
except FilterNotFoundInResponse as e:
    # ═══════════════════════════════════════════════════════
    # 1. 设置错误消息
    # ═══════════════════════════════════════════════════════
    err_text = (
        "Warning, no filters were found, no change detection ran - "
        "Did the page change layout? update your Visual Filter if necessary."
    )
    datastore.update_watch(uuid=uuid, update_obj={'last_error': err_text})

    # ═══════════════════════════════════════════════════════
    # 2. 保存诊断数据
    # ═══════════════════════════════════════════════════════
    if e.screenshot:
        watch.save_screenshot(screenshot=e.screenshot)
    if e.xpath_data:
        watch.save_xpath_data(data=e.xpath_data)

    # ═══════════════════════════════════════════════════════
    # 3. 连续失败计数与通知
    # ═══════════════════════════════════════════════════════
    if watch.get('filter_failure_notification_send', False):
        c = watch.get('consecutive_filter_failures', 0)
        c += 1
        
        # 获取通知阈值
        threshold = datastore.data['settings']['application'].get(
            'filter_failure_notification_threshold_attempts', 0
        )
        
        if c >= threshold:
            if not watch.get('notification_muted'):
                # 发送通知
                await send_filter_failure_notification(
                    uuid, notification_q, datastore
                )
            c = 0  # 重置计数
        
        datastore.update_watch(
            uuid=uuid, 
            update_obj={'consecutive_filter_failures': c}
        )

    # ═══════════════════════════════════════════════════════
    # 4. 标记本次检测失败
    # ═══════════════════════════════════════════════════════
    process_changedetection_results = False
```

### 6.4 诊断数据存储

**保存实现：** `changedetectionio/model/Watch.py:1168-1184`

```python
def save_xpath_data(self, data, as_error=False):
    """保存元素定位数据（压缩存储）"""
    import json, zlib
    
    if as_error:
        target_path = os.path.join(
            str(self.data_dir), 
            "elements-error.deflate"
        )
    else:
        target_path = os.path.join(
            str(self.data_dir), 
            "elements.deflate"
        )

    with open(target_path, 'wb') as f:
        if not isinstance(data, str):
            f.write(zlib.compress(json.dumps(data).encode()))
        else:
            f.write(zlib.compress(data.encode()))
```

**诊断文件清单：**

| 文件名 | 类型 | 用途 |
|-------|------|------|
| `last-screenshot.png` | 图片 | 正常抓取的截图 |
| `last-error-screenshot.png` | 图片 | 错误时的截图 |
| `elements.deflate` | 压缩数据 | 正常的元素定位数据 |
| `elements-error.deflate` | 压缩数据 | 错误时的元素定位数据 |
| `last-error.txt` | 文本 | 错误页面文本 |

### 6.5 用户可见诊断信息

#### 6.5.1 表单配置选项

**文件：** `changedetectionio/forms.py:873`

```python
filter_failure_notification_send = BooleanField(
    _l('Send a notification when the filter can no longer be found on the page'),
    default=False
)
```

**说明：** 当过滤器持续失败时，用户可选择接收通知

#### 6.5.2 页面错误提示

**模板：** `changedetectionio/blueprint/ui/templates/edit.html`

```
1. Watch 列表页
   - 显示红色错误状态
   - last_error 作为 tooltip

2. 编辑页面
   - Flash 消息："Warning, no filters were found..."
   - Visual Filter Selector 标签页显示最新截图
   - 可使用可视化选择器重新配置

3. 历史/差异页面
   - 显示 last_error 字段内容
   - 提供错误截图链接
```

#### 6.5.3 通知配置

**阈值设置：** `changedetectionio/forms.py:1078-1081`

```python
filter_failure_notification_threshold_attempts = IntegerField(
    _l('Number of times the filter can be missing before sending a notification'),
    ...
)
```

## 7. 阶段五：数据回填监控

### 7.1 文本提取

**HTML 转文本：** `changedetectionio/html_tools.py:651-703`

```python
def html_to_text(html_content: str, 
                 render_anchor_tag_content=False, 
                 is_rss=False, 
                 timeout=10) -> str:
    """使用 inscriptis 将 HTML 转换为纯文本"""
    from inscriptis import get_text
    
    # 预处理：移除脚本、样式等不可渲染标签
    if not is_rss:
        from bs4 import BeautifulSoup
        soup = BeautifulSoup(html_content, 'html.parser')
        for tag in soup.find_all([
            'head', 'script', 'style', 'noscript', 
            'svg', 'math', 'canvas', 'iframe', 'template'
        ]):
            tag.decompose()
        html_content = str(soup)
    
    # 转换为文本
    text_content = get_text(html_content, config=parser_config)
    return text_content
```

### 7.2 文本转换管道

**转换模块：** `changedetectionio/processors/text_json_diff/processor.py:145-205`

```python
class ContentTransformer:
    """文本转换管道"""
    
    @staticmethod
    def trim_whitespace(text):
        """去除每行首尾空白"""
        return '\n'.join(line.strip() for line in text.splitlines())
    
    @staticmethod
    def remove_duplicate_lines(text):
        """去重（保持顺序）"""
        return '\n'.join(dict.fromkeys(text.splitlines()))
    
    @staticmethod
    def sort_alphabetically(text):
        """按字母排序"""
        return '\n'.join(sorted(text.splitlines(), key=lambda x: x.lower()))
    
    @staticmethod
    def extract_lines_containing(text, substrings):
        """提取包含特定子串的行"""
        needles = [s.lower() for s in substrings if s.strip()]
        return '\n'.join(
            line for line in text.splitlines()
            if any(needle in line.lower() for needle in needles)
        )
    
    @staticmethod
    def extract_by_regex(text, regex_patterns):
        """正则表达式提取"""
        # 支持 Perl 风格正则 /pattern/flags
        regex_matched_output = []
        for s_re in regex_patterns:
            if re.search(PERL_STYLE_REGEX, s_re, re.IGNORECASE):
                regex = html_tools.perl_style_slash_enclosed_regex_to_options(s_re)
                result = re.findall(regex, text)
                # ... 处理结果
        return ''.join(regex_matched_output) if regex_matched_output else ''
```

### 7.3 忽略文本处理

**处理函数：** `changedetectionio/html_tools.py:571-636`

```python
def strip_ignore_text(content, wordlist, mode="content"):
    """
    处理忽略文本
    mode="content": 返回过滤后的内容
    mode="line numbers": 返回被忽略的行号列表
    """
    ignore_text = []      # 普通文本
    ignore_regex = []     # 正则表达式
    ignore_regex_multiline = []  # 多行正则
    
    # 分类处理
    for k in wordlist:
        if re.search(PERL_STYLE_REGEX, k, re.IGNORECASE):
            # 正则表达式
            res = re.compile(perl_style_slash_enclosed_regex_to_options(k))
            if res.flags & re.DOTALL or res.flags & re.MULTILINE:
                ignore_regex_multiline.append(res)
            else:
                ignore_regex.append(res)
        else:
            # 普通文本（大小写不敏感）
            ignore_text.append(k.strip())
    
    # ... 匹配并移除 ...
```

### 7.4 变更检测

**检测流程：** `changedetectionio/processors/text_json_diff/processor.py:569-631`

```python
# ═══════════════════════════════════════════════════════
# 1. 计算校验和
# ═══════════════════════════════════════════════════════
text_for_checksuming = stripped_text

# 应用忽略文本（仅影响校验和，不影响显示）
if filter_config.ignore_text:
    text_for_checksuming = html_tools.strip_ignore_text(
        stripped_text, 
        filter_config.ignore_text
    )

# 可选：从显示中也移除忽略的行
if strip_ignored_lines:
    stripped_text = text_for_checksuming

# 计算 MD5
ignore_whitespace = self.datastore.data['settings']['application'].get(
    'ignore_whitespace', False
)
fetched_md5 = ChecksumCalculator.calculate(
    text_for_checksuming, 
    ignore_whitespace=ignore_whitespace
)

# ═══════════════════════════════════════════════════════
# 2. 阻塞规则检查
# ═══════════════════════════════════════════════════════
blocked = False

# trigger_text: 内容必须包含触发文本才允许变更
if rule_engine.evaluate_trigger_text(
    text_for_checksuming, 
    filter_config.trigger_text
):
    blocked = True

# text_should_not_be_present: 包含禁止文本则阻塞
if rule_engine.evaluate_text_should_not_be_present(
    stripped_text, 
    filter_config.text_should_not_be_present
):
    blocked = True

# 自定义条件规则
if rule_engine.evaluate_conditions(watch, self.datastore, stripped_text):
    blocked = True

# ═══════════════════════════════════════════════════════
# 3. 变更判定
# ═══════════════════════════════════════════════════════
if blocked:
    changed_detected = False
else:
    # 比较校验和
    if watch.get('previous_md5') != fetched_md5:
        changed_detected = True
    
    # 更新记录的校验和
    update_obj["previous_md5"] = fetched_md5
    
    # 首次运行初始化
    if not watch.get('previous_md5'):
        watch['previous_md5'] = fetched_md5
```

### 7.5 配置变更追踪

**配置哈希：** `changedetectionio/processors/text_json_diff/processor.py:108-130`

```python
def get_filter_config_hash(self):
    """计算过滤器配置的稳定哈希
    
    用于检测配置变更，当：
    - 原始内容不变
    - 配置也不变
    时，可以跳过处理，直接返回"无变化"
    """
    app = self.datastore.data['settings']['application']
    config = {
        'extract_lines_containing':   sorted(self.extract_lines_containing),
        'extract_text':              sorted(self.extract_text),
        'ignore_text':               sorted(self.ignore_text),
        'include_filters':           sorted(self.include_filters),
        'subtractive_selectors':     sorted(self.subtractive_selectors),
        'text_should_not_be_present': sorted(self.text_should_not_be_present),
        'trigger_text':              sorted(self.trigger_text),
        # 全局处理标志
        'ignore_whitespace':         app.get('ignore_whitespace', False),
        'strip_ignored_lines':       app.get('strip_ignored_lines', False),
    }
    return hashlib.md5(
        json.dumps(config, sort_keys=True).encode()
    ).hexdigest()
```

**跳过逻辑：** `changedetectionio/processors/text_json_diff/processor.py:430-436`

```python
# 满足以下所有条件时跳过处理
if (not force_reprocess and
    not watch.was_edited and                            # 未编辑
    self.last_raw_content_checksum and                  # 有历史校验和
    self.last_raw_content_checksum == current_raw_document_checksum and  # 原始内容相同
    watch.get('last_filter_config_hash') and            # 有配置哈希
    watch.get('last_filter_config_hash') == current_filter_config_hash):  # 配置相同
    raise checksumFromPreviousCheckWasTheSame()
```

### 7.6 历史记录管理

**存储实现：** `changedetectionio/model/Watch.py:653-729`

```python
def save_history_blob(self, contents, timestamp, snapshot_id):
    """保存历史快照"""
    
    # ═══════════════════════════════════════════════════════
    # 1. 二进制数据（图片、PDF 等）
    # ═══════════════════════════════════════════════════════
    if isinstance(contents, bytes):
        import puremagic
        detections = puremagic.magic_string(contents[:2048])
        ext = detections[0].extension if detections else 'bin'
        snapshot_fname = f"{snapshot_id}.{ext}"
        # 直接保存，不压缩
    
    # ═══════════════════════════════════════════════════════
    # 2. 文本数据（支持 Brotli 压缩）
    # ═══════════════════════════════════════════════════════
    else:
        if not skip_brotli and len(contents) > BROTLI_COMPRESS_SIZE_THRESHOLD:
            # 压缩保存
            snapshot_fname = f"{snapshot_id}.txt.br"
            _brotli_save(contents, dest, mode=brotli.MODE_TEXT)
        else:
            # 普通保存
            snapshot_fname = f"{snapshot_id}.txt"
            self._write_atomic(dest, contents.encode('utf-8'))
    
    # ═══════════════════════════════════════════════════════
    # 3. 追加到历史索引
    # ═══════════════════════════════════════════════════════
    index_fname = os.path.join(self.data_dir, self.history_index_filename)
    index_line = f"{timestamp},{snapshot_fname}\n"
    with open(index_fname, 'a', encoding='utf-8') as f:
        f.write(index_line)
        f.flush()
        os.fsync(f.fileno())
    
    # ═══════════════════════════════════════════════════════
    # 4. 历史修剪（可选）
    # ═══════════════════════════════════════════════════════
    maxlen = self.get('history_snapshot_max_length') or \
             self.get_global_setting('application', 'history_snapshot_max_length')
    if maxlen and self.__history_n and self.__history_n > maxlen:
        self.history_trim(newest_n_items=maxlen)
```

## 8. 完整流程时序图

```
┌────────────┐      ┌────────────┐      ┌────────────┐      ┌────────────┐
│   用户     │      │  UI 层     │      │  Worker    │      │  处理器    │
│            │      │            │      │            │      │            │
└─────┬──────┘      └─────┬──────┘      └─────┬──────┘      └─────┬──────┘
      │                   │                   │                   │
      │ 1. 配置选择器     │                   │                   │
      │──────────────────>│                   │                   │
      │                   │                   │                   │
      │                   │ 2. 表单验证       │                   │
      │                   │ (ValidateCSS...)  │                   │
      │                   │                   │                   │
      │                   │ 3. 保存配置       │                   │
      │                   │ (url-watches.json)│                   │
      │<──────────────────│                   │                   │
      │                   │                   │                   │
      │                   │                   │ 4. 调度检查       │
      │                   │                   │ (queue)           │
      │                   │                   │                   │
      │                   │                   │ 5. 抓取内容       │
      │                   │                   │ (content_fetchers)│
      │                   │                   │                   │
      │                   │                   │ 6. 传递给处理器   │
      │                   │                   │──────────────────>│
      │                   │                   │                   │
      │                   │                   │                   │ 7. 合并配置
      │                   │                   │                   │ (FilterConfig)
      │                   │                   │                   │
      │                   │                   │                   │ 8. 应用过滤器
      │                   │                   │                   │ (XPath/CSS 汇合)
      │                   │                   │                   │
      │                   │                   │                   │ 9. 检测变更
      │                   │                   │                   │ (MD5 校验和)
      │                   │                   │                   │
      │                   │                   │ 10. 保存历史      │
      │                   │                   │<──────────────────│
      │                   │                   │ (save_history_)   │
      │                   │                   │                   │
      │                   │ 11. 结果反馈      │                   │
      │<──────────────────────────────────────│                   │
      │ (Watch 列表更新)   │                   │                   │
      │                   │                   │                   │
```

## 9. 关键设计特点总结

### 9.1 多语法统一入口
- 通过前缀检测自动识别选择器类型
- 无需用户显式指定语法类型
- 同一配置中可混合使用多种语法

### 9.2 配置层级覆盖
```
Watch 配置 → Tag 覆盖 → Global 默认
```
- 支持灵活的组管理
- 简化批量配置

### 9.3 丰富的失败诊断
- **三重诊断信息**：截图 + XPath 元素数据 + 页面文本
- **可视化选择器**：Visual Filter Selector 标签页
- **连续失败通知**：阈值配置 + 通知发送

### 9.4 安全防护
- XPath 危险函数移除（文件读取、环境变量等）
- jq 表达式安全检查
- 路径访问限制（历史快照 confined to watch 目录）

### 9.5 性能优化
- CSS 选择器合并去重（避免 DOM 索引偏移）
- 配置哈希跳过（内容和配置都不变时无需重处理）
- Brotli 压缩历史快照
- 异步执行（抓取 async，检测 run_in_executor）

### 9.6 容错机制
- 连续失败计数 + 阈值通知
- 防止静默失败
- 配置变更追踪（was_edited 标志）

## 10. 文件索引

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| 选择器配置表单 | `changedetectionio/forms.py` | 表单定义、验证器 |
| 抓取器 | `changedetectionio/content_fetchers/*.py` | 多种抓取后端 |
| 处理器核心 | `changedetectionio/processors/text_json_diff/processor.py` | XPath/CSS 汇合、变更检测 |
| 抽取工具 | `changedetectionio/html_tools.py` | CSS/XPath/JSON 提取实现 |
| Watch 模型 | `changedetectionio/model/Watch.py` | 历史存储、诊断数据保存 |
| Worker | `changedetectionio/worker.py` | 调度、异常处理 |
| 编辑 UI | `changedetectionio/blueprint/ui/edit.py` | 配置保存、路由 |
| 编辑模板 | `changedetectionio/blueprint/ui/templates/edit.html` | 用户界面 |
