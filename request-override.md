# 自定义请求头与请求体的三层传递链路分析

## 概览

changedetection.io 中，用户配置的自定义 HTTP 请求头（`headers`）、请求体（`body`）和请求方法（`method`）需经过 **配置实体层 → 调度执行层 → 抓取后端层** 才能最终生效。这条链路涉及 Watch 模型、处理器基类、多个 fetcher 实现，以及多种来源的 header 合并，代码路径并不直白。

---

## 第一层：配置实体层 —— 用户配置的持久化

### 1.1 数据模型定义

Watch 对象继承自 `dict`，三个关键字段在 [watch_base.__init__()](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/model/__init__.py#L174-L306) 中初始化：

```python
# changedetectionio/model/__init__.py L205, L214, L306
'headers': {},       # Extra headers to send (dict, key:value)
'method': 'GET',     # HTTP method
'body': None,        # Request body (string, 可含 Jinja2 模板)
```

- `headers`：`dict` 类型，存储 `key: value` 形式的自定义请求头
- `method`：字符串，支持 GET / POST / PUT / PATCH / DELETE / OPTIONS（定义在 [forms.py#L54-L61](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L54-L61)）
- `body`：字符串或 None，可包含 Jinja2 模板语法

### 1.2 表单层（UI 写入路径）

用户在编辑页面提交的表单由 [processor_text_json_diff_form](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L827-L954) 处理：

| 字段 | 表单控件 | 数据类型 | 特殊处理 |
|------|----------|----------|----------|
| `headers` | `StringDictKeyValue`（key:value 文本区域） | `dict` | 逐行解析 `key: value`，支持 Jinja2 模板验证 |
| `body` | `TextAreaField` | `str` | Jinja2 模板验证；GET 方法时不允许设置 body |
| `method` | `SelectField`（valid_method 集合） | `str` | 默认 `GET` |

**表单验证逻辑** ([forms.py#L900-L954](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L900-L954))：
- GET + body → 验证失败
- headers 中每个 value 和 body 内容都会经过 Jinja2 模板语法验证
- URL 也经过 Jinja2 模板验证

### 1.3 表单数据写回 Watch

在 [edit.py#L225-L226](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/blueprint/ui/edit.py#L225-L226) 中：

```python
datastore.data['watching'][uuid].update(form.data)
datastore.data['watching'][uuid].update(extra_update_obj)
```

`form.data` 包含 `headers`、`body`、`method`，直接通过 dict 的 `update()` 写入 Watch 对象，触发 `__setitem__` 标记 watch 为 edited。

### 1.4 API 写入路径

API 层通过 OpenAPI schema（[api/Spec.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/api/Spec.py)）定义的 `UpdateWatch` schema 直接写入 `headers`、`body`、`method` 字段到 Watch dict。

---

## 第二层：调度执行层 —— 从 Watch 配置到处理器上下文

### 2.1 Worker 调度入口

[async_update_worker()](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/worker.py#L46) 从优先级队列取出 `PrioritizedItem`，其中 `item` 仅包含 `{'uuid': uuid}`。

Worker **不直接传递** headers/body/method，而是通过 `datastore.data['watching'][uuid]` 获取 Watch 对象，再将 `watch_uuid` 传递给处理器。

### 2.2 处理器初始化 —— Watch 快照

[difference_detection_processor.__init__()](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L29-L44)：

```python
self.watch = deepcopy(self.datastore.data['watching'].get(watch_uuid))
```

此处 `deepcopy` 创建了 Watch 的**稳定快照**，包含 `headers`、`body`、`method` 的完整副本。后续处理器使用 `self.watch` 而非直接访问 datastore，避免并发修改。

### 2.3 Worker 调用链

```
Worker → processor_module.perform_site_check(datastore, watch_uuid)
       → update_handler.call_browser()    # 核心：组装请求参数并调用 fetcher
       → update_handler.run_changedetection(watch=watch)
```

关键：`call_browser()` 是请求头/体从 Watch 配置流向 fetcher 的**唯一装配点**。

---

## 第三层：抓取后端层 —— 请求参数的组装与注入

### 3.1 call_browser() 中的请求参数组装

[processors/base.py#L117-L257](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L117-L257) 是整条链路的核心。详细步骤：

#### Step 1: 确定目标 URL

```python
url = self.watch.link   # 支持 Jinja2 渲染、source: 前缀等
```

#### Step 2: 确定抓取后端

```python
prefer_fetch_backend = self.watch.get('fetch_backend', 'system')
if not prefer_fetch_backend or prefer_fetch_backend == 'system':
    prefer_fetch_backend = self.datastore.data['settings']['application'].get('fetch_backend')
```

PDF 自动降级为 `html_requests`。

#### Step 3: 请求头的多层合并（关键！）

```python
request_headers = CaseInsensitiveDict()

# 来源 1: 全局默认 UA（按 fetcher 类型选择）
ua = self.datastore.data['settings']['requests'].get('default_ua')
if ua and ua.get(prefer_fetch_backend):
    request_headers.update({'User-Agent': ua.get(prefer_fetch_backend)})

# 来源 2: Watch 级自定义 headers（用户在 Edit 页面配置的）
request_headers.update(self.watch.get('headers', {}))

# 来源 3: 全局 settings.headers（来自 DataStore 设置页）
request_headers.update(self.datastore.get_all_base_headers())

# 来源 4: headers.txt 文件（全局 + Watch 级 + Tag 级）
request_headers.update(self.datastore.get_all_headers_in_textfile_for_watch(uuid=self.watch.get('uuid')))
```

**合并优先级**（后写入覆盖先写入，因 `CaseInsensitiveDict.update()` 是覆盖语义）：

| 优先级 | 来源 | 位置 | 说明 |
|--------|------|------|------|
| 低 | 全局默认 UA | `settings.requests.default_ua.{fetcher}` | 按 fetcher 类型选择 UA |
| 中 | Watch 自定义 headers | `watch['headers']` | 用户在 Edit 页面配置 |
| 中高 | 全局 settings headers | `settings.headers` | 全局设置页配置 |
| **高** | headers.txt 文件 | 全局/Watch/Tag 三级 | 运维人员手动放置的文件 |

> ⚠️ 注意：`get_all_headers_in_textfile_for_watch()` 会合并三级 headers.txt 文件（全局 > Watch 级 > Tag 级），优先级最高。

#### Step 4: Brotli 编码安全处理

```python
if 'Accept-Encoding' in request_headers and "br" in request_headers['Accept-Encoding']:
    request_headers['Accept-Encoding'] = request_headers['Accept-Encoding'].replace(', br', '')
```

Python `requests` 库不支持 Brotli 解码，强制移除 `br`。

#### Step 5: Jinja2 模板渲染所有 header value

```python
for header_name in request_headers:
    request_headers.update({header_name: jinja_render(template_str=request_headers.get(header_name))})
```

**每个 header 的 value 都经过 Jinja2 渲染**，支持 `{{ now }}` 等动态模板。

#### Step 6: 请求体的 Jinja2 渲染

```python
request_body = self.watch.get('body')
if request_body:
    request_body = jinja_render(template_str=self.watch.get('body'))
```

请求体同样支持 Jinja2 模板，仅在非空时渲染。

#### Step 7: 请求方法

```python
request_method = self.watch.get('method')
```

直接从 Watch 配置读取，无额外处理。

### 3.2 调用 Fetcher

组装完所有参数后，统一传入 `fetcher.run()`：

```python
await self.fetcher.run(
    current_include_filters=self.watch.get('include_filters'),
    empty_pages_are_a_change=empty_pages_are_a_change,
    fetch_favicon=self.watch.favicon_is_expired(),
    ignore_status_codes=ignore_status_codes,
    is_binary=is_binary,
    request_body=request_body,         # ← 渲染后的请求体
    request_headers=request_headers,   # ← 合并+渲染后的请求头
    request_method=request_method,     # ← 原始请求方法
    screenshot_format=self.screenshot_format,
    timeout=timeout,
    url=url,                           # ← 渲染后的 URL
    watch_uuid=self.watch_uuid,
)
```

---

## 第四层：各 Fetcher 后端的注入实现

### 4.1 html_requests (requests 库)

[content_fetchers/requests.py#L211-L245](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/requests.py#L211-L245)

`run()` 是 async wrapper，内部调用 `_run_sync()`：

```python
r = session.request(
    method=request_method,                                    # 直接使用
    data=request_body.encode('utf-8') if type(request_body) is str else request_body,  # 编码后传入
    url=url,
    headers=request_headers,                                  # 直接传入
    timeout=timeout,
    proxies=proxies,
    verify=False,
    allow_redirects=False
)
```

- `request_method` → `session.request(method=...)`
- `request_body` → `data=` 参数（str 先 encode 为 bytes）
- `request_headers` → `headers=` 参数

重定向时仅传递 `headers`，不传递 `method` 和 `data`（重定向统一用 GET）。

### 4.2 html_webdriver (Playwright)

[content_fetchers/playwright.py#L250-L293](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/playwright.py#L250-L293)

Playwright 的注入方式完全不同——通过浏览器上下文（BrowserContext）配置注入：

```python
context = await browser.new_context(
    accept_downloads=False,
    bypass_csp=True,
    extra_http_headers=request_headers,     # ← headers 注入到浏览器上下文
    ignore_https_errors=True,
    proxy=self.proxy,
    service_workers=...,
    user_agent=manage_user_agent(headers=request_headers),  # ← 从 headers 提取 UA
)
```

**关键差异**：
- `request_headers` 通过 `extra_http_headers` 注入，成为浏览器上下文的默认 HTTP 头
- `User-Agent` 从 headers 中提取后单独设置（通过 [manage_user_agent()](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/base.py#L8-L39)）
- `request_body` 和 `request_method` **在 Playwright fetcher 中未使用**——Playwright 仅导航到 URL，不支持自定义请求方法/请求体
- 页面通过 `action_goto_url(value=url)` 加载

### 4.3 Puppeteer (旧版)

[content_fetchers/puppeteer.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/puppeteer.py) 类似 Playwright，同样通过浏览器上下文注入 headers，不支持自定义 method/body。

---

## 完整数据流图

```
┌─────────────────────────────────────────────────────────┐
│                 第一层：配置实体层                         │
│                                                         │
│  Watch dict (持久化到 watch.json)                        │
│  ├─ headers: {}     ← Edit 表单 / API 写入               │
│  ├─ method: 'GET'   ← Edit 表单 / API 写入               │
│  └─ body: None      ← Edit 表单 / API 写入               │
│                                                         │
│  其他 header 来源（运行时读取，不持久化在 Watch 中）:        │
│  ├─ settings.headers            (全局设置页)              │
│  ├─ settings.requests.default_ua (全局 UA 配置)           │
│  ├─ datastore/headers.txt       (全局 headers 文件)       │
│  ├─ datastore/{uuid}/headers.txt(Watch 级 headers 文件)   │
│  └─ datastore/tag-name.txt      (Tag 级 headers 文件)    │
└────────────────────────┬────────────────────────────────┘
                         │ deepcopy (快照)
                         ▼
┌─────────────────────────────────────────────────────────┐
│               第二层：调度执行层                           │
│                                                         │
│  difference_detection_processor                         │
│  ├─ self.watch = deepcopy(watch)  ← 稳定快照            │
│  └─ call_browser() 是请求参数的装配点                     │
│                                                         │
│  Worker 仅传递 uuid，不直接传递 headers/body/method       │
└────────────────────────┬────────────────────────────────┘
                         │ call_browser() 组装
                         ▼
┌─────────────────────────────────────────────────────────┐
│          第三层：call_browser() 参数组装                   │
│                                                         │
│  request_headers = CaseInsensitiveDict()                │
│    ① 全局默认 UA (低优先级)                               │
│    ② Watch['headers'] (中优先级)                          │
│    ③ 全局 settings.headers (中高优先级)                   │
│    ④ headers.txt 文件三级合并 (最高优先级)                 │
│    ⑤ 移除 Accept-Encoding 中的 br                        │
│    ⑥ 所有 header value 做 Jinja2 渲染                    │
│                                                         │
│  request_body = jinja_render(watch['body'])  (如有)      │
│  request_method = watch['method']                       │
└────────────────────────┬────────────────────────────────┘
                         │ fetcher.run(...)
                         ▼
┌─────────────────────────────────────────────────────────┐
│             第四层：抓取后端层                             │
│                                                         │
│  html_requests:                                         │
│    session.request(method, data=body, headers=headers)  │
│    → 完整支持 method + body + headers                   │
│                                                         │
│  Playwright:                                            │
│    browser.new_context(extra_http_headers=headers)       │
│    manage_user_agent(headers=headers) → user_agent       │
│    → 仅支持 headers，method/body 被忽略                  │
│                                                         │
│  Puppeteer:                                             │
│    类似 Playwright，仅支持 headers                       │
└─────────────────────────────────────────────────────────┘
```

---

## 关键代码引用

| 环节 | 文件 | 行号 | 说明 |
|------|------|------|------|
| Watch 模型字段定义 | [model/__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/model/__init__.py#L205) | L205 | `headers: {}` |
| Watch 模型字段定义 | [model/__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/model/__init__.py#L214) | L214 | `method: 'GET'` |
| Watch 模型字段定义 | [model/__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/model/__init__.py#L179) | L179 | `body: None` |
| 表单 headers 字段 | [forms.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L861) | L861 | `StringDictKeyValue` |
| 表单 body 字段 | [forms.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L862) | L862 | `TextAreaField` |
| 表单 method 字段 | [forms.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L863) | L863 | `SelectField` |
| 表单验证（GET+body 拒绝） | [forms.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L908-L909) | L908-909 | GET 时 body 必须为空 |
| 表单验证（Jinja2 headers） | [forms.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/forms.py#L940-L953) | L940-953 | 验证每个 header 的 Jinja2 模板 |
| Edit 页面写回 Watch | [edit.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/blueprint/ui/edit.py#L225) | L225 | `datastore.data['watching'][uuid].update(form.data)` |
| 处理器 deepcopy Watch | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L38) | L38 | `self.watch = deepcopy(...)` |
| 全局默认 UA 合并 | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L202-L204) | L202-204 | 按 fetcher 类型选 UA |
| Watch headers 合并 | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L206) | L206 | `request_headers.update(self.watch.get('headers', {}))` |
| 全局 settings headers 合并 | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L207) | L207 | `request_headers.update(self.datastore.get_all_base_headers())` |
| headers.txt 文件合并 | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L208) | L208 | 三级 headers.txt 合并 |
| Brotli 安全处理 | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L213-L214) | L213-214 | 移除 Accept-Encoding 中的 br |
| Jinja2 渲染 headers | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L216-L217) | L216-217 | 所有 header value 做 Jinja2 渲染 |
| Jinja2 渲染 body | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L221-L223) | L221-223 | body 做 Jinja2 渲染 |
| 读取 method | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L225) | L225 | `self.watch.get('method')` |
| 调用 fetcher.run() | [processors/base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L244-L257) | L244-257 | 传入全部参数 |
| requests fetcher 实现 | [requests.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/requests.py#L98-L105) | L98-105 | `session.request(method, data, headers)` |
| Playwright fetcher 实现 | [playwright.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/playwright.py#L285-L293) | L285-293 | `extra_http_headers` + `user_agent` |
| UA 提取工具函数 | [base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/base.py#L8-L39) | L8-39 | `manage_user_agent()` |
| 全局 headers.txt | [store/__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/store/__init__.py#L900-L932) | L900-932 | 三级 headers.txt 合并逻辑 |

---

## 注意事项与潜在问题

1. **Playwright/Puppeteer 不支持自定义 method 和 body**：用户在 Edit 页面配置了 POST + body，但如果 fetch_backend 是 Playwright，这些配置会被静默忽略，请求仍以 GET 方式发出。UI 层没有对此做任何提示或限制。

2. **headers 合并优先级反直觉**：`headers.txt` 文件的优先级最高（最后写入覆盖），而非用户在 Edit 页面配置的 Watch 级 headers。运维人员可能通过 headers.txt 覆盖用户配置而不自知。

3. **CaseInsensitiveDict 的覆盖语义**：`update()` 是覆盖而非追加，因此 `settings.headers` 中的 key 会覆盖 Watch 级同名 key。但 `get_all_base_headers()` 在 Watch headers **之后**调用，所以全局 headers 会覆盖 Watch 级同名 header。

4. **Jinja2 渲染时机**：headers 和 body 的 Jinja2 渲染发生在 `call_browser()` 中，而非表单提交时。这意味着模板中的动态值（如 `{{ now }}`）每次检查时都会重新求值，但表单验证仅检查模板语法是否合法。

5. **重定向丢失 method/body**：requests fetcher 在跟随重定向时统一使用 GET，不传递原始的 method 和 body，这可能导致 POST 请求在 302 重定向后变为 GET。

---

## 附录：四大 Fetcher 后端的注入机制深度对比

### A.1 后端选择机制

四个 fetcher 共享同一个注册名 `html_webdriver`，在运行时根据环境变量选择实际实现：

[content_fetchers/__init__.py#L93-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/__init__.py#L93-L105)：

```python
use_playwright_as_chrome_fetcher = os.getenv('PLAYWRIGHT_DRIVER_URL', False)
if use_playwright_as_chrome_fetcher:
    if not strtobool(os.getenv('FAST_PUPPETEER_CHROME_FETCHER', 'False')):
        from .playwright import fetcher as html_webdriver      # Playwright
    else:
        from .puppeteer import fetcher as html_webdriver        # Puppeteer (pyppeteer-ng)
else:
    from .webdriver_selenium import fetcher as html_webdriver   # Selenium
```

`html_requests` 始终独立可用。因此用户面对的选择只有两个：plaintext（requests）和 Chrome（Playwright/Puppeteer/Selenium 三选一）。

### A.2 字段注入支持度总览

| 字段 | html_requests | Playwright | Puppeteer | Selenium |
|------|:---:|:---:|:---:|:---:|
| `request_headers` | ✅ 完整支持 | ✅ `extra_http_headers` | ✅ `setExtraHTTPHeaders` | ❌ **完全忽略** |
| `request_method` | ✅ 直接传入 | ❌ 未使用 | ❌ 未使用 | ❌ 未使用 |
| `request_body` | ✅ 编码后传入 | ❌ 未使用 | ❌ 未使用 | ❌ 未使用 |
| `User-Agent` 特殊处理 | 无（headers 原样传入） | ✅ `manage_user_agent()` 提取后单独设为 `user_agent` | ✅ `setUserAgent()` + 从 headers 中 pop 出 UA | ❌ 无处理 |

### A.3 html_requests —— HTTP 协议级注入

[content_fetchers/requests.py#L96-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/requests.py#L96-L105)

```python
r = session.request(
    method=request_method,
    data=request_body.encode('utf-8') if type(request_body) is str else request_body,
    url=url,
    headers=request_headers,
    ...
)
```

**机制**：直接通过 Python `requests` 库的 API 参数注入。`requests` 是 HTTP 客户端库，每个请求都是独立的 HTTP 事务，可以精确控制 method / headers / body 的每一个细节。

**设计取舍**：这是唯一一个"全字段"后端，因为 `requests` 库本身提供了一等公民的 method/data/headers 参数，没有任何协议层限制。程序只需把 `call_browser()` 组装好的参数原样传递即可。

**重定向行为**：跟随 302 时只传 headers，method 退化为 GET，body 丢弃。这是 HTTP 语义决定的——大多数浏览器和 HTTP 客户端对 302 的处理也是如此。

### A.4 Playwright —— 浏览器上下文级注入

[content_fetchers/playwright.py#L285-L293](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/playwright.py#L285-L293)

```python
context = await browser.new_context(
    extra_http_headers=request_headers,                        # ← 全部 headers
    user_agent=manage_user_agent(headers=request_headers),     # ← UA 单独提取
    ...
)
page = await context.new_page()
response = await browsersteps_interface.action_goto_url(value=url)  # 仅导航
```

**机制**：Playwright 的架构是"浏览器上下文 → 页面 → 导航"。自定义 headers 在 `new_context()` 时作为 `extra_http_headers` 注入，此后该上下文下的所有页面请求（包括子资源）都会自动携带这些 headers。UA 则通过 `manage_user_agent()` 从 headers 中提取后作为独立参数 `user_agent` 设置。

**为什么不支持 method / body**：

Playwright 的导航原语是 `page.goto(url)`——这等价于用户在浏览器地址栏输入 URL 按回车，本质是一个 **GET 导航**。Playwright 的 `page.goto()` API 没有提供 `method` 或 `postData` 参数。如果需要发送 POST 请求，理论上需要通过 `page.evaluate()` 在浏览器内执行 `fetch()` API，但 changedetection.io 没有实现这种方式。

**设计取舍**：

1. **为什么用 `extra_http_headers` 而非 CDP 拦截**：Playwright 官方推荐的方式就是 `extra_http_headers`，它通过 CDP（Chrome DevTools Protocol）的 `Network.setExtraHTTPHeaders` 命令实现，在浏览器进程的 Network 层注入，对页面 JS 不可见，不会触发 CORS 预检。
2. **为什么 UA 要单独处理**：浏览器的 User-Agent 不属于 HTTP headers 的范畴——它由 Chromium 的 `user_agent` 客户端配置项控制。如果仅在 `extra_http_headers` 里设置 UA，浏览器的 JS 上下文（`navigator.userAgent`）不会改变。因此需要两步：从 headers 字典中提取 UA → 分别设置到 `user_agent` 参数。

### A.5 Puppeteer (pyppeteer-ng) —— 页面级注入

[content_fetchers/puppeteer.py#L339-L352](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/puppeteer.py#L339-L352)

```python
# UA: 先从 headers 中 pop 出来，再通过 setUserAgent 设置
user_agent = None
if request_headers and request_headers.get('User-Agent'):
    user_agent = request_headers.pop('User-Agent').strip()
    await self.page.setUserAgent(user_agent)

if not user_agent:
    await self.page.setUserAgent(manage_user_agent(headers=request_headers, current_ua=await self.page.evaluate('navigator.userAgent')))

# 其他 headers: 通过 setExtraHTTPHeaders 注入
await self.page.setBypassCSP(True)
if request_headers:
    await self.page.setExtraHTTPHeaders(request_headers)
```

**机制**：与 Playwright 类似但粒度不同。Puppeteer 在 **page 级别**（而非 context 级别）注入 headers。

**与 Playwright 的关键差异**：

| 维度 | Playwright | Puppeteer |
|------|------------|-----------|
| headers 注入层级 | `browser.new_context()` — Context 级 | `page.setExtraHTTPHeaders()` — Page 级 |
| UA 设置方式 | `new_context(user_agent=...)` 构造参数 | `page.setUserAgent()` 方法调用 |
| UA 是否从 headers 中移除 | 否（`CaseInsensitiveDict` 中 UA 仍在，但 Playwright 内部处理不重复发送） | **是**——`request_headers.pop('User-Agent')` 显式移除后再 `setExtraHTTPHeaders` |
| 底层 CDP 命令 | 相同：`Network.setExtraHTTPHeaders` | 相同：`Network.setExtraHTTPHeaders` |

**为什么 Puppeteer 要 pop UA 而 Playwright 不用**：Playwright 的 `new_context(user_agent=..., extra_http_headers=...)` 在内部会协调两者——当 `user_agent` 参数设置后，`extra_http_headers` 中的 `User-Agent` 会被自动忽略或去重。而 pyppeteer-ng 的 `setExtraHTTPHeaders` 会原样把传入的字典设为额外 headers，如果 UA 同时存在于 `setUserAgent()` 和 `setExtraHTTPHeaders()` 中，会导致重复发送或 Chrome 报 `ERR_INVALID_ARGUMENT` 错误（Chrome DevTools Protocol 不允许通过 `setExtraHTTPHeaders` 设置某些受保护的 headers，包括 User-Agent）。

**为什么不支持 method / body**：与 Playwright 完全相同的理由——`page.goto(url)` 是 GET 导航，没有 method/postData 参数。

### A.6 Selenium WebDriver —— 完全无注入

[content_fetchers/webdriver_selenium.py#L63-L151](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L63-L151)

```python
async def run(self, ..., request_body=None, request_headers=None, request_method=None, ...):

    def _run_sync():
        # request_body, request_method unused for now, until some magic in the future happens.
        options = ChromeOptions()
        # ... proxy 配置 ...
        driver = RemoteWebDriver(command_executor=remote_connection, options=options)
        driver.get(url)           # ← 仅导航，无 headers / method / body
        # ...
        self.content = driver.page_source
        self.headers = {}          # ← 响应 headers 也不可用！
```

**机制**：Selenium 是四个后端中注入能力最弱的——`request_headers`、`request_method`、`request_body` 三个参数虽然出现在函数签名中，但在 `_run_sync()` 函数体内**完全未使用**。源码注释明确承认这一点（L83）：

> `# request_body, request_method unused for now, until some magic in the future happens.`

**为什么 headers 也无法注入**：Selenium WebDriver 协议的 `driver.get(url)` 命令（对应 W3C WebDriver 规范的 `Navigate To`）不支持自定义 HTTP headers。这与 Playwright/Puppeteer 通过 CDP 的 `Network.setExtraHTTPHeaders` 绕过限制的方式不同——Selenium WebDriver 是 W3C 标准协议，没有等价的 CDP 通道。

**理论上可行的替代方案及为什么没用**：

| 方案 | 原理 | 为什么未采用 |
|------|------|-------------|
| Selenium Wire | 拦截并修改浏览器发出的 HTTP 请求 | 源码注释（L98-99）明确指出 `selenium-wire` 与 `pyppeteer-ng` 存在依赖冲突（websocket 库版本不兼容） |
| CDP 直接调用 | 通过 `driver.execute_cdp_cmd('Network.setExtraHTTPHeaders', ...)` | W3C 兼容模式的 RemoteWebDriver 不暴露 CDP 命令接口；只有 `ChromeDriver` 本地实例才支持 |
| Chrome 扩展注入 | 编写 Chrome 扩展在 `onBeforeSendHeaders` 中修改请求头 | 复杂度过高，且需要维护扩展代码 |
| `page.addInitScript()` / JS `fetch()` | 在页面中注入 JS 执行 fetch 请求 | Selenium 的 JS 注入时机在页面加载之后，无法修改首次导航的请求头 |

**响应 headers 也不可用**：Selenium 无法获取 HTTP 响应头，代码中 `self.headers = {}` 写死了空字典。对比 Playwright 通过 `response.all_headers()` 获取、Puppeteer 通过 `response.headers` 获取。

**status_code 也是硬编码**：`self.status_code = 200`（L143），Selenium 无法获取真实的 HTTP 状态码。注释中也承认这是一个 TODO。

### A.7 为什么四个后端差距这么大——根本原因分析

差距的根源不在 changedetection.io 的代码设计，而在**底层驱动协议的能力边界**：

```
能力从弱到强：

Selenium (W3C WebDriver)          ← 标准协议，只有 "导航到 URL" 的能力
    ↓
Puppeteer (CDP over pyppeteer-ng) ← 非标准 CDP，可注入 headers / UA，但 method/body 仍不行
    ↓
Playwright (CDP over Playwright)  ← 同样 CDP，但封装更完善，context 级 header 注入
    ↓
requests (Python HTTP 客户端)      ← 完全控制 HTTP 事务的每个字节
```

**W3C WebDriver 协议的根本限制**：Selenium 实现的是 W3C WebDriver 规范，该规范只定义了 `Navigate To`（`driver.get(url)`）这一导航命令，没有任何机制可以：
- 在导航请求中附加自定义 headers
- 改变导航请求的 HTTP 方法
- 在导航请求中附加请求体

这是规范层面的缺失，不是实现缺陷。

**CDP 协议的额外能力**：Playwright 和 Puppeteer 都通过 Chrome DevTools Protocol (CDP) 与浏览器通信。CDP 的 `Network.setExtraHTTPHeaders` 命令可以在 Network 层拦截并修改所有请求的 headers，从而绕过 W3C WebDriver 的限制。但 CDP 同样没有提供"以 POST 方式导航到 URL 并携带请求体"的命令——CDP 的 `Page.navigate` 只接受 URL，CDP 的 `Fetch.enable` 虽然可以拦截和修改请求，但需要配合复杂的请求拦截模式使用，changedetection.io 未实现。

**Python requests 库的完全控制**：作为 HTTP 客户端而非浏览器驱动，`requests` 库直接发送 HTTP 请求，不涉及任何浏览器进程，因此对 method / headers / body 拥有完全的控制权。代价是无法执行 JavaScript、无法渲染 SPA 页面。

**changedetection.io 的设计取舍总结**：

1. **所有 fetcher 共享同一个 `run()` 签名**（定义在 [base.py#L122-L134](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/base.py#L122-L134)），`request_headers`、`request_body`、`request_method` 始终作为参数传入——即使某些 fetcher 不使用它们。这是"宽接口"设计，保证了 `call_browser()` 无需感知后端差异。

2. **静默丢弃而非报错**：不支持 method / body 的后端选择了"静默忽略"策略。从工程角度这是合理的——用户在 Edit 页面配置 method=POST + body 时，系统无法在保存时验证"你的 fetcher 后端是否支持 POST"，因为 fetcher 后端可以随时切换。但在运行时抛异常又会导致检查失败。权衡之下，静默丢弃是最安全的策略，尽管用户体验上有隐患。

3. **Selenium 是遗留后端**：从 [__init__.py#L93-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/__init__.py#L93-L105) 可以看出，当 `PLAYWRIGHT_DRIVER_URL` 环境变量存在时优先使用 Playwright，Selenium 仅在没有任何 Playwright/Puppeteer 配置时作为 fallback。现代部署几乎都会设置 `PLAYWRIGHT_DRIVER_URL`，Selenium 实际上是遗留方案。

### A.8 后端能力差异对用户的影响

| 用户操作 | html_requests | Playwright/Puppeteer | Selenium |
|----------|:---:|:---:|:---:|
| 设置自定义请求头（如 `Authorization`） | ✅ 正常生效 | ✅ 正常生效 | ❌ 静默丢弃 |
| 设置 User-Agent | ✅ 作为普通 header 传入 | ✅ 单独提取后设为浏览器 UA | ❌ 静默丢弃 |
| 用 POST + body 监测 API | ✅ 正常生效 | ❌ method 退化为 GET，body 丢弃 | ❌ method 退化为 GET，body 丢弃 |
| 用 PUT/PATCH/DELETE 调用 API | ✅ 正常生效 | ❌ 退化为 GET | ❌ 退化为 GET |
| 通过 headers.txt 添加运维级 header | ✅ 正常生效 | ✅ 正常生效 | ❌ 静默丢弃 |

**核心结论**：如果用户需要自定义 method / body，**必须使用 `html_requests` 后端**。浏览器类后端（Playwright/Puppeteer/Selenium）在架构上无法支持非 GET 的首次导航请求。如果用户需要自定义 headers 且使用浏览器后端，只能选择 Playwright 或 Puppeteer（Selenium 连 headers 都不支持）。

---

## 附录 B：requests fetcher 重定向链逐跳保护的设计动机

### B.1 代码位置

[content_fetchers/requests.py#L93-L127](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/content_fetchers/requests.py#L93-L127)

```python
# 初始请求
if is_url_private_or_parser_confused(url):
    raise Exception(f"Fetch blocked: ...")

r = session.request(..., allow_redirects=False)   # ← 禁用 requests 库自带的重定向

# 手动跟随重定向，逐跳校验
current_url = url
for _ in range(10):
    if not r.is_redirect:
        break
    location = r.headers.get('Location', '')
    redirect_url = urljoin(current_url, location)
    if not allow_iana_restricted:
        if is_url_private_or_parser_confused(redirect_url):
            raise Exception(f"Redirect blocked: ...")
    current_url = redirect_url
    r = session.request('GET', redirect_url,   # ← 重定向统一用 GET，丢 body
                        headers=request_headers,
                        ...,
                        allow_redirects=False)
else:
    raise Exception("Too many redirects")
```

代码注释在 L107-108 已经点出了动机：

> "Manually follow redirects so each hop's resolved IP can be validated, preventing SSRF via an open redirect on a public host."

但注释背后的攻击场景和为什么要双重校验（初始 URL + 每一跳 redirect）需要展开分析。

### B.2 第一层防护：初始 URL 的 SSRF 检查

[is_url_private_or_parser_confused()](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/validate_url.py#L112-L126)

```python
def is_url_private_or_parser_confused(url):
    if '\\' in url:
        return True          # 反斜杠 —— 解析器差异攻击向量
    for hostname in extract_url_hostnames(url):
        if is_private_hostname(hostname):
            return True      # 任一解析器认为是私有 IP → 阻断
    return False
```

这一层在初始请求发出前校验用户输入的 URL，防止：

1. **直接 SSRF**：用户输入 `http://127.0.0.1:8080/admin`，试图让服务器请求自己的内网服务。
2. **反斜杠解析器差异攻击（GHSA-rph4-96w6-q594）**：形如 `http://INTERNAL:8888\@PUBLIC/` 的 URL，Python 标准库 `urlparse` 会解析出 hostname 为 `PUBLIC`（认为 `@PUBLIC` 是路径的一部分），但 `urllib3/requests` 实际连接时会连接到 `INTERNAL`。如果只信任 urlparse 的结果，攻击者可以绕过 SSRF 检查访问内网。
3. **DNS Rebinding（TOCTOU 攻击）**：用户提交的域名在表单验证时解析为公共 IP，但在实际请求时 DNS 返回私有 IP。注意 [is_private_hostname()](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/validate_url.py#L61-L80) 的注释明确说它**不使用 LRU 缓存**，每次调用都重新 DNS 解析——这是专门为 fetch-time 设计的，在请求发出前最后一刻做 DNS 解析，降低 DNS 重绑定的时间窗口。

### B.3 第二层防护：逐跳重定向校验

这是最容易被忽略的一层。为什么不直接用 `allow_redirects=True` 让 requests 库自动跟随？因为：

**攻击场景：开放重定向作为跳板**

```
用户输入:   https://trusted-public-site.com/redirect?target=http://169.254.169.254/latest/meta-data/
               ↓ (通过初始 SSRF 检查 — trusted-public-site.com 是公共 IP)
第 1 跳:     302 Location: http://169.254.169.254/latest/meta-data/
               ↓ (如果 requests 自动跟随，就会直接请求 EC2 元数据服务，泄露 IAM 凭证)
第 2 跳:     200 (敏感数据)
```

`trusted-public-site.com` 是合法公共网站，但存在一个开放重定向漏洞（如 `/redirect?target=...`）。初始检查只校验了第 1 跳的 URL（公共 IP，放行），如果 requests 自动跟随，**后续跳完全不受校验**，最终请求会打到内网服务。

逐跳校验确保**重定向链中的每一个 Location** 都重新经过 SSRF 检查，即使攻击者通过合法公共站点跳板也无法到达私有网络。

### B.4 为什么是 `range(10)` 跳数上限？

Python `requests` 库默认的 `max_redirects` 也是 **30**（在 `requests/adapters.py` 中 `DEFAULT_RETRIES` 相关配置）。changedetection.io 收紧到 **10 跳**，原因：

1. **防止重定向循环 DoS**：恶意站点可以构造 A→B→A→B→... 的循环重定向，如果没有跳数上限，worker 线程会被永久卡住，导致整个调度队列阻塞。
2. **合理业务边界**：合法网站几乎不会超过 5 跳重定向（通常是 http→https → www 规范化 → 单点登录回调 → 最终页面）。10 跳对所有合法场景都足够。
3. **比 requests 默认更严格**：30 跳的循环会消耗显著的网络 I/O 时间，10 跳在安全性和可用性之间取平衡。

### B.5 为什么重定向要退化为 GET、丢 body？

代码 L120：`r = session.request('GET', redirect_url, ...)`

这符合 RFC 7231（HTTP/1.1 语义）的建议：
- 301/302/303：客户端**可以**将 POST 改为 GET（303 明确要求改为 GET）
- 307/308：**必须**保持原 method

changedetection.io 没有区分 301/302/303/307/308，统一用 GET。这是务实的选择：
- changedetection.io 的主要场景是 GET 请求（抓取网页内容），method 保持不是高优先级需求
- 如果传递 body，可能泄露 POST body 给第三方重定向目标（安全隐患）
- 实现简单，不需要维护额外的状态判断

### B.6 SSRF 防护链路总览

```
用户提交 URL
    │
    ▼
表单验证 is_safe_valid_url()               ← 第一道门：URL 格式 + 协议 + DNS 预检（可缓存）
    │
    ▼
Fetch 前 is_url_private_or_parser_confused  ← 第二道门：fetch-time DNS 重新解析 + 双解析器校验
    │
    ▼
发起 HTTP 请求 (allow_redirects=False)
    │
    ▼
收到 3xx 响应
    │
    ▼
每一跳 is_url_private_or_parser_confused    ← 第三道门：逐跳重定向校验，防止开放重定向跳板
    │（最多 10 跳）
    ▼
非 3xx 响应 → 处理内容
```

### B.7 可绕过点与局限

1. **DNS Rebinding 的时间窗口**：`is_url_private_or_parser_confused` 在 `session.request()` **之前**做 DNS 解析，而 `requests` 内部会再次做 DNS 解析。两次解析之间仍有一个毫秒级的时间窗口，攻击者如果能精确控制 DNS TTL，可以在两次解析之间切换 IP。这是所有 DNS 级 SSRF 防护的固有缺陷，除非使用自定义 DNS 解析器并缓存结果。

2. **HTTP → HTTPS 重定向后的证书问题**：代码中 `verify=False` 禁用了证书校验，这降低了中间人攻击的防护，但在重定向场景中主要是为了避免自签名证书导致抓取失败，与 SSRF 防护无关。

3. **代理绕过**：如果配置了 SOCKS/HTTP 代理，`is_url_private_or_parser_confused` 检查的是本地 DNS 解析结果，但实际请求通过代理发出——代理可能有不同的 DNS 解析（如代理位于内网）。不过这属于运维配置风险，非代码漏洞。

---

## 附录 C：三级 headers.txt 合并的完整命名规则

### C.1 合并顺序与优先级

[store/__init__.py#get_all_headers_in_textfile_for_watch()](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/store/__init__.py#L900-L934)

合并顺序（后写入覆盖先写入，优先级从低到高）：

| 优先级 | 级别 | 文件路径 | 作用范围 |
|:---:|------|----------|----------|
| 1 低 | 全局 | `{datastore_path}/headers.txt` | 所有 watch |
| 2 中 | Watch 级 | `{watch.data_dir}/headers.txt` | 单个 watch（按 UUID） |
| 3 高 | Tag 级 | `{datastore_path}/headers-{sanitized_tag}.txt` | 挂载了该 tag 的所有 watch |
| 4 最高 | settings.headers | `datastore.data['settings']['headers']` | 通过 UI 设置页配置（见调用方顺序） |

每个文件都经过同一个解析函数 `parse_headers_from_text_file()` 读取（逐行解析 `Key: Value` 格式），如果文件不存在则静默跳过。每个文件的解析错误只记录日志，不影响其他文件。

### C.2 Tag 级文件名的精确生成规则（经 Python 3.14 实证验证）

Tag 级 headers.txt 的文件名不是简单的 `tag-title.txt`，而是经过严格清洗的。代码 L926：

```python
fname = "headers-" + re.sub(r'[\W_]', '', tag.get('title')).lower().strip() + ".txt"
```

分步拆解：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | `tag.get('title')` | 取 tag 的显示名称 |
| 2 | `re.sub(r'[\W_]', '', …)` | 用正则删除匹配字符 |
| 3 | `.lower()` | 转为小写（Unicode-aware） |
| 4 | `.strip()` | 去除首尾空白（通常已空，防御性处理） |
| 5 | 拼接 `"headers-" + 结果 + ".txt"` | 最终文件名 |

#### 正则 `[\W_]` 在 Python 3 Unicode 模式下的精确语义

这是最容易被误解的部分。Python 3 中 `re` 模块**默认启用 Unicode 模式**（即 `re.UNICODE` 是隐含标志），`\w` 和 `\W` 的定义范围远大于 ASCII：

- **`\w`** = 所有 Unicode Letter（中/日/韩/西里尔/拉丁字母等） + 所有 Unicode Digit（阿拉伯数字、阿拉伯文数字等） + 下划线 `_`
- **`\W`** = **`\w` 的补集**，即：空格、制表符、换行、标点符号、货币符号、数学符号、emoji、零宽字符、连接符等
- **`[\W_]`** = `\W` **加上显式的下划线**。因为 `\W` 是 `\w` 的补集，而 `\w` 包含下划线，所以 `\W` **不**匹配下划线——必须手动把 `_` 加进字符类。

**最终保留的字符**（即 **不**匹配 `[\W_]` 的字符）：

| 字符类别 | 是否保留 | 举例 |
|----------|:---:|------|
| ASCII 字母 `[a-zA-Z]` | ✅ 保留 | `A` `z` |
| ASCII 数字 `[0-9]` | ✅ 保留 | `0` `9` |
| 下划线 `_` | ❌ **删除**（显式加在正则里） | `_` |
| 中文（CJK 汉字） | ✅ **保留**（属于 Unicode Letter） | `生` `产` `环` `境` |
| 日文假名/汉字 | ✅ **保留**（属于 Unicode Letter） | `テ` `ス` `ト` |
| 韩文谚文 | ✅ **保留**（属于 Unicode Letter） | `테` `스` `트` |
| 西里尔字母 | ✅ **保留**（属于 Unicode Letter） | `П` `р` `и` |
| 带重音的拉丁字母 | ✅ **保留**（属于 Unicode Letter） | `é` `ü` `ñ` `ß` |
| Unicode 数字（非 ASCII） | ✅ **保留**（属于 Unicode Digit） | `١` `٢`（阿拉伯文数字） |
| 空格 ` `、制表符 `\t`、换行 `\n` | ❌ 删除 | |
| ASCII 标点 `!@#$%^&*()-=+[]{};:'",.<>?/\\\|` | ❌ 删除 | |
| 全角标点 `，。！？：；「」` | ❌ 删除 | `，` `！` |
| 货币符号 | ❌ 删除 | `$` `€` `£` `¥` |
| 数学符号 | ❌ 删除 | `+` `=` `×` `÷` |
| Emoji | ❌ **删除**（不属于 Unicode Letter/Digit） | `😀` `🎉` `🔥` |
| 零宽字符 | ❌ 删除 | U+200B U+200C |
| 各种连字符/破折号 | ❌ 删除 | `-` `–` `—` |

**一句话概括**：`[\W_]` 删除下划线以及**所有不属于 Unicode 字母/数字**的字符。中文、日文、韩文、西里尔文、阿拉伯文数字等非 ASCII 字母/数字都会被**保留**。

### C.3 Tag 名称到文件名的映射示例（经 Python 3.14 实证验证）

| Tag 显示名称 | 每步清洗过程 | 最终文件名 |
|-------------|-------------|-----------|
| `Production` | `Production` → `Production` → `production` | `headers-production.txt` |
| `My API Key` | `My API Key` → 删空格 → `MyAPIKey` → `myapikey` | `headers-myapikey.txt` |
| `Dev & Test!` | `Dev & Test!` → 删 `&` `!` ` ` → `DevTest` → `devtest` | `headers-devtest.txt` |
| `api_v2` | `api_v2` → 删下划线 → `apiv2` → `apiv2` | `headers-apiv2.txt` |
| `生产环境` | `生产环境` → **保留中文** → `生产环境` → `生产环境` | `headers-生产环境.txt` ✅ |
| `测试环境` | `测试环境` → 保留中文 → `测试环境` → `测试环境` | `headers-测试环境.txt` ✅ |
| `Auth-Token: Bearer` | `Auth-Token: Bearer` → 删 `-` `:` ` ` → `AuthTokenBearer` → `authtokenbearer` | `headers-authtokenbearer.txt` |
| `😀 emoji tag` | `😀 emoji tag` → 删 emoji 和空格 → `emojitag` → `emojitag` | `headers-emojitag.txt` |
| `  Trim Me  ` | `  Trim Me  ` → 删空格 → `TrimMe` → `trimme` | `headers-trimme.txt` |
| `テスト環境`（日文） | `テスト環境` → 保留日文 → `テスト環境` → `テスト環境` | `headers-テスト環境.txt` ✅ |
| `테스트환경`（韩文） | `테스트환경` → 保留韩文 → `테스트환경` → `테스트환경` | `headers-테스트환경.txt` ✅ |
| `Привет мир`（西里尔） | `Привет мир` → 删空格 → `Приветмир` → `приветмир` | `headers-приветмир.txt` ✅ |
| `über naïve`（拉丁重音） | `über naïve` → 删空格 → `übernaïve` → `übernaïve` | `headers-übernaïve.txt` ✅ |
| `café-résumé` | `café-résumé` → 删 `-` → `caférésumé` → `caférésumé` | `headers-caférésumé.txt` ✅ |
| `生产环境API-v2!测试😀` | `生产环境API-v2!测试😀` → 删 `-` `!` emoji → `生产环境APIv2测试` → `生产环境apiv2测试` | `headers-生产环境apiv2测试.txt` ✅ |
| `$100€50£25` | `$100€50£25` → 删货币符号 → `1005025` → `1005025` | `headers-1005025.txt` |
| `[foo]{bar}(baz)` | `[foo]{bar}(baz)` → 删括号 → `foobarbaz` → `foobarbaz` | `headers-foobarbaz.txt` |
| `😀😁😂`（纯 emoji） | `😀😁😂` → 全删 → `` → `` → `headers-.txt` | `headers-.txt` ⚠️ |

### C.4 需要注意的命名陷阱

**陷阱 1：纯 emoji/纯符号 Tag 名会生成 `headers-.txt`**

如果 Tag 名**完全由 emoji 或非字母数字符号**组成（如 `😀😁😂`、`$€£¥`），正则会把所有字符全部删除，得到空字符串，最终文件名是 `headers-.txt`。

- **不会**出现纯中文 Tag 生成 `headers-.txt` 的情况——中文属于 Unicode Letter，会被完整保留
- 多个纯 emoji Tag（如 `😀` 和 `🎉`）会**共享同一个 `headers-.txt` 文件**，互相覆盖

**陷阱 2：仅大小写不同的 Tag 名会冲突**

清洗时会 `.lower()`（Unicode -aware），所以：
- Tag `API`、`api`、`Api` → 都生成 `headers-api.txt`
- Tag `Привет`、`привет` → 都生成 `headers-привет.txt`

**陷阱 3：标点、空格、下划线被完全移除导致意外合并**

以下所有 Tag 都会生成同一个文件 `headers-userauth.txt`：
- `user auth`（空格被删）
- `user-auth`（连字符被删）
- `user_auth`（下划线被删）
- `UserAuth`（直接 `.lower()`）

类似地，`api-v2` 和 `apiv2` 都生成 `headers-apiv2.txt`。

**陷阱 4：多 Tag 合并顺序不确定**

如果一个 watch 被同时打上 `tagA` 和 `tagB`，代码通过 `tags.items()` 遍历 tag 字典（Python 3.7+ 字典保持插入顺序）。如果两个 tag 的 headers.txt 中有同名 header，**后插入的 tag 的值会覆盖先插入的 tag**。而 tag 的插入顺序由创建顺序决定，用户无法在 UI 中控制。

**陷阱 5：非 ASCII 文件名的跨平台兼容性**

中文、日文、西里尔字母等会被完整保留在文件名中。这在大多数现代操作系统上没问题，但需要注意：
- Windows 传统 FAT32 不支持 Unicode 文件名（但 NTFS 支持）
- 通过非 Unicode FTP/SMB 协议传输时文件名可能乱码
- 某些 shell 脚本如果假设文件名是纯 ASCII 可能出错

这与 `[\W_]` 正则最初意图"生成纯 ASCII 安全文件名"的预期完全相反——该正则在 Python 3 Unicode 模式下并**不能**保证输出是纯 ASCII。

### C.5 全局 / Watch 级 / Tag 级的路径解析

```
datastore/
├── headers.txt                              ← 全局（优先级 1）
├── headers-production.txt                   ← Tag "Production"
├── headers-myapikey.txt                     ← Tag "My API Key"
├── headers-生产环境.txt                       ← Tag "生产环境" ✅（中文被保留）
├── headers-测试环境.txt                       ← Tag "测试环境" ✅（中文被保留）
├── headers-テスト.txt                         ← Tag "テスト" ✅（日文被保留）
├── headers-.txt                             ← 纯 emoji/纯符号 Tag（⚠️ 共享）
├── 1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d/
│   ├── watch.json                           ← Watch 元数据
│   └── headers.txt                          ← Watch 级（优先级 2）
└── 9f8e7d6c-5b4a-3928-1706-958473625140/
    ├── watch.json
    └── headers.txt
```

### C.6 解析函数 parse_headers_from_text_file() 的格式

文件名虽然叫 `.txt`，但内容格式有严格要求。每个文件是逐行 `Key: Value` 格式，与 HTTP 请求头的文本格式一致：

```
# 以 # 开头的行是注释
Authorization: Bearer my-secret-token
X-Custom-Header: some value

# 空行被忽略
Accept: application/json
```

如果某一行没有冒号，该行会被忽略。Value 中可以包含冒号（如 `X-Forwarded-Proto: https` 没问题）。

### C.7 与 settings.headers 的优先级关系

需要特别注意：`get_all_headers_in_textfile_for_watch()` 是**纯文件级**合并，但在调用方 `call_browser()` 中（[processors/base.py#L206-L208](file:///d:/fz/0601-2/solo-dogfeeding/code/3-changedetection.io/changedetectionio/processors/base.py#L206-L208)），完整的合并顺序是：

```python
request_headers.update(self.watch.get('headers', {}))                  # ① Watch dict 中的 headers
request_headers.update(self.datastore.get_all_base_headers())          # ② settings.headers（UI 设置页）
request_headers.update(self.datastore.get_all_headers_in_textfile_for_watch(uuid=...))  # ③ 三级文件
```

所以最终优先级（后覆盖前）：
**Watch dict → settings.headers → 全局 headers.txt → Watch 级 headers.txt → Tag 级 headers.txt**

文件级的优先级最高，tag 级是文件级中的最高——运维人员通过文件部署的 header 始终可以覆盖用户在 UI 中的任何配置。这是一种"运维控制"的设计取舍：允许运维团队在不改动数据库的情况下，通过文件系统强制注入安全头（如 `Authorization`、`X-Forwarded-*`），且用户无法绕过。
