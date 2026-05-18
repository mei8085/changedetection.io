# 代理选择策略完整说明

## 概述

本文档详细说明了 changedetection.io 项目中为每次抓取选择代理的整套策略，包括全局代理池配置、watch 级别覆盖、no-proxy 判定顺序、请求库与浏览器 fetcher 之间的代理传递，以及失败回退机制。

---

## 1. 全局代理池配置

### 1.1 代理池的实际来源

全局代理池（`proxy_list`）仅由以下两个来源组成，定义于 `store/__init__.py:825-853`：

#### 1.1.1 本地配置文件来源 (proxies.json)
- 从 `datastore_path/proxies.json` 文件加载
- 支持 orjson (优先) 和标准 json 格式
- 这是主要的代理配置来源

#### 1.1.2 UI 配置来源 (extra_proxies)
- 从 `settings['requests']['extra_proxies']` 中读取
- 每个代理包含 `proxy_name` 和 `proxy_url` 字段
- 键名格式：`ui-{index}{proxy_name}`
- 通过 Web 界面设置的额外代理

#### 1.1.3 No-Proxy 选项的注入时机
**No-proxy 不是代理池的来源，而是在代理池构建完成后自动注入的选项**：

在 `store/__init__.py:850-851` 中：
```python
if proxy_list and strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')):
    proxy_list["no-proxy"] = {'label': "No proxy", 'url': ''}
```

- 注入条件：代理池非空 + `ENABLE_NO_PROXY_OPTION=True`（默认启用）
- 注入时机：在加载完 proxies.json 和 extra_proxies 之后
- 该选项的 URL 为空字符串，表示不使用代理

> **重要**：系统环境变量 `HTTP_PROXY` / `HTTPS_PROXY` **不是** 全局代理池的成员。它们仅在 Requests fetcher 中作为 fallback 使用。

### 1.2 代理池数据结构

```python
proxy_list = {
    "proxy-key-1": {
        "label": "代理显示名称",
        "url": "http://user:pass@proxy-host:port"
    },
    "ui-0my-proxy": {
        "label": "my-proxy",
        "url": "socks5://proxy-host:1080"
    },
    "no-proxy": {
        "label": "No proxy",
        "url": ""
    }
}
```

### 1.3 全局默认代理配置

在 `model/App.py:28-39` 中定义了默认配置：
- `settings['requests']['proxy']`: 系统默认代理的键（从代理池中选择）
- `settings['requests']['extra_proxies']`: UI 配置的额外代理列表

---

## 2. Watch 级别代理覆盖机制

### 2.1 Watch 级别的代理配置

每个 Watch 对象可以独立配置自己的代理设置，存储在 `watch['proxy']` 字段中。

在 `blueprint/ui/edit.py:84-89` 中：
- 当 Watch 的 `proxy` 字段为 `None` 或无效时，使用系统默认代理
- 当 Watch 的 `proxy` 字段为有效代理键时，使用该代理
- 当 Watch 的 `proxy` 字段为 `"no-proxy"` 时，不使用任何代理

### 2.2 代理选择的核心逻辑

代理选择的核心逻辑位于 `store/__init__.py:855-886` 的 `get_preferred_proxy_for_watch()` 方法：

```python
def get_preferred_proxy_for_watch(self, uuid):
    # 1. 如果代理池为空，返回 None
    if self.proxy_list is None:
        return None
    
    watch = self.data['watching'].get(uuid)
    
    # 2. 如果 Watch 选择了 no-proxy，返回 None（不使用代理）
    if strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')) and watch.get('proxy') == "no-proxy":
        return None
    
    # 3. 如果 Watch 配置了有效代理键，返回该键
    if watch.get('proxy') and watch.get('proxy') in list(self.proxy_list.keys()):
        return watch.get('proxy')
    
    # 4. 否则使用系统默认代理
    system_proxy_id = self.data['settings']['requests'].get('proxy')
    if self.proxy_list.get(system_proxy_id):
        return system_proxy_id
    
    # 5. 最后返回代理池中的第一个代理
    first_default = list(self.proxy_list)[0]
    return first_default
```

### 2.3 代理键选择优先级（从高到低）

```
1. Watch 级别代理配置 (watch['proxy'])
   ├─ "no-proxy" + ENABLE_NO_PROXY_OPTION=True → 返回 None（不使用代理）
   └─ 有效代理键 → 返回该键
2. 系统默认代理 (settings['requests']['proxy'])
   └─ 有效则返回
3. 代理池第一个代理
   └─ 返回
```

---

## 3. No-Proxy 判定顺序与行为

### 3.1 No-Proxy 的判定逻辑

No-Proxy 的判定发生在 `get_preferred_proxy_for_watch()` 方法中 `store/__init__.py:868-869`：

```python
if strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')) and watch.get('proxy') == "no-proxy":
    return None
```

### 3.2 判定顺序（从先到后）

1. **检查 `ENABLE_NO_PROXY_OPTION` 环境变量**（默认为 `True`）
   - 如果设置为 `False`，no-proxy 选项不会注入代理池，也不会被判定
   
2. **检查 Watch 的 `proxy` 字段是否等于 `"no-proxy"`**
   - 只有显式设置为 `"no-proxy"` 才会触发

3. **如果两者都满足，返回 `None`**，表示不使用代理

### 3.3 No-Proxy 命中后的实际行为

**关键点**：当 no-proxy 命中时，`get_preferred_proxy_for_watch()` 返回 `None`。

在 `processors/base.py:176-185` 中：
```python
proxy_url = None
if preferred_proxy_id:  # preferred_proxy_id 是 None，条件不成立
    if not prefer_fetch_backend.startswith('extra_browser_'):
        proxy_url = self.datastore.proxy_list.get(preferred_proxy_id).get('url')
        ...

logger.debug(f"Using proxy '{proxy_url}' for {self.watch['uuid']}")  # proxy_url 是 None

# 传递给 fetcher
self.fetcher = fetcher_obj(
    proxy_override=proxy_url,  # proxy_override 是 None
    ...
)
```

**结论**：
- no-proxy 命中后，`preferred_proxy_id = None`
- **不会** 尝试按 "no-proxy" 这个 key 去 proxy_list 中取 URL
- 直接传递 `proxy_override=None` 给 fetcher
- 各 fetcher 根据自己的逻辑处理 `proxy_override=None`

### 3.4 ENABLE_NO_PROXY_OPTION=False 时的回落路径

当 `ENABLE_NO_PROXY_OPTION=False` 时：

1. **代理池构建阶段**：`"no-proxy"` 不会被注入代理池
2. **Watch 配置了 `"no-proxy"` 时**：
   - 第 868 行条件不成立（`ENABLE_NO_PROXY_OPTION=False`），不会提前 return None
   - 继续执行第 871 行：`if watch.get('proxy') and watch.get('proxy') in list(self.proxy_list.keys()):`
   - 由于 `"no-proxy"` 不在代理池中，条件不成立
   - **继续回落**到系统默认代理 → 代理池第一个代理

**回落路径流程图**：
```
ENABLE_NO_PROXY_OPTION=False, watch['proxy']="no-proxy"
    ↓
get_preferred_proxy_for_watch()
    ↓
第 868 行: 条件不成立（ENABLE_NO_PROXY_OPTION=False）
    ↓
第 871 行: "no-proxy" 不在 proxy_list.keys() 中，条件不成立
    ↓
第 876-879 行: 尝试使用系统默认代理
    ├─ 系统默认代理有效 → 返回系统默认代理
    └─ 系统默认代理无效 → 继续
    ↓
第 882-884 行: 返回代理池第一个代理
    ↓
使用代理池中的某个代理（不会直连）
```

> **重要**：当 `ENABLE_NO_PROXY_OPTION=False` 时，即使 Watch 配置了 `"no-proxy"`，也**不会**直连，而是会回落到代理池中的其他代理。

---

## 4. 完整代理选择传递链

### 4.1 代理选择入口点

代理选择的入口点位于 `processors/base.py:117-192` 的 `call_browser()` 方法中。这是代理选择传递链的起点。

#### 4.1.1 完整传递链流程图

```
抓取请求
    ↓
worker.py: async_update_worker()
    ↓
update_handler.call_browser()  [processors/base.py:117]
    ↓
┌─ Step 1: 获取首选代理键 ───────────────────────────────┐
│  preferred_proxy_id = datastore.get_preferred_proxy_for_watch()
│  (调用 store/__init__.py:855-886)
│  返回: 代理键 或 None (no-proxy 或无代理池)
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 2: 转换为代理 URL ───────────────────────────────┐
│  if preferred_proxy_id:  # 注意：None 会跳过整个 if 块
│      if not extra_browser_*:
│          proxy_url = proxy_list[preferred_proxy_id]['url']
│      else:
│          proxy_url = None
│  else:
│      proxy_url = None  # no-proxy 命中时走这里
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 3: 创建 Fetcher 实例 ────────────────────────────┐
│  fetcher = fetcher_obj(
│      proxy_override=proxy_url,  # 可能是 None 或代理 URL
│      custom_browser_connection_url=...
│  )
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 4: Fetcher 内部代理配置（根据类型不同）──────────┐
│  各 fetcher 独立处理 proxy_override，并有各自的回退规则
└─────────────────────────────────────────────────────────┘
    ↓
fetcher.run() 执行实际请求
```

### 4.2 各 Fetcher 的代理回退规则（独立处理）

**重要**：每个 fetcher 有自己独立的代理回退规则，不是统一的优先级。请分别查看各 fetcher 的说明。

---

## 5. Requests Fetcher 的代理规则

### 5.1 传递路径 `content_fetchers/requests.py:19-57`

```python
def __init__(self, proxy_override=None, **kwargs):
    self.proxy_override = proxy_override  # 保存传递过来的代理 URL

def _run_sync(self, ...):
    proxies = {}
    
    # 优先级 1: 使用传递过来的 proxy_override
    if self.proxy_override:
        proxies = {
            'http': self.proxy_override,
            'https': self.proxy_override,
            'ftp': self.proxy_override
        }
    # 优先级 2: 当 proxy_override 为 None 或空字符串时，回落到系统环境变量
    else:
        if self.system_http_proxy:    # os.getenv('HTTP_PROXY')
            proxies['http'] = self.system_http_proxy
        if self.system_https_proxy:   # os.getenv('HTTPS_PROXY')
            proxies['https'] = self.system_https_proxy
    
    # 传递给 requests.Session.request()
    r = session.request(..., proxies=proxies, ...)
```

### 5.2 Requests 代理回退规则（独立）

```
1. proxy_override (来自 call_browser)
   ├─ 非空字符串 → 使用该代理（proxies 字典包含该代理）
   └─ None 或空字符串 → 继续检查环境变量
2. 系统环境变量（仅当 proxy_override 为空时检查）
   ├─ HTTP_PROXY → 用于 HTTP 请求
   └─ HTTPS_PROXY → 用于 HTTPS 请求
3. 不使用代理（直连）
   └─ proxies 字典为空，requests 直连目标地址
```

> **关键说明**：只有当 `proxy_override` 为 `None` 或空字符串时，Requests 才会使用 `HTTP_PROXY` / `HTTPS_PROXY` 环境变量。
> 这是系统环境变量代理唯一生效的场景。

### 5.3 重试机制

在 `content_fetchers/requests.py:61-80` 中配置了重试策略：

```python
max_retries = int(os.getenv("REQUESTS_RETRY_MAX_COUNT", "6"))
retry_strategy = Retry(
    total=max_retries,
    connect=max_retries,    # 重试连接超时
    read=max_retries,       # 重试读取超时
    status=0,               # 不重试 HTTP 状态码
    backoff_factor=0.5,     # 退避因子：0.3s, 0.6s, 1.2s...
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],
    raise_on_status=False
)
```

- **重试触发条件**：连接超时、读取超时、连接重置等网络层错误
- **不重试**：HTTP 状态码（如 407、500 等）
- **代理失败**：不会自动切换到其他代理或直连，重试失败后抛出异常

---

## 6. Playwright Fetcher 的代理规则

### 6.1 传递路径 `content_fetchers/playwright.py:183-216`

```python
def __init__(self, proxy_override=None, **kwargs):
    # 1. 从环境变量读取 Playwright 专用代理配置
    proxy_args = {}
    for k in ['bypass', 'server', 'username', 'password']:
        v = os.getenv('playwright_proxy_' + k, False)
        if v:
            proxy_args[k] = v.strip('"')
    
    if proxy_args:
        self.proxy = proxy_args
    
    # 2. Watch 级别代理覆盖（优先级更高）
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
        proxy=self.proxy,  # None 表示不使用代理
        ...
    )
```

### 6.2 Playwright 代理回退规则（独立）

```
1. proxy_override (来自 call_browser)
   ├─ 非空字符串 → self.proxy = {'server': proxy_override}
   └─ None 或空字符串 → 继续检查 Playwright 环境变量
2. playwright_proxy_* 环境变量（仅当 proxy_override 为空时检查）
   ├─ playwright_proxy_server
   ├─ playwright_proxy_bypass
   ├─ playwright_proxy_username
   └─ playwright_proxy_password
3. 不使用代理（直连）
   └─ self.proxy = None，Playwright 直连目标地址
```

> **注意**：Playwright **不使用** `HTTP_PROXY` / `HTTPS_PROXY` 环境变量。

---

## 7. Puppeteer Fetcher 的代理规则

### 7.1 传递路径 `content_fetchers/puppeteer.py:197-225`

```python
def __init__(self, proxy_override=None, **kwargs):
    if proxy_override:
        parsed = urlparse(proxy_override)
        if parsed:
            # 提取用户名和密码用于认证
            self.proxy = {
                'username': parsed.username,
                'password': parsed.password
            }
            # 代理服务器通过 URL 参数传递给浏览器
            proxy_url = parsed.scheme + "://" + parsed.hostname + ...
            self.browser_connection_url += f"&--proxy-server={proxy_url}"

async def run(self, ...):
    self.browser = await pyppeteer_instance.connect(
        browserWSEndpoint=self.browser_connection_url,
        ...
    )
    # 如果有代理认证
    if self.proxy and self.proxy.get('username'):
        await self.page.authenticate(self.proxy)
```

### 7.2 Puppeteer 代理回退规则（独立）

```
1. proxy_override (来自 call_browser)
   ├─ 非空字符串 → 通过 --proxy-server 参数附加到 browser_connection_url
   └─ None 或空字符串 → 不添加 --proxy-server 参数
2. 不使用代理（直连）
   └─ 没有 --proxy-server 参数，Puppeteer 直连目标地址
```

> **注意**：Puppeteer **没有其他回退**，不检查任何环境变量代理。

---

## 8. Selenium WebDriver Fetcher 的代理规则

### 8.1 传递路径 `content_fetchers/webdriver_selenium.py:31-62`

```python
def __init__(self, proxy_override=None, **kwargs):
    proxy_sources = [
        self.system_http_proxy,          # HTTP_PROXY
        self.system_https_proxy,         # HTTPS_PROXY
        os.getenv('webdriver_proxySocks'),
        os.getenv('webdriver_socksProxy'),
        os.getenv('webdriver_proxyHttp'),
        os.getenv('webdriver_httpProxy'),
        os.getenv('webdriver_proxyHttps'),
        os.getenv('webdriver_httpsProxy'),
        os.getenv('webdriver_sslProxy'),
        proxy_override,                  # 最后一个，优先级最高
    ]
    
    # 遍历所有来源，最后一个非空值生效
    for k in filter(None, proxy_sources):
        if k:
            self.proxy_url = k.strip()

def run(self, ...):
    if self.proxy_url:
        options.add_argument(f'--proxy-server={self.proxy_url}')
```

### 8.2 Selenium 代理回退规则（独立）

Selenium 使用**遍历覆盖**机制：按顺序检查所有代理来源，**最后一个非空值**生效。

```
遍历顺序（先检查的会被后检查的覆盖）：
1. HTTP_PROXY 环境变量
2. HTTPS_PROXY 环境变量
3. webdriver_proxySocks 环境变量
4. webdriver_socksProxy 环境变量
5. webdriver_proxyHttp 环境变量
6. webdriver_httpProxy 环境变量
7. webdriver_proxyHttps 环境变量
8. webdriver_httpsProxy 环境变量
9. webdriver_sslProxy 环境变量
10. proxy_override (来自 call_browser) → 最后一个，优先级最高

最终结果：
├─ 最后一个非空值 → 作为 --proxy-server 参数
└─ 全部为空 → 不添加 --proxy-server 参数，直连
```

> **注意**：Selenium 的机制与其他 fetcher 不同。即使 `proxy_override` 为空，前面的环境变量也可能生效。
> 但 `proxy_override` 放在列表最后，所以只要它非空，就会覆盖所有环境变量。

---

## 9. 失败时的处理机制

### 9.1 代理连接错误处理

在 `content_fetchers/requests.py:127-131` 中：

```python
except Exception as e:
    msg = str(e)
    if proxies and 'SOCKSHTTPSConnectionPool' in msg:
        msg = f"Proxy connection failed? {msg}"
    raise Exception(msg) from e
```

### 9.2 无代理自动切换

**重要：系统不支持代理失败自动切换到其他代理或直连。**

当代理失败时：
1. Requests 会根据重试策略重试指定次数（默认 6 次）
2. 重试失败后，异常会被抛出
3. 没有自动切换到其他代理或直连的机制
4. Watch 会记录错误信息到 `watch['last_error']`

### 9.3 Worker 层的错误处理

在 `worker.py:187-409` 中捕获各种异常并记录：

```python
except content_fetchers_exceptions.Non200ErrorCodeReceived as e:
    if e.status_code == 407:
        err_text = "Error - 407 (Proxy authentication required) received, did you need a username and password for the proxy?"
    datastore.update_watch(uuid=uuid, update_obj={'last_error': err_text})
```

---

## 10. 完整决策流程总结

### 10.1 代理选择总流程

```
开始
    ↓
┌─ 代理池构建 ───────────────────────────────────────────┐
│ 1. 加载 proxies.json
│ 2. 合并 extra_proxies (UI 配置)
│ 3. 如 ENABLE_NO_PROXY_OPTION=True，注入 "no-proxy" 选项
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 选择代理键 (get_preferred_proxy_for_watch) ───────────┐
│ 1. 如果代理池为空 → 返回 None
│ 2. 如果 ENABLE_NO_PROXY_OPTION=True 且 watch['proxy'] == "no-proxy" → 返回 None
│ 3. 如果 watch['proxy'] 是有效代理键 → 返回该键
│ 4. 如果系统默认代理有效 → 返回系统默认代理键
│ 5. 返回代理池第一个代理键
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 转换为 proxy_url ─────────────────────────────────────┐
│ if preferred_proxy_id and not extra_browser_*:
│     proxy_url = proxy_list[preferred_proxy_id]['url']
│ else:
│     proxy_url = None （no-proxy 命中时走这里）
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 传递给 Fetcher (proxy_override=proxy_url) ────────────┐
│ 根据 Fetcher 类型，按各自的独立规则处理
└─────────────────────────────────────────────────────────┘
    ↓
结束
```

### 10.2 各 Fetcher 的代理决策对比表

| Fetcher 类型 | 优先级 1 (最高) | 优先级 2 | 优先级 3 | 优先级 4 (最低) |
|-------------|-----------------|----------|----------|-----------------|
| **Requests** | proxy_override | HTTP_PROXY / HTTPS_PROXY 环境变量 | 直连 | - |
| **Playwright** | proxy_override | playwright_proxy_* 环境变量 | 直连 | - |
| **Puppeteer** | proxy_override | 直连 | - | - |
| **Selenium** | proxy_override (最后遍历) | webdriver_* 环境变量 | HTTP_PROXY / HTTPS_PROXY 环境变量 | 直连 |

### 10.3 特殊场景说明

#### 场景 1：no-proxy 命中 + Requests Fetcher
```
proxy_override = None
    ↓
Requests 检查 HTTP_PROXY / HTTPS_PROXY 环境变量
    ├─ 环境变量存在 → 使用环境变量代理
    └─ 环境变量不存在 → 直连
```

#### 场景 2：no-proxy 命中 + Playwright Fetcher
```
proxy_override = None
    ↓
Playwright 检查 playwright_proxy_* 环境变量
    ├─ 环境变量存在 → 使用环境变量代理
    └─ 环境变量不存在 → 直连
```

#### 场景 3：no-proxy 命中 + Puppeteer Fetcher
```
proxy_override = None
    ↓
不添加 --proxy-server 参数
    ↓
直连
```

#### 场景 4：no-proxy 命中 + Selenium Fetcher
```
proxy_override = None
    ↓
Selenium 遍历前面的环境变量
    ├─ webdriver_* 环境变量存在 → 使用该代理
    ├─ HTTP_PROXY / HTTPS_PROXY 存在 → 使用该代理
    └─ 全部为空 → 直连
```

#### 场景 5：ENABLE_NO_PROXY_OPTION=False + watch['proxy']="no-proxy"
```
"no-proxy" 不在代理池中
    ↓
回落到系统默认代理 → 代理池第一个代理
    ↓
使用代理池中的某个代理（不会直连）
```

---

## 11. 关键代码位置汇总

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 代理池初始化 | `store/__init__.py` | 825-853 |
| no-proxy 注入 | `store/__init__.py` | 850-851 |
| 代理选择逻辑 | `store/__init__.py` | 855-886 |
| no-proxy 判定 | `store/__init__.py` | 868-869 |
| 代理传递入口 | `processors/base.py` | 117-192 |
| proxy_url 转换 | `processors/base.py` | 176-185 |
| Requests 代理配置 | `content_fetchers/requests.py` | 19-57 |
| Requests 重试机制 | `content_fetchers/requests.py` | 61-80 |
| Playwright 代理配置 | `content_fetchers/playwright.py` | 183-216 |
| Puppeteer 代理配置 | `content_fetchers/puppeteer.py` | 197-225 |
| Selenium 代理配置 | `content_fetchers/webdriver_selenium.py` | 31-62 |
| Watch 代理配置 UI | `blueprint/ui/edit.py` | 84-89, 202-203 |
| 全局代理配置 | `model/App.py` | 28-39 |
| Worker 错误处理 | `worker.py` | 187-409 |

---

## 12. 注意事项

1. **全局代理池仅包含**：`proxies.json` 配置 + UI 配置的 `extra_proxies`
2. **系统环境变量 `HTTP_PROXY` / `HTTPS_PROXY` 不是代理池成员**，仅在 Requests 和 Selenium fetcher 中作为 fallback
3. **No-proxy 命中后返回 `None`**，不会按 key 去 proxy_list 取 URL，直接传递 `proxy_override=None`
4. **当 `ENABLE_NO_PROXY_OPTION=False` 时**，Watch 配置 `"no-proxy"` 不会直连，会回落到代理池中的其他代理
5. **每个 fetcher 有独立的代理回退规则**，不要混成一套统一优先级
6. **自定义浏览器端点（`extra_browser_*`）不使用代理**
7. **代理失败不会自动切换**到其他代理或直连
8. **Requests 库的重试**：仅重试网络层错误，不重试 HTTP 状态码
9. **Selenium 使用遍历覆盖机制**：最后一个非空值生效，`proxy_override` 放在最后确保优先级最高
10. **no-proxy 命中后**，各 fetcher 仍可能根据自己的规则使用环境变量代理，不一定真正直连
