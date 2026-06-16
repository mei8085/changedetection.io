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
