# 代理选择策略完整说明

## 概述

本文档详细说明了 changedetection.io 项目中为每次抓取选择代理的整套策略，包括全局代理池配置、watch 级别覆盖、no-proxy 判定顺序、请求库与浏览器 fetcher 之间的代理传递，以及失败回退机制。

---

## 1. 全局代理池配置

### 1.1 代理池的来源与初始化

代理池的配置位于 `store/__init__.py:825-853`，由以下几个来源组成：

#### 1.1.1 配置文件来源 (proxies.json)
- 从 `datastore_path/proxies.json` 文件加载
- 支持 orjson (优先) 和标准 json 格式

#### 1.1.2 UI 配置来源 (extra_proxies)
- 从 `settings['requests']['extra_proxies']` 中读取
- 每个代理包含 `proxy_name` 和 `proxy_url` 字段
- 键名格式：`ui-{index}{proxy_name}`

#### 1.1.3 系统环境变量来源
- `HTTP_PROXY` / `HTTPS_PROXY` 环境变量（在 `content_fetchers/base.py:61-62` 中读取，主要用于请求库）

#### 1.1.4 No-Proxy 选项
- 当 `ENABLE_NO_PROXY_OPTION` 环境变量为 `True` 时，自动添加 `no-proxy` 选项
- 该选项的 URL 为空字符串，表示不使用代理

### 1.2 代理池数据结构

```python
proxy_list = {
    "proxy-key": {
        "label": "显示名称",
        "url": "代理地址"
    },
    "no-proxy": {
        "label": "No proxy",
        "url": ""
    }
}
```

### 1.3 全局默认代理配置

在 `model/App.py:31` 中定义了默认配置：
- `settings['requests']['proxy']`: 系统默认代理的键
- `settings['requests']['extra_proxies']: UI 配置的额外代理列表

---

## 2. Watch 级别代理覆盖机制

### 2.1 Watch 级别的代理配置

每个 Watch 对象可以独立配置自己的代理设置，存储在 `watch['proxy']` 字段中。

在 `blueprint/ui/edit.py:84-89` 中：
- 当 Watch 的 `proxy` 字段为 `None` 或无效时，使用系统默认代理
- 当 Watch 的 `proxy` 字段为有效代理键时，使用该代理

### 2.2 代理选择的优先级

代理选择的核心逻辑位于 `store/__init__.py:855-886` 的 `get_preferred_proxy_for_watch()` 方法：

```python
def get_preferred_proxy_for_watch(self, uuid):
    # 1. 如果代理池为空，返回 None
    if self.proxy_list is None:
        return None
    
    # 2. 如果 Watch 选择了 no-proxy，返回 None（不使用代理
    if watch.get('proxy') == "no-proxy":
        return None
    
    # 3. 如果 Watch 配置了有效代理键，返回该键
    if watch.get('proxy') in list(self.proxy_list.keys()):
        return watch.get('proxy')
    
    # 4. 否则使用系统默认代理
    system_proxy_id = self.data['settings']['requests'].get('proxy')
    if self.proxy_list.get(system_proxy_id):
        return system_proxy_id
    
    # 5. 最后返回代理池中的第一个代理
    first_default = list(self.proxy_list)[0]
    return first_default
```

### 2.3 代理选择优先级总结

```
Watch 级别代理配置 (watch['proxy'])
    ↓
系统默认代理 (settings['requests']['proxy'])
    ↓
代理池第一个代理
```

---

## 3. No-Proxy 判定顺序

### 3.1 No-Proxy 的判定逻辑

No-Proxy 的判定发生在 `get_preferred_proxy_for_watch()` 方法中 `store/__init__.py:868-869`：

```python
if strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')) and watch.get('proxy') == "no-proxy":
    return None
```

### 3.2 判定顺序

1. **检查 `ENABLE_NO_PROXY_OPTION` 环境变量（默认为 `True`
2. **检查 Watch 的 `proxy` 字段是否等于 `"no-proxy"`
3. **如果两者都满足，返回 `None`，表示不使用代理

### 3.3 No-Proxy 的效果

当返回 `None` 时：
- 在 `processors/base.py:176-183` 中，`proxy_url` 为 `None`
- 不向 fetcher 传递 `proxy_override=None`
- 各 fetcher 根据自己的逻辑决定如何处理 `None` 代理

---

## 4. 抓取层代理传递机制

### 4.1 代理选择的入口点

代理选择的入口点位于 `processors/base.py:117-192` 的 `call_browser()` 方法中：

```python
async def call_browser(self, preferred_proxy_id=None):
    # 1. 获取首选代理 ID
    preferred_proxy_id = preferred_proxy_id if preferred_proxy_id else self.datastore.get_preferred_proxy_for_watch(uuid=self.watch.get('uuid'))
    
    # 2. 转换为代理 URL
    proxy_url = None
    if preferred_proxy_id:
        if not prefer_fetch_backend.startswith('extra_browser_'):
            proxy_url = self.datastore.proxy_list.get(preferred_proxy_id).get('url')
    
    # 3. 传递给 fetcher
    self.fetcher = fetcher_obj(
        proxy_override=proxy_url,
        custom_browser_connection_url=custom_browser_connection_url,
        ...
    )
```

### 4.2 Requests Fetcher 的代理传递

#### 4.2.1 Requests Fetcher (`content_fetchers/requests.py:19-57`

```python
def __init__(self, proxy_override=None, **kwargs):
    self.proxy_override = proxy_override

def _run_sync(self, ...):
    proxies = {}
    
    # 如果有代理覆盖
    if self.proxy_override:
        proxies = {
            'http': self.proxy_override,
            'https': self.proxy_override,
            'ftp': self.proxy_override
        }
    else:
        # 否则使用系统环境变量
        if self.system_http_proxy:
            proxies['http'] = self.system_http_proxy
        if self.system_https_proxy:
            proxies['https'] = self.system_https_proxy
    
    # 传递给 requests.Session.request()
```

#### 4.2.2 代理优先级

```
proxy_override (Watch 级别)
    ↓
HTTP_PROXY / HTTPS_PROXY (系统环境变量)
    ↓
不使用代理
```

### 4.3 Playwright Fetcher 的代理传递

#### 4.3.1 Playwright Fetcher (`content_fetchers/playwright.py:183-216`

```python
def __init__(self, proxy_override=None, **kwargs):
    # 1. 从环境变量读取代理配置
    proxy_args = {}
    for k in ['bypass', 'server', 'username', 'password']:
        v = os.getenv('playwright_proxy_' + k, False)
        if v:
            proxy_args[k] = v.strip('"')
    
    if proxy_args:
        self.proxy = proxy_args
    
    # 2. Watch 级别代理覆盖
    if proxy_override:
        self.proxy = {'server': proxy_override}
    
    # 3. 解析代理 URL 中的用户名和密码
    if self.proxy:
        parsed = urlparse(self.proxy.get('server'))
        if parsed.username:
            self.proxy['username'] = parsed.username
            self.proxy['password'] = parsed.password

async def run(self, ...):
    # 传递给 browser.new_context()
    context = await browser.new_context(
        proxy=self.proxy,
        ...
    )
```

#### 4.3.2 代理优先级

```
proxy_override (Watch 级别)
    ↓
playwright_proxy_* 环境变量
    ↓
不使用代理
```

### 4.4 Puppeteer Fetcher 的代理传递

#### 4.4.1 Puppeteer Fetcher (`content_fetchers/puppeteer.py:197-225`

```python
def __init__(self, proxy_override=None, **kwargs):
    if proxy_override:
        parsed = urlparse(proxy_override)
        if parsed:
            self.proxy = {
                'username': parsed.username,
                'password': parsed.password
            }
            # 代理服务器通过 URL 参数传递给浏览器
            proxy_url = parsed.scheme + "://" + parsed.hostname + ...
            self.browser_connection_url += f"&--proxy-server={proxy_url}
```

### 4.5 Selenium WebDriver Fetcher 的代理传递

#### 4.5.1 Selenium Fetcher (`content_fetchers/webdriver_selenium.py:31-62`

```python
def __init__(self, proxy_override=None, **kwargs):
    proxy_sources = [
        self.system_http_proxy,
        self.system_https_proxy,
        os.getenv('webdriver_proxySocks'),
        os.getenv('webdriver_socksProxy'),
        ...,
        proxy_override,  # 最后一个覆盖
    ]
    
    for k in filter(None, proxy_sources):
        if k:
            self.proxy_url = k.strip()

def run(self, ...):
    if self.proxy_url:
        options.add_argument(f'--proxy-server={self.proxy_url}')
```

#### 4.5.2 代理优先级

```
proxy_override (Watch 级别)
    ↓
webdriver_* 环境变量
    ↓
HTTP_PROXY / HTTPS_PROXY (系统环境变量)
    ↓
不使用代理
```

---

## 5. 失败时的回退机制

### 5.1 代理失败处理

#### 5.1.1 Requests Fetcher 的重试机制 (`content_fetchers/requests.py:61-80`

```python
max_retries = int(os.getenv("REQUESTS_RETRY_MAX_COUNT", "6"))
retry_strategy = Retry(
    total=max_retries,
    connect=max_retries,
    read=max_retries,
    status=0,  # 不重试 HTTP 状态码
    backoff_factor=0.5,
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],
    raise_on_status=False
)
```

- **重试触发条件：
  - 连接超时
  - 读取超时
  - 连接重置
  - **不重试：HTTP 状态码

#### 5.1.2 代理连接错误处理

在 `content_fetchers/requests.py:127-131`：

```python
except Exception as e:
    msg = str(e)
    if proxies and 'SOCKSHTTPSConnectionPool' in msg:
        msg = f"Proxy connection failed? {msg}"
    raise Exception(msg) from e
```

### 5.2 无代理回退

**重要：系统不支持代理失败自动切换到其他代理。**

- 当代理失败时：
1. 请求会根据重试指定次数（默认 6 次）
2. 重试失败后，异常会被抛出
3. 没有自动切换到其他代理或直连的机制
4. Watch 会记录错误信息到 `watch['last_error']`

### 5.3 Worker 层的错误处理

在 `worker.py:187-409` 中：

- 捕获各种异常并记录到 Watch 的 `last_error` 字段：

```python
except content_fetchers_exceptions.Non200ErrorCodeReceived as e:
    if e.status_code == 407:
        err_text = "Error - 407 (Proxy authentication required) received, did you need a username and password for the proxy?"
    datastore.update_watch(uuid=uuid, update_obj={'last_error': err_text})
```

---

## 6. 代理选择完整流程图

```
抓取请求
    ↓
worker.py 处理队列
    ↓
processors/base.py: call_browser()
    ↓
get_preferred_proxy_for_watch()
    ├─ 检查代理池是否为空 → 返回 None
    ├─ 检查是否为 no-proxy → 返回 None
    ├─ 检查 Watch 级别代理配置 → 有效则返回
    ├─ 检查系统默认代理 → 有效则返回
    └─ 返回代理池第一个代理
    ↓
转换为 proxy_url
    ↓
创建 fetcher 实例 (proxy_override=proxy_url)
    ↓
fetcher.run()
    ├─ Requests: proxies 参数
    ├─ Playwright: context.proxy 参数
    ├─ Puppeteer: --proxy-server URL 参数
    └─ Selenium: --proxy-server 启动参数
    ↓
执行请求
    ↓
成功 / 失败
    ↓
重试 (Requests: 最多 6 次重试)
    ↓
成功 / 抛出异常
    ↓
记录到 Watch.last_error
```

---

## 7. 关键代码位置汇总

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 代理池初始化 | `store/__init__.py | 825-853 |
| 代理选择逻辑 | `store/__init__.py | 855-886 |
| 代理传递入口 | `processors/base.py | 117-192 |
| Requests 代理 | `content_fetchers/requests.py | 19-57 |
| Playwright 代理 | `content_fetchers/playwright.py | 183-216 |
| Puppeteer 代理 | `content_fetchers/puppeteer.py | 197-225 |
| Selenium 代理 | `content_fetchers/webdriver_selenium.py | 31-62 |
| Watch 代理配置 | `blueprint/ui/edit.py | 84-89, 202-203 |
| 全局代理配置 | `model/App.py | 28-39 |
| 重试机制 | `content_fetchers/requests.py | 61-80 |
| 错误处理 | `worker.py | 187-409 |

---

## 8. 注意事项

1. **No-Proxy 选项需要 `ENABLE_NO_PROXY_OPTION=True`（默认启用）
2. **自定义浏览器端点（`extra_browser_*）不使用代理
3. **代理失败不会自动切换到其他代理
4. **Requests 库的重试只重试网络层错误，不重试 HTTP 状态码
5. **不同 fetcher 的代理配置方式不同，需根据 fetcher 类型进行适配
6. **Watch 级别的代理优先级高于系统默认代理
7. **系统默认代理优先级高于环境变量代理
8. **Selenium 的代理优先级顺序为：proxy_override → webdriver_* 环境变量 → HTTP_PROXY/HTTPS_PROXY
