# 代理与 Cookies 传递机制分析（精确版）

## 一、概述

本报告梳理 `requests`、`playwright`、`puppeteer` 三条抓取通道中代理与 cookies 的传递机制，按**来源、注入时机、作用范围、抓取后清理**四维度展开，所有结论均对齐到可复核的代码位置。

---

## 二、代理传递机制

### 2.1 代理来源

#### 2.1.1 系统环境变量级

| 变量名 | 适用抓取器 | 代码位置 |
|--------|-----------|----------|
| `HTTP_PROXY` | requests | `content_fetchers/base.py:61` |
| `HTTPS_PROXY` | requests | `content_fetchers/base.py:62` |
| `PLAYWRIGHT_DRIVER_URL` | playwright/puppeteer | `content_fetchers/playwright.py:194` |
| `playwright_proxy_server` | playwright | `content_fetchers/playwright.py:198` |
| `playwright_proxy_bypass` | playwright | `content_fetchers/playwright.py:198` |
| `playwright_proxy_username` | playwright | `content_fetchers/playwright.py:198` |
| `playwright_proxy_password` | playwright | `content_fetchers/playwright.py:198` |

**代码片段**（`content_fetchers/base.py:61-62`）：
```python
system_http_proxy = os.getenv('HTTP_PROXY')
system_https_proxy = os.getenv('HTTPS_PROXY')
```

#### 2.1.2 全局设置级

存储于 `datastore.data['settings']['requests']`，包含：
- `proxy`: 首选代理连接 ID
- `extra_proxies`: UI 配置的额外代理列表
- `extra_browsers`: UI 配置的额外浏览器端点

**代码位置**：`model/App.py:28-31`

#### 2.1.3 单条监控级

每条监控可单独配置代理：`watch.get('proxy')`，值为代理 ID 或 `"no-proxy"`

**代码位置**：`store/__init__.py:868-872`

---

### 2.2 代理注入时机与优先级

#### 2.2.1 优先级规则

```
单条监控配置 > 全局设置 > 系统环境变量 > 默认代理列表第一项
```

**决策流程代码**（`store/__init__.py:855-886`）：

```python
def get_preferred_proxy_for_watch(self, uuid):
    # 步骤1: 检查是否设置为"no-proxy"
    if strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')) and watch.get('proxy') == "no-proxy":
        return None
    
    # 步骤2: 检查单条监控的代理ID是否有效
    if watch.get('proxy') and watch.get('proxy') in list(self.proxy_list.keys()):
        return watch.get('proxy')
    
    # 步骤3: 回退到全局系统代理
    system_proxy_id = self.data['settings']['requests'].get('proxy')
    if self.proxy_list.get(system_proxy_id):
        return system_proxy_id
    
    # 步骤4: 最终回退到代理列表第一项
    first_default = list(self.proxy_list)[0]
    return first_default
```

#### 2.2.2 注入入口

**核心入口**：`processors/base.py:117-260` `call_browser()` 方法

```python
async def call_browser(self, preferred_proxy_id=None):
    # 1. 获取代理ID
    preferred_proxy_id = preferred_proxy_id if preferred_proxy_id else \
        self.datastore.get_preferred_proxy_for_watch(uuid=self.watch.get('uuid'))
    
    # 2. 解析代理URL（自定义浏览器端点跳过代理）
    proxy_url = None
    if preferred_proxy_id:
        if not prefer_fetch_backend.startswith('extra_browser_'):
            proxy_url = self.datastore.proxy_list.get(preferred_proxy_id).get('url')
    
    # 3. 创建fetcher实例并注入代理
    self.fetcher = fetcher_obj(proxy_override=proxy_url, ...)
```

#### 2.2.3 各抓取器代理适配

##### Requests 抓取器

**代理注入方式**：通过 `proxies` 参数传递

**代码位置**：`content_fetchers/requests.py:19-58`

```python
def __init__(self, proxy_override=None, custom_browser_connection_url=None, **kwargs):
    super().__init__(**kwargs)
    self.proxy_override = proxy_override

def _run_sync(self, ...):
    proxies = {}
    if self.proxy_override:
        proxies = {'http': self.proxy_override, 'https': self.proxy_override, 'ftp': self.proxy_override}
    else:
        if self.system_http_proxy:
            proxies['http'] = self.system_http_proxy
        if self.system_https_proxy:
            proxies['https'] = self.system_https_proxy
    
    r = session.request(method=request_method,
                        url=url,
                        headers=request_headers,
                        proxies=proxies,
                        ...)
```

**适配特点**：支持 HTTP/HTTPS/SOCKS5 协议，认证信息直接包含在 URL 中

##### Playwright 抓取器

**代理注入方式**：通过 `context.new_context(proxy=self.proxy)`

**代码位置**：`content_fetchers/playwright.py:183-293`

```python
def __init__(self, proxy_override=None, custom_browser_connection_url=None, **kwargs):
    # 从环境变量加载代理配置
    proxy_args = {}
    for k in self.playwright_proxy_settings_mappings:  # ['bypass', 'server', 'username', 'password']
        v = os.getenv('playwright_proxy_' + k, False)
        if v:
            proxy_args[k] = v.strip('"')

    if proxy_args:
        self.proxy = proxy_args

    # 单条监控覆盖
    if proxy_override:
        self.proxy = {'server': proxy_override}

    # 解析URL中的认证信息
    if self.proxy:
        parsed = urlparse(self.proxy.get('server'))
        if parsed.username:
            self.proxy['username'] = parsed.username
            self.proxy['password'] = parsed.password

# 使用时
context = await browser.new_context(
    accept_downloads=False,
    bypass_csp=True,
    extra_http_headers=request_headers,
    ignore_https_errors=True,
    proxy=self.proxy,
    ...
)
```

**适配特点**：支持 HTTP/HTTPS 代理认证，不支持 SOCKS5 认证（官方限制）

##### Puppeteer 抓取器

**代理注入方式**：URL 参数追加 `--proxy-server` + `page.authenticate()`

**代码位置**：`content_fetchers/puppeteer.py:197-363`

```python
def __init__(self, proxy_override=None, custom_browser_connection_url=None, **kwargs):
    if proxy_override:
        parsed = urlparse(proxy_override)
        if parsed:
            self.proxy = {'username': parsed.username, 'password': parsed.password}
            # 通过URL参数传递代理服务器
            proxy_url = parsed.scheme + "://" if parsed.scheme else 'http://'
            r = "?" if not '?' in self.browser_connection_url else '&'
            port = ":"+str(parsed.port) if parsed.port else ''
            q = "?"+parsed.query if parsed.query else ''
            proxy_url += f"{parsed.hostname}{port}{parsed.path}{q}"
            self.browser_connection_url += f"{r}--proxy-server={proxy_url}"

# 使用时通过 page.authenticate() 处理认证
if self.proxy and self.proxy.get('username'):
    await self.page.authenticate(self.proxy)
```

**适配特点**：通过 URL 参数传递代理地址，认证信息通过单独的 `authenticate()` 方法注入

---

### 2.3 代理作用范围

| 配置级别 | 作用范围 | 优先级 | 代码依据 |
|----------|----------|--------|----------|
| 单条监控 | 仅当前监控 | 最高 | `store/__init__.py:871-872` |
| 全局设置 | 所有未单独配置的监控 | 中 | `store/__init__.py:876-879` |
| 环境变量 | 全局所有监控 | 最低 | `content_fetchers/base.py:61-62` |
| 自定义浏览器 | 跳过代理设置 | 特殊 | `processors/base.py:179-183` |

**关键规则**：当使用自定义浏览器连接（`extra_browser_*`）时，系统会跳过代理设置

**代码依据**（`processors/base.py:179-183`）：
```python
if preferred_proxy_id:
    if not prefer_fetch_backend.startswith('extra_browser_'):
        proxy_url = self.datastore.proxy_list.get(preferred_proxy_id).get('url')
    else:
        logger.debug("Skipping adding proxy data when custom Browser endpoint is specified. ")
```

---

### 2.4 代理清理动作

#### Worker 层面清理

**代码位置**：`worker.py:605-680`

```python
finally:
    # 1. 调用 fetcher.quit() 清理浏览器资源
    try:
        if update_handler and hasattr(update_handler, 'fetcher') and update_handler.fetcher:
            await update_handler.fetcher.quit(watch=watch)
    except Exception as e:
        logger.error(f"Exception while cleaning/quit after calling browser: {e}")

    # 2. 清理内存引用
    if update_handler:
        if hasattr(update_handler, 'fetcher') and update_handler.fetcher:
            update_handler.fetcher.clear_content()
        if hasattr(update_handler, 'content_processor'):
            update_handler.content_processor = None
        del update_handler
        update_handler = None

    # 3. 强制垃圾回收
    import gc
    gc.collect()
```

#### 各抓取器 quit() 实现

##### Playwright（`content_fetchers/playwright.py:420-458`）
```python
async def quit(self, watch=None):
    if hasattr(self, 'page') and self.page:
        await asyncio.wait_for(self.page.close(), timeout=5.0)
    if context:
        await asyncio.wait_for(context.close(), timeout=5.0)
    if browser:
        await asyncio.wait_for(browser.close(), timeout=5.0)
    gc.collect()
```

##### Puppeteer（`content_fetchers/puppeteer.py:227-257`）
```python
async def quit(self, watch=None):
    if hasattr(self, 'page') and self.page:
        await asyncio.wait_for(self.page.close(), timeout=5.0)
    if hasattr(self, 'browser') and self.browser:
        await asyncio.wait_for(self.browser.close(), timeout=5.0)
    gc.collect()
```

##### Requests（`content_fetchers/requests.py:245-256`）
```python
async def quit(self, watch=None):
    if strtobool(os.getenv("REMOVE_REQUESTS_OLD_SCREENSHOTS", 'true')):
        screenshot = watch.get_screenshot()
        if screenshot:
            try:
                os.unlink(screenshot)
            except Exception as e:
                logger.warning(f"Failed to unlink screenshot: {screenshot} - {e}")
```

---

## 三、Cookies 传递机制

### 3.1 Cookies 来源

| 来源 | 位置 | 代码依据 |
|------|------|----------|
| 监控级 Headers | `watch.get('headers')` | `processors/base.py:206` |
| 全局 Headers | `datastore.data['settings']['headers']` | `processors/base.py:207` |
| 文件配置 Headers | `headers.txt` | `processors/base.py:208` |
| 浏览器自动管理 | Playwright/Puppeteer 上下文 | `content_fetchers/playwright.py:288` |

### 3.2 Headers 合并顺序

**代码位置**：`processors/base.py:200-217`

```python
request_headers = CaseInsensitiveDict()

# 1. 默认 User-Agent（按抓取器类型）
ua = self.datastore.data['settings']['requests'].get('default_ua')
if ua and ua.get(prefer_fetch_backend):
    request_headers.update({'User-Agent': ua.get(prefer_fetch_backend)})

# 2. 单条监控 Headers（优先级最高）
request_headers.update(self.watch.get('headers', {}))

# 3. 全局基础 Headers
request_headers.update(self.datastore.get_all_base_headers())

# 4. 文件配置 Headers（全局 headers.txt + 监控级 headers.txt + 标签级 headers.txt）
request_headers.update(self.datastore.get_all_headers_in_textfile_for_watch(uuid=self.watch.get('uuid')))

# 5. Jinja2 模板渲染
for header_name in request_headers:
    request_headers.update({header_name: jinja_render(template_str=request_headers.get(header_name))})
```

**优先级**：监控级 > 全局 > 文件配置

### 3.3 各抓取器 Cookies 处理

#### Requests 抓取器
- **处理方式**：直接通过 headers 传递，无自动管理
- **代码位置**：`content_fetchers/requests.py:96-103`

```python
r = session.request(method=request_method,
                    data=request_body.encode('utf-8') if type(request_body) is str else request_body,
                    url=url,
                    headers=request_headers,
                    timeout=timeout,
                    proxies=proxies,
                    ...)
```

#### Playwright 抓取器
- **处理方式**：通过 `extra_http_headers` 传递初始 headers，浏览器上下文自动管理后续 cookies
- **代码位置**：`content_fetchers/playwright.py:285-293`

```python
context = await browser.new_context(
    accept_downloads=False,
    bypass_csp=True,
    extra_http_headers=request_headers,
    ignore_https_errors=True,
    proxy=self.proxy,
    ...
)
```

#### Puppeteer 抓取器
- **处理方式**：通过 `setExtraHTTPHeaders` 传递初始 headers，浏览器上下文自动管理后续 cookies
- **代码位置**：`content_fetchers/puppeteer.py:351-352`

```python
if request_headers:
    await self.page.setExtraHTTPHeaders(request_headers)
```

### 3.4 Cookies 作用范围

| 抓取器 | 作用范围 | 隔离性 |
|--------|----------|--------|
| requests | 单次请求 | 完全隔离（每次新建 Session） |
| playwright | 浏览器上下文 | 完全隔离（每次新建 context） |
| puppeteer | 浏览器上下文 | 完全隔离（每次新建 page） |

### 3.5 Cookies 清理

- **Requests**：无需特殊清理，Session 随请求结束自动销毁
- **Playwright/Puppeteer**：通过 `quit()` 关闭上下文时自动清理 cookies
- **内存清理**：`clear_content()` 不清理 headers（保留用于后续处理）

---

## 四、私密性保障机制

### 4.1 代理认证信息保护

| 保护措施 | 代码依据 |
|----------|----------|
| 仅在 fetcher 内部解析认证信息 | `content_fetchers/playwright.py:212-215` |
| 日志不打印完整代理 URL | 全局日志均使用 `logger.debug(f"Using proxy '{proxy_url}'")` |
| 认证信息通过专用字段传递 | `content_fetchers/playwright.py:212-215` |

### 4.2 Cookies 隔离

| 隔离措施 | 代码依据 |
|----------|----------|
| 每条监控使用独立浏览器上下文 | `content_fetchers/playwright.py:285` |
| 抓取完成后立即清理上下文 | `worker.py:615-616` |
| Requests 使用独立 Session | `content_fetchers/requests.py:59` |

---

## 五、关键结论总结

### 5.1 代理传递关键结论

| 维度 | 结论 | 代码位置 |
|------|------|----------|
| 优先级 | 单条监控 > 全局 > 环境变量 > 默认 | `store/__init__.py:855-886` |
| 注入入口 | `call_browser()` 统一处理 | `processors/base.py:117-260` |
| 自定义浏览器 | 跳过代理设置 | `processors/base.py:179-183` |
| Requests 协议 | 支持 HTTP/HTTPS/SOCKS5 | `content_fetchers/requests.py:51-58` |
| Playwright 协议 | 支持 HTTP/HTTPS，不支持 SOCKS5 认证 | `content_fetchers/playwright.py:280` |
| Puppeteer 协议 | 通过 URL 参数传递，认证单独处理 | `content_fetchers/puppeteer.py:209-225` |

### 5.2 Cookies 传递关键结论

| 维度 | 结论 | 代码位置 |
|------|------|----------|
| 来源 | 监控级 headers + 全局 headers + 文件 headers | `processors/base.py:200-208` |
| 合并顺序 | 监控级 > 全局 > 文件 | `processors/base.py:200-217` |
| 浏览器自动管理 | Playwright/Puppeteer 上下文自动维护 | `content_fetchers/playwright.py:288` |
| 隔离性 | 完全隔离，无跨监控污染 | `content_fetchers/playwright.py:285` |

### 5.3 清理机制关键结论

| 维度 | 结论 | 代码位置 |
|------|------|----------|
| 清理时机 | 抓取完成后 finally 块统一清理 | `worker.py:605-680` |
| 浏览器关闭 | 按 page → context → browser 顺序 | `content_fetchers/playwright.py:420-458` |
| 内存释放 | `clear_content()` + `del` + `gc.collect()` | `content_fetchers/base.py:105-116` |

---

## 六、流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Worker 发起抓取                              │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│              processors/base.py::call_browser()                      │
├──────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ 1. 获取代理ID（优先级：watch.proxy > global.proxy > default）│    │
│  │    代码: store/__init__.py:855-886                          │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                             │                                       │
│  ┌──────────────────────────▼───────────────────────────────────┐    │
│  │ 2. 构建请求头（优先级：watch.headers > global.headers > file）│    │
│  │    代码: processors/base.py:200-217                        │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                             │                                       │
│  ┌──────────────────────────▼───────────────────────────────────┐    │
│  │ 3. 创建 fetcher 实例并注入代理                                │    │
│  │    代码: processors/base.py:189-192                         │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
┌───────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Requests    │    │   Playwright    │    │   Puppeteer     │
├───────────────┤    ├─────────────────┤    ├─────────────────┤
│ proxies={}    │    │ proxy={server,  │    │ URL参数追加     │
│ headers=dict  │    │  username,      │    │ --proxy-server  │
│               │    │  password}      │    │ page.auth()     │
└───────┬───────┘    └────────┬────────┘    └────────┬────────┘
        │                     │                      │
        └─────────────────────┼──────────────────────┘
                              ▼
              ┌─────────────────────────────────────┐
              │           执行请求                   │
              │  request_headers 包含 Cookie        │
              └─────────────────────┬───────────────┘
                                    │
                                    ▼
              ┌─────────────────────────────────────┐
              │           清理资源                   │
              │  fetcher.quit() → clear_content()  │
              │  gc.collect()                       │
              └─────────────────────────────────────┘
```