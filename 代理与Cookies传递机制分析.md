# 代理与 Cookies 在不同抓取通道之间的传递机制分析

## 一、概述

changedetection.io 支持多种内容抓取器（fetchers），包括 `requests`、`playwright` 和 `puppeteer`。代理和 cookies 在这些抓取通道之间的传递需要兼顾私密性和请求体适配。

---

## 二、代理配置来源

### 2.1 系统环境变量级别

| 变量名 | 适用抓取器 | 说明 |
|--------|-----------|------|
| `HTTP_PROXY` | requests | HTTP 请求代理地址 |
| `HTTPS_PROXY` | requests | HTTPS 请求代理地址 |
| `PLAYWRIGHT_DRIVER_URL` | playwright/puppeteer | 浏览器连接地址 |
| `playwright_proxy_server` | playwright | Playwright 代理服务器地址 |
| `playwright_proxy_bypass` | playwright | 代理绕过规则 |
| `playwright_proxy_username` | playwright | 代理认证用户名 |
| `playwright_proxy_password` | playwright | 代理认证密码 |

**代码位置**：`content_fetchers/base.py:61-62`、`content_fetchers/playwright.py:197-204`

### 2.2 全局设置级别

存储于 `datastore.data['settings']['requests']`：

```python
{
    'proxy': None,  # 首选代理连接ID
    'extra_proxies': [],  # UI配置的额外代理列表
    'extra_browsers': [],  # UI配置的额外浏览器端点
}
```

**代码位置**：`model/App.py:28-31`

### 2.3 单条监控级别

每条监控可以单独配置代理：

```python
watch.get('proxy')  # 代理ID或 "no-proxy"
```

**代码位置**：`store/__init__.py:855-886`

---

## 三、代理优先级与注入时机

### 3.1 代理选择优先级

```
单条监控配置 > 全局设置 > 系统环境变量 > 默认值
```

**决策流程**（`store/__init__.py:855-886`）：

1. **第一步**：检查监控是否设置了 `proxy` 且值为 `no-proxy`
   - 如果是，返回 `None`（不使用代理）

2. **第二步**：检查监控的代理ID是否在代理列表中有效
   - 如果有效，返回该代理ID

3. **第三步**：回退到全局系统代理设置
   - 检查 `settings['requests']['proxy']` 是否有效

4. **第四步**：最终回退到代理列表的第一个代理

### 3.2 代理注入流程

**核心入口**：`processors/base.py:117-257` `call_browser()` 方法

```python
async def call_browser(self, preferred_proxy_id=None):
    # 1. 获取代理ID（优先传入的参数，否则从datastore获取）
    preferred_proxy_id = preferred_proxy_id if preferred_proxy_id else \
        self.datastore.get_preferred_proxy_for_watch(uuid=self.watch.get('uuid'))
    
    # 2. 解析代理URL
    proxy_url = None
    if preferred_proxy_id:
        if not prefer_fetch_backend.startswith('extra_browser_'):
            proxy_url = self.datastore.proxy_list.get(preferred_proxy_id).get('url')
    
    # 3. 创建fetcher实例并注入代理
    self.fetcher = fetcher_obj(proxy_override=proxy_url,
                               custom_browser_connection_url=custom_browser_connection_url,
                               screenshot_format=self.screenshot_format)
```

### 3.3 各抓取器的代理适配

| 抓取器 | 代理注入方式 | 适配特点 |
|--------|-------------|----------|
| **requests** | `proxies` 参数传递 | 支持 HTTP/HTTPS/SOCKS5 |
| **playwright** | `context.new_context(proxy=self.proxy)` | 支持认证，不支持SOCKS5认证 |
| **puppeteer** | URL参数追加 `--proxy-server` | 通过 `page.authenticate()` 处理认证 |

#### Requests 抓取器（`content_fetchers/requests.py:51-58`）

```python
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

#### Playwright 抓取器（`content_fetchers/playwright.py:183-215`）

```python
def __init__(self, proxy_override=None, custom_browser_connection_url=None, **kwargs):
    # 从环境变量加载代理配置
    proxy_args = {}
    for k in self.playwright_proxy_settings_mappings:
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
    proxy=self.proxy,
    ...
)
```

#### Puppeteer 抓取器（`content_fetchers/puppeteer.py:209-225`）

```python
def __init__(self, proxy_override=None, custom_browser_connection_url=None, **kwargs):
    if proxy_override:
        parsed = urlparse(proxy_override)
        if parsed:
            self.proxy = {'username': parsed.username, 'password': parsed.password}
            # 通过URL参数传递代理服务器
            proxy_url = parsed.scheme + "://" if parsed.scheme else 'http://'
            r = "?" if not '?' in self.browser_connection_url else '&'
            proxy_url += f"{parsed.hostname}{port}{parsed.path}{q}"
            self.browser_connection_url += f"{r}--proxy-server={proxy_url}"

# 使用时通过 page.authenticate() 处理认证
if self.proxy and self.proxy.get('username'):
    await self.page.authenticate(self.proxy)
```

---

## 四、Cookies 传递机制

### 4.1 Cookies 的来源

Cookies 在系统中主要通过以下方式传递：

| 来源 | 位置 | 说明 |
|------|------|------|
| **自定义 Headers** | `watch.get('headers')` | 用户手动配置的 Cookie header |
| **全局 Headers** | `datastore.data['settings']['headers']` | 全局配置的 headers |
| **文件配置** | `headers.txt` | 全局或监控级别的 headers 文件 |
| **浏览器自动管理** | Playwright/Puppeteer | 浏览器上下文自动维护 |

### 4.2 Headers 合并顺序（`processors/base.py:200-208`）

```python
request_headers = CaseInsensitiveDict()

# 1. 默认 User-Agent
ua = self.datastore.data['settings']['requests'].get('default_ua')
if ua and ua.get(prefer_fetch_backend):
    request_headers.update({'User-Agent': ua.get(prefer_fetch_backend)})

# 2. 单条监控 Headers
request_headers.update(self.watch.get('headers', {}))

# 3. 全局基础 Headers
request_headers.update(self.datastore.get_all_base_headers())

# 4. 文件配置 Headers
request_headers.update(self.datastore.get_all_headers_in_textfile_for_watch(uuid=self.watch.get('uuid')))

# 5. Jinja2 模板渲染
for header_name in request_headers:
    request_headers.update({header_name: jinja_render(template_str=request_headers.get(header_name))})
```

### 4.3 Cookies 在各抓取器中的处理

#### Requests 抓取器

直接通过 headers 传递，浏览器不会自动管理：

```python
r = session.request(..., headers=request_headers, ...)
```

#### Playwright 抓取器

通过 `extra_http_headers` 传递初始 headers，浏览器上下文会自动管理后续 cookies：

```python
context = await browser.new_context(
    extra_http_headers=request_headers,
    ...
)
```

#### Puppeteer 抓取器

同样通过 `setExtraHTTPHeaders` 传递初始 headers：

```python
if request_headers:
    await self.page.setExtraHTTPHeaders(request_headers)
```

---

## 五、影响范围

### 5.1 代理影响范围矩阵

| 配置级别 | 影响范围 | 优先级 |
|----------|----------|--------|
| **环境变量** | 全局所有监控 | 最低 |
| **全局设置** | 所有未单独配置的监控 | 中 |
| **单条监控** | 仅当前监控 | 最高 |
| **自定义浏览器** | 跳过代理设置 | 特殊 |

> **注意**：当使用自定义浏览器连接（`extra_browser_*`）时，系统会跳过代理设置（`processors/base.py:179-183`）。

### 5.2 私密性保障

1. **代理认证信息保护**：
   - 从 URL 解析用户名密码后，通过专用字段传递
   - 不会在日志中打印完整认证信息

2. **Cookies 隔离**：
   - 每条监控使用独立的浏览器上下文（Playwright/Puppeteer）
   - Requests 抓取器每次请求都是独立的 Session

3. **清理机制**：
   - 抓取完成后立即清理浏览器上下文
   - 避免跨监控的 cookies 泄漏

---

## 六、抓取后的清理动作

### 6.1 清理触发点

**Worker 层面**（`worker.py:605-680`）：

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

### 6.2 各抓取器的 quit() 实现

#### Playwright（`content_fetchers/playwright.py:420-458`）

```python
async def quit(self, watch=None):
    # 关闭页面
    if hasattr(self, 'page') and self.page:
        await asyncio.wait_for(self.page.close(), timeout=5.0)
    
    # 关闭上下文
    if context:
        await asyncio.wait_for(context.close(), timeout=5.0)
    
    # 关闭浏览器连接
    if browser:
        await asyncio.wait_for(browser.close(), timeout=5.0)
    
    # 强制垃圾回收
    gc.collect()
```

#### Puppeteer（`content_fetchers/puppeteer.py:227-257`）

```python
async def quit(self, watch=None):
    # 关闭页面
    if hasattr(self, 'page') and self.page:
        await asyncio.wait_for(self.page.close(), timeout=5.0)
    
    # 关闭浏览器连接
    if hasattr(self, 'browser') and self.browser:
        await asyncio.wait_for(self.browser.close(), timeout=5.0)
    
    # 强制垃圾回收
    gc.collect()
```

#### Requests（`content_fetchers/requests.py:245-256`）

```python
async def quit(self, watch=None):
    # 删除旧截图（如果切换到requests抓取器）
    if strtobool(os.getenv("REMOVE_REQUESTS_OLD_SCREENSHOTS", 'true')):
        screenshot = watch.get_screenshot()
        if screenshot:
            try:
                os.unlink(screenshot)
            except Exception as e:
                logger.warning(f"Failed to unlink screenshot: {screenshot} - {e}")
```

### 6.3 clear_content() 方法（`content_fetchers/base.py:105-116`）

```python
def clear_content(self):
    """显式清理内存中的内容"""
    self.content = None
    if hasattr(self, 'raw_content'):
        self.raw_content = None
    self.screenshot = None
    self.xpath_data = None
    # 保留 headers 和 status_code（占用空间小）
```

---

## 七、代理与 Cookies 传递流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        请求发起（Worker）                               │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   processors/base.py::call_browser()                   │
├─────────────────────────────────────────────────────────────────────────┤
│  1. 获取代理ID:                                                        │
│     watch.proxy → settings.requests.proxy → proxy_list[0]              │
│                                                                       │
│  2. 构建请求头:                                                        │
│     default_ua + watch.headers + global.headers + file.headers         │
│                                                                       │
│  3. 创建Fetcher实例并注入代理                                           │
│     fetcher_obj(proxy_override=proxy_url)                              │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
┌───────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Requests     │    │   Playwright    │    │   Puppeteer     │
│  Fetcher      │    │   Fetcher       │    │   Fetcher       │
├───────────────┤    ├─────────────────┤    ├─────────────────┤
│ proxies={}    │    │ proxy={server,  │    │ URL参数追加     │
│ http/https    │    │  username,      │    │ --proxy-server  │
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

---

## 八、关键设计要点

### 8.1 私密性保障机制

1. **代理认证信息不暴露**：
   - URL 中的认证信息仅在 fetcher 内部解析
   - 日志中不会打印完整代理 URL

2. **浏览器上下文隔离**：
   - 每条监控使用独立的浏览器上下文
   - 避免跨监控的 cookies 污染

3. **及时清理**：
   - 抓取完成后立即关闭浏览器连接
   - 强制垃圾回收释放内存

### 8.2 请求体适配

1. **Headers 合并策略**：
   - 支持多级配置（监控 > 全局 > 文件）
   - 支持 Jinja2 模板渲染动态值

2. **代理协议适配**：
   - Requests 支持 HTTP/HTTPS/SOCKS5
   - Playwright 支持 HTTP/HTTPS（不支持 SOCKS5 认证）
   - Puppeteer 通过 URL 参数传递

3. **认证信息解析**：
   - 自动从代理 URL 中提取用户名密码
   - 根据不同抓取器的要求格式化认证信息

---

## 九、总结

| 维度 | 实现方式 |
|------|----------|
| **代理来源** | 环境变量、全局设置、单条监控、UI配置 |
| **优先级** | 单条监控 > 全局设置 > 环境变量 |
| **Cookies 传递** | 通过 headers 注入，浏览器自动管理 |
| **私密性** | 独立上下文 + 及时清理 + 日志脱敏 |
| **清理机制** | quit() 方法 + clear_content() + gc.collect() |

该设计实现了代理与 cookies 在不同抓取通道之间的安全传递，既保证了私密性，又适配了各抓取器的请求体格式。