# 代理与 Cookies 传递机制分析（最终校准版）

## 一、概述

本报告针对 `requests`、`playwright`、`puppeteer` 三条抓取通道，梳理代理与 cookies 的传递机制。所有结论均基于真实代码实现，文件路径统一为从仓库根目录开始的相对路径，行号经过逐行复核。

---

## 二、代理传递机制

### 2.1 代理来源与优先级

| 优先级 | 来源级别 | 配置位置 | 代码位置 |
|--------|----------|----------|----------|
| 1（最高） | 单条监控 | `watch.get('proxy')` | `changedetectionio/store/__init__.py:871-872` |
| 2 | 全局设置 | `datastore.data['settings']['requests']['proxy']` | `changedetectionio/model/App.py:31` |
| 3 | 环境变量 | `HTTP_PROXY`、`HTTPS_PROXY` | `changedetectionio/content_fetchers/base.py:61-62` |
| 4（最低） | 默认回退 | 代理列表第一项 | `changedetectionio/store/__init__.py:883-884` |

**全局代理配置结构**（`changedetectionio/model/App.py:27-31`）：

```python
'requests': {
    'extra_proxies': [],     # UI 配置的额外代理列表
    'extra_browsers': [],    # UI 配置的额外浏览器端点
    'jitter_seconds': 0,
    'proxy': None,           # 首选代理连接 ID（第31行）
    ...
}
```

**代理选择决策流程**（`changedetectionio/store/__init__.py:855-886`）：

```python
def get_preferred_proxy_for_watch(self, uuid):
    if self.proxy_list is None:
        return None

    watch = self.data['watching'].get(uuid)

    # 步骤1: 检查是否设置为"no-proxy"
    if strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')) and watch.get('proxy') == "no-proxy":
        return None

    # 步骤2: 单条监控的代理ID是否有效（优先级最高）
    if watch.get('proxy') and watch.get('proxy') in list(self.proxy_list.keys()):
        return watch.get('proxy')

    # 步骤3: 回退到全局系统代理
    else:
        system_proxy_id = self.data['settings']['requests'].get('proxy')
        if self.proxy_list.get(system_proxy_id):
            return system_proxy_id

    # 步骤4: 最终回退到代理列表第一项
    if system_proxy_id is None or not self.proxy_list.get(system_proxy_id):
        first_default = list(self.proxy_list)[0]
        return first_default

    return None
```

**环境变量加载**（`changedetectionio/content_fetchers/base.py:61-62`）：

```python
system_http_proxy = os.getenv('HTTP_PROXY')
system_https_proxy = os.getenv('HTTPS_PROXY')
```

### 2.2 代理注入时机

**统一注入入口**（`changedetectionio/processors/base.py:117-192`）：

```python
async def call_browser(self, preferred_proxy_id=None):
    # 1. 获取代理ID
    preferred_proxy_id = preferred_proxy_id if preferred_proxy_id else \
        self.datastore.get_preferred_proxy_for_watch(uuid=self.watch.get('uuid'))

    # 2. 解析代理URL
    proxy_url = None
    if preferred_proxy_id:
        # 自定义浏览器端点跳过代理
        if not prefer_fetch_backend.startswith('extra_browser_'):
            proxy_url = self.datastore.proxy_list.get(preferred_proxy_id).get('url')
        else:
            logger.debug("Skipping adding proxy data when custom Browser endpoint is specified. ")

    # 3. 创建fetcher实例并注入代理
    self.fetcher = fetcher_obj(proxy_override=proxy_url,
                               custom_browser_connection_url=custom_browser_connection_url,
                               screenshot_format=self.screenshot_format
                               )
```

### 2.3 各抓取器代理适配

#### Requests 抓取器

**代理注入方式**：通过 `proxies` 参数传递

**代码位置**：`changedetectionio/content_fetchers/requests.py:45-57`

```python
proxies = {}

if self.proxy_override:
    # 单条监控覆盖优先级最高
    proxies = {'http': self.proxy_override, 'https': self.proxy_override, 'ftp': self.proxy_override}
else:
    # 回退到环境变量
    if self.system_http_proxy:
        proxies['http'] = self.system_http_proxy
    if self.system_https_proxy:
        proxies['https'] = self.system_https_proxy

session = requests.Session()
r = session.request(method=request_method,
                    data=request_body.encode('utf-8') if type(request_body) is str else request_body,
                    url=url,
                    headers=request_headers,
                    timeout=timeout,
                    proxies=proxies,
                    ...)
```

**适配特点**：支持 HTTP/HTTPS/SOCKS5 协议，认证信息直接包含在 URL 中

#### Playwright 抓取器

**代理注入方式**：通过 `context.new_context(proxy=self.proxy)`

**代码位置**：`changedetectionio/content_fetchers/playwright.py:196-215, 285-293`

```python
# 从环境变量加载代理配置
proxy_args = {}
for k in self.playwright_proxy_settings_mappings:  # ['bypass', 'server', 'username', 'password']
    v = os.getenv('playwright_proxy_' + k, False)
    if v:
        proxy_args[k] = v.strip('"')

if proxy_args:
    self.proxy = proxy_args

# 单条监控覆盖（优先级高于环境变量）
if proxy_override:
    self.proxy = {'server': proxy_override}

# 解析URL中的认证信息
if self.proxy:
    parsed = urlparse(self.proxy.get('server'))
    if parsed.username:
        self.proxy['username'] = parsed.username
        self.proxy['password'] = parsed.password

# run 方法中使用
context = await browser.new_context(
    accept_downloads=False,
    bypass_csp=True,
    extra_http_headers=request_headers,
    ignore_https_errors=True,
    proxy=self.proxy,  # 第290行
    service_workers=os.getenv('PLAYWRIGHT_SERVICE_WORKERS', 'allow'),
    user_agent=manage_user_agent(headers=request_headers),
)
```

**适配特点**：支持 HTTP/HTTPS 代理认证，SOCKS5 认证不支持（官方限制）

#### Puppeteer 抓取器

**代理注入方式**：URL 参数追加 `--proxy-server` + `page.authenticate()`

**代码位置**：`changedetectionio/content_fetchers/puppeteer.py:208-225, 363`

```python
# 允许单条监控代理覆盖
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

# run 方法中通过 page.authenticate() 处理认证
if self.proxy and self.proxy.get('username'):
    await self.page.authenticate(self.proxy)  # 第363行
```

**适配特点**：代理地址通过 URL 参数传递，认证信息通过单独的 `authenticate()` 方法注入

### 2.4 代理作用范围

| 配置级别 | 作用范围 | 优先级 | 代码位置 |
|----------|----------|--------|----------|
| 单条监控 | 仅当前监控 | 最高 | `changedetectionio/store/__init__.py:871-872` |
| 全局设置 | 所有未单独配置的监控 | 中 | `changedetectionio/store/__init__.py:876-879` |
| 环境变量 | 全局所有监控 | 最低 | `changedetectionio/content_fetchers/base.py:61-62` |
| 自定义浏览器 | 跳过代理设置 | 特殊 | `changedetectionio/processors/base.py:179-183` |

---

## 三、Cookies 传递机制

### 3.1 Cookies Headers 覆盖顺序

**核心代码**（`changedetectionio/processors/base.py:198-208`）：

```python
# Tweak the base config with the per-watch ones
from changedetectionio.jinja2_custom import render as jinja_render
request_headers = CaseInsensitiveDict()

# 层级1: 默认 User-Agent（按抓取器类型）
ua = self.datastore.data['settings']['requests'].get('default_ua')
if ua and ua.get(prefer_fetch_backend):
    request_headers.update({'User-Agent': ua.get(prefer_fetch_backend)})

# 层级2: 单条监控 Headers
request_headers.update(self.watch.get('headers', {}))

# 层级3: 全局基础 Headers
request_headers.update(self.datastore.get_all_base_headers())

# 层级4: 文件配置 Headers（内部包含多层级）
request_headers.update(self.datastore.get_all_headers_in_textfile_for_watch(uuid=self.watch.get('uuid')))
```

**文件配置 Headers 内部层级**（`changedetectionio/store/__init__.py:900-934`）：

```python
def get_all_headers_in_textfile_for_watch(self, uuid):
    from ..model.App import parse_headers_from_text_file
    headers = {}

    # 层级4.1: 全局 headers.txt
    filepath = os.path.join(self.datastore_path, 'headers.txt')
    if os.path.isfile(filepath):
        headers.update(parse_headers_from_text_file(filepath))

    watch = self.data['watching'].get(uuid)
    if watch:
        # 层级4.2: 监控级 headers.txt
        filepath = os.path.join(watch.data_dir, 'headers.txt')
        if os.path.isfile(filepath):
            headers.update(parse_headers_from_text_file(filepath))

        # 层级4.3: 标签级 headers-tagname.txt
        tags = self.get_all_tags_for_watch(uuid=uuid)
        for tag_uuid, tag in tags.items():
            fname = "headers-" + re.sub(r'[\W_]', '', tag.get('title')).lower().strip() + ".txt"
            filepath = os.path.join(self.datastore_path, fname)
            if os.path.isfile(filepath):
                headers.update(parse_headers_from_text_file(filepath))

    return headers
```

**完整覆盖顺序表**（后加载的会覆盖先加载的同名 header）：

| 优先级 | 层级 | 来源 | 代码位置 |
|--------|------|------|----------|
| 1（最低） | 层级1 | 默认 User-Agent | `changedetectionio/processors/base.py:202-204` |
| 2 | 层级2 | 单条监控 Headers (`watch.get('headers')`) | `changedetectionio/processors/base.py:206` |
| 3 | 层级3 | 全局基础 Headers (`get_all_base_headers()`) | `changedetectionio/processors/base.py:207` |
| 4 | 层级4.1 | 文件 - 全局 (`/datastore/headers.txt`) | `changedetectionio/store/__init__.py:905-910` |
| 5 | 层级4.2 | 文件 - 监控 (`/datastore/{uuid}/headers.txt`) | `changedetectionio/store/__init__.py:916-921` |
| 6（最高） | 层级4.3 | 文件 - 标签 (`/datastore/headers-tagname.txt`) | `changedetectionio/store/__init__.py:924-932` |

### 3.2 各抓取器 Cookies 处理

#### Requests 抓取器

**处理方式**：直接通过 headers 传递，无自动管理

**代码位置**：`changedetectionio/content_fetchers/requests.py:96`

```python
r = session.request(method=request_method,
                    data=request_body.encode('utf-8') if type(request_body) is str else request_body,
                    url=url,
                    headers=request_headers,
                    timeout=timeout,
                    proxies=proxies,
                    ...)
```

**特点**：每次请求使用独立 Session，Cookies 完全隔离

#### Playwright 抓取器

**处理方式**：通过 `extra_http_headers` 传递初始 headers，浏览器上下文自动管理后续 cookies

**代码位置**：`changedetectionio/content_fetchers/playwright.py:288`

```python
context = await browser.new_context(
    accept_downloads=False,
    bypass_csp=True,
    extra_http_headers=request_headers,
    ignore_https_errors=True,
    proxy=self.proxy,
    service_workers=os.getenv('PLAYWRIGHT_SERVICE_WORKERS', 'allow'),
    user_agent=manage_user_agent(headers=request_headers),
)
```

**特点**：每条监控使用独立 context，Cookies 完全隔离

#### Puppeteer 抓取器

**处理方式**：通过 `setExtraHTTPHeaders` 传递初始 headers，浏览器上下文自动管理后续 cookies

**代码位置**：`changedetectionio/content_fetchers/puppeteer.py:352`

```python
await self.page.setExtraHTTPHeaders(request_headers)
```

**特点**：每条监控使用独立 page，Cookies 完全隔离

---

## 四、抓取后清理动作

### 4.1 Worker 层面清理

**代码位置**：`changedetectionio/worker.py:605-640`

```python
finally:
    # 步骤1: 调用 quit() 清理浏览器资源
    try:
        if update_handler and hasattr(update_handler, 'fetcher') and update_handler.fetcher:
            await update_handler.fetcher.quit(watch=watch)
    except Exception as e:
        logger.error(f"Exception while cleaning/quit after calling browser: {e}")

    # 步骤2: 清理内存引用
    if update_handler:
        if hasattr(update_handler, 'fetcher') and update_handler.fetcher:
            update_handler.fetcher.clear_content()
        if hasattr(update_handler, 'content_processor'):
            update_handler.content_processor = None
        del update_handler
        update_handler = None

    # 步骤3: 强制垃圾回收
    import gc
    gc.collect()
```

### 4.2 各抓取器 quit() 实现

#### Requests 抓取器（修正版）

**quit() 真实行为**：删除旧截图文件（当从其他抓取器切换到 requests 时清理遗留截图）

**代码位置**：`changedetectionio/content_fetchers/requests.py:245-255`

```python
async def quit(self, watch=None):
    # In case they switched to `requests` fetcher from something else
    # Then the screenshot could be old, in any case, it's not used here.
    # REMOVE_REQUESTS_OLD_SCREENSHOTS - Mainly used for testing
    if strtobool(os.getenv("REMOVE_REQUESTS_OLD_SCREENSHOTS", 'true')):
        screenshot = watch.get_screenshot()
        if screenshot:
            try:
                os.unlink(screenshot)
            except Exception as e:
                logger.warning(f"Failed to unlink screenshot: {screenshot} - {e}")
```

**关键说明**：Requests 抓取器不需要关闭浏览器连接（无浏览器资源），`quit()` 方法主要用于清理从其他抓取器（如 Playwright/Puppeteer）切换过来时遗留的截图文件。

#### Playwright 抓取器

**代码位置**：`changedetectionio/content_fetchers/playwright.py:420-454`

```python
finally:
    # Clean up resources properly with timeouts to prevent hanging
    try:
        if hasattr(self, 'page') and self.page:
            await self.page.request_gc()
            await asyncio.wait_for(self.page.close(), timeout=5.0)
    except asyncio.TimeoutError:
        logger.warning(f"Timed out closing page for {url} (5s)")

    try:
        if context:
            await asyncio.wait_for(context.close(), timeout=5.0)
    except asyncio.TimeoutError:
        logger.warning(f"Timed out closing context for {url} (5s)")

    try:
        if browser:
            await asyncio.wait_for(browser.close(), timeout=5.0)
    except asyncio.TimeoutError:
        logger.warning(f"Timed out closing browser connection for {url} (5s)")
```

#### Puppeteer 抓取器

**代码位置**：`changedetectionio/content_fetchers/puppeteer.py:227-257`

```python
async def quit(self, watch=None):
    watch_uuid = watch.get('uuid') if watch else 'unknown'

    # Close page
    try:
        if hasattr(self, 'page') and self.page:
            await asyncio.wait_for(self.page.close(), timeout=5.0)
            logger.debug(f"[{watch_uuid}] Page closed successfully")
    except asyncio.TimeoutError:
        logger.warning(f"[{watch_uuid}] Timed out closing page (5s)")
```

### 4.3 clear_content() 方法

**代码位置**：`changedetectionio/content_fetchers/base.py:105-116`

```python
def clear_content(self):
    """
    Explicitly clear all content from memory to free up heap space.
    Call this after content has been saved to disk.
    """
    self.content = None
    if hasattr(self, 'raw_content'):
        self.raw_content = None
    self.screenshot = None
    self.xpath_data = None
    # Keep headers and status_code as they're small
```

---

## 五、私密性保障机制

### 5.1 代理认证信息保护

| 保护措施 | 代码依据 |
|----------|----------|
| 仅在 fetcher 内部解析认证信息 | `changedetectionio/content_fetchers/playwright.py:212-215` |
| 认证信息通过专用字段传递 | `changedetectionio/content_fetchers/playwright.py:212-215` |
| 日志仅打印代理 URL（不含认证信息） | `changedetectionio/processors/base.py:181` |

### 5.2 Cookies 隔离

| 隔离措施 | 代码依据 |
|----------|----------|
| 每条监控使用独立浏览器上下文 | `changedetectionio/content_fetchers/playwright.py:285` |
| Requests 使用独立 Session | `changedetectionio/content_fetchers/requests.py:59` |
| 抓取完成后立即清理上下文 | `changedetectionio/worker.py:616` |

---

## 六、关键结论总结

### 6.1 代理传递关键结论

| 维度 | 结论 | 代码位置 |
|------|------|----------|
| 全局代理配置 | `settings['requests']['proxy']` | `changedetectionio/model/App.py:31` |
| 优先级 | 单条监控 > 全局设置 > 环境变量 > 默认列表第一项 | `changedetectionio/store/__init__.py:855-886` |
| 注入入口 | `call_browser()` 统一处理 | `changedetectionio/processors/base.py:117` |
| 自定义浏览器 | 跳过代理设置 | `changedetectionio/processors/base.py:179-183` |
| Requests 协议 | 支持 HTTP/HTTPS/SOCKS5 | `changedetectionio/content_fetchers/requests.py:45-57` |
| Playwright 协议 | 支持 HTTP/HTTPS，不支持 SOCKS5 认证 | `changedetectionio/content_fetchers/playwright.py:280` |
| Puppeteer 协议 | 通过 URL 参数传递，认证单独处理 | `changedetectionio/content_fetchers/puppeteer.py:208-225` |

### 6.2 Cookies 传递关键结论

| 维度 | 结论 | 代码位置 |
|------|------|----------|
| 覆盖顺序 | 标签文件(6) > 监控文件(5) > 全局文件(4) > 全局基础(3) > 监控级(2) > 默认(1) | `changedetectionio/processors/base.py:198-208` |
| 文件内部顺序 | 全局 headers.txt < 监控级 headers.txt < 标签级 headers-tagname.txt | `changedetectionio/store/__init__.py:900-934` |
| Playwright | `extra_http_headers` 参数 | `changedetectionio/content_fetchers/playwright.py:288` |
| Puppeteer | `setExtraHTTPHeaders` 方法 | `changedetectionio/content_fetchers/puppeteer.py:352` |
| 隔离性 | 完全隔离，无跨监控污染 | `changedetectionio/content_fetchers/playwright.py:285` |

### 6.3 清理机制关键结论

| 维度 | 结论 | 代码位置 |
|------|------|----------|
| 清理入口 | Worker finally 块 | `changedetectionio/worker.py:605` |
| Requests quit() | 删除旧截图文件 | `changedetectionio/content_fetchers/requests.py:245-255` |
| Playwright 关闭 | page → context → browser | `changedetectionio/content_fetchers/playwright.py:425,436,447` |
| Puppeteer 关闭 | page.close() | `changedetectionio/content_fetchers/puppeteer.py:233` |
| 内存释放 | `clear_content()` + `gc.collect()` | `changedetectionio/content_fetchers/base.py:105, worker.py:632` |

---

## 七、流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Worker 发起抓取                              │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│         changedetectionio/processors/base.py::call_browser()         │
├──────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ 1. 获取代理ID                                                  │    │
│  │    代码: changedetectionio/store/__init__.py:855-886        │    │
│  │    优先级: watch.proxy > global.proxy > default              │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                             │                                       │
│  ┌──────────────────────────▼───────────────────────────────────┐    │
│  │ 2. 构建请求头                                                  │    │
│  │    代码: changedetectionio/processors/base.py:198-208      │    │
│  │    优先级: 标签文件 > 监控文件 > 全局文件 > 全局 > 监控 > 默认│    │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                             │                                       │
│  ┌──────────────────────────▼───────────────────────────────────┐    │
│  │ 3. 创建 fetcher 实例并注入代理                                │    │
│  │    代码: changedetectionio/processors/base.py:189-192       │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
┌───────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Requests    │    │   Playwright    │    │   Puppeteer     │
├───────────────┤    ├─────────────────┤    ├─────────────────┤
│ proxies={}    │    │ proxy={server,  │    │ URL参数追加     │
│ 第45-57行     │    │  username,      │    │ --proxy-server  │
│               │    │  password}      │    │ 第208-225行     │
│               │    │ 第196-215行      │    │                 │
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
              │  changedetectionio/worker.py:605   │
              │  fetcher.quit() → clear_content()  │
              │  gc.collect()                       │
              └─────────────────────────────────────┘
```

---

## 八、代码位置速查表

| 功能模块 | 代码位置 |
|----------|----------|
| 全局代理配置 | `changedetectionio/model/App.py:31` |
| 代理选择逻辑 | `changedetectionio/store/__init__.py:855-886` |
| 环境变量加载 | `changedetectionio/content_fetchers/base.py:61-62` |
| 代理注入入口 | `changedetectionio/processors/base.py:117` |
| 跳过代理逻辑 | `changedetectionio/processors/base.py:179-183` |
| Headers 合并顺序 | `changedetectionio/processors/base.py:198-208` |
| 文件配置 Headers | `changedetectionio/store/__init__.py:900-934` |
| Requests 代理处理 | `changedetectionio/content_fetchers/requests.py:45-57` |
| Requests quit() | `changedetectionio/content_fetchers/requests.py:245-255` |
| Playwright 代理处理 | `changedetectionio/content_fetchers/playwright.py:196-215` |
| Playwright new_context | `changedetectionio/content_fetchers/playwright.py:285-293` |
| Playwright 清理 | `changedetectionio/content_fetchers/playwright.py:420-454` |
| Puppeteer 代理处理 | `changedetectionio/content_fetchers/puppeteer.py:208-225` |
| Puppeteer authenticate | `changedetectionio/content_fetchers/puppeteer.py:363` |
| Puppeteer quit() | `changedetectionio/content_fetchers/puppeteer.py:227` |
| clear_content() | `changedetectionio/content_fetchers/base.py:105-116` |
| Worker 清理流程 | `changedetectionio/worker.py:605-640` |

---

## 九、事实校准说明

### 9.1 Requests quit() 修正

**原错误结论**：Requests 继承自 base.py 的抽象 quit() 方法，无实际清理操作

**修正后结论**：Requests 有独立的 quit() 实现，用于删除从其他抓取器切换时遗留的旧截图文件

**代码证据**：`changedetectionio/content_fetchers/requests.py:245-255`

### 9.2 其他校准内容

1. **全局代理配置位置**：从 `App.py:32` 校准为 `App.py:31`
2. **Playwright 代理解析范围**：从 `183-215` 校准为 `196-215`（实际处理代理的代码段）
3. **Puppeteer 代理代码范围**：从 `209-225` 校准为 `208-225`
4. **Worker 清理代码范围**：从 `605-680` 校准为 `605-640`（核心清理逻辑）

所有行号均已逐行复核，确保可直接定位到对应代码。