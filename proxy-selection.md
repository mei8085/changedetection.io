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

> **重要**：系统环境变量 `HTTP_PROXY` / `HTTPS_PROXY` **不是** 全局代理池的成员。它们仅在 Requests 和 Selenium fetcher 中作为 fallback 使用。

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
┌─ Step 2: 解析 prefer_fetch_backend ────────────────────┐
│  如果为空或 'system'，从设置中获取
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 3: 处理 extra_browser_* 类型 ────────────────────┐
│  详见第 9 章的完整分支分析
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 4: PDF 特殊处理 ─────────────────────────────────┐
│  if watch.is_pdf: prefer_fetch_backend = "html_requests"
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 5: 转换为代理 URL ───────────────────────────────┐
│  if preferred_proxy_id:
│      if not prefer_fetch_backend.startswith('extra_browser_'):
│          proxy_url = proxy_list[preferred_proxy_id]['url']
│      else:
│          proxy_url = None （跳过代理）
│  else:
│      proxy_url = None  # no-proxy 命中时走这里
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 6: 创建 Fetcher 实例 ────────────────────────────┐
│  fetcher = fetcher_obj(
│      proxy_override=proxy_url,
│      custom_browser_connection_url=custom_browser_connection_url
│  )
└─────────────────────────────────────────────────────────┘
    ↓
┌─ Step 7: Fetcher 内部代理配置（根据类型不同）──────────┐
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

## 9. Extra Browser (extra_browser_*) 场景的代理行为

### 9.1 场景说明

`extra_browser_*` 是一种特殊的 fetch backend 类型，用于指定自定义的浏览器连接端点（如远程 Selenium Grid、Browserless 等）。

配置位置：`settings['requests']['extra_browsers']`，每个配置包含：
- `browser_name`: 浏览器名称，用于标识
- `browser_connection_url`: 自定义浏览器连接地址

### 9.2 代码执行路径完整分支分析

代码位于 `processors/base.py:135-183`，执行顺序和分支如下：

```
开始
    ↓
┌─ 前置条件 ─────────────────────────────────────────────┐
│ 1. preferred_proxy_id 已获取（可能是代理键或 None）
│ 2. prefer_fetch_backend 已解析（从参数或设置中获取）
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 分支 A: prefer_fetch_backend 以 'extra_browser_' 开头 ─┐
│  是 → 进入 extra_browser_* 处理逻辑
│  否 → 跳到分支 D（普通流程）
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 分支 A1: 找到对应的连接配置 ────────────────────────────┐
│  if connection: （在 extra_browsers 列表中找到匹配项）
│  ├─ 是 → prefer_fetch_backend = 'html_webdriver'
│  │       custom_browser_connection_url = 连接地址
│  │       继续执行
│  └─ 否 → prefer_fetch_backend 保持 'extra_browser_*'
│          custom_browser_connection_url = None
│          继续执行
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 分支 B: 是否是 PDF ───────────────────────────────────┐
│  if watch.is_pdf:
│  ├─ 是 → prefer_fetch_backend = 'html_requests'
│  │       覆盖之前的改写
│  └─ 否 → 保持不变
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 分支 C: 转换为 proxy_url ─────────────────────────────┐
│  if preferred_proxy_id: （no-proxy 命中时为 None，跳过）
│  ├─ if not prefer_fetch_backend.startswith('extra_browser_'):
│  │   ├─ 是 → proxy_url = 代理 URL（注入代理）
│  │   └─ 否 → proxy_url = None（跳过代理）
│  └─ preferred_proxy_id 为 None 时 → proxy_url = None
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 分支 D: 创建 Fetcher 实例 ────────────────────────────┐
│  fetcher = fetcher_obj(
│      proxy_override=proxy_url,
│      custom_browser_connection_url=custom_browser_connection_url
│  )
└─────────────────────────────────────────────────────────┘
    ↓
结束
```

### 9.3 各分支的实际执行结果

以下是所有可能的组合及其结果：

#### 组合 1：extra_browser_* + 找到连接配置 + 非 PDF

| 步骤 | 变量值 |
|------|--------|
| 初始 `prefer_fetch_backend` | `extra_browser_my-grid` |
| 找到连接配置 | ✅ 是 |
| 改写后 `prefer_fetch_backend` | `html_webdriver` |
| `custom_browser_connection_url` | 远程浏览器地址 |
| 是否 PDF | ❌ 否 |
| 第 179 行检查 `startswith('extra_browser_')` | ❌ 否（已是 `html_webdriver`） |
| `proxy_url` (如果 `preferred_proxy_id` 非空) | ✅ 代理 URL |
| `proxy_override` | 代理 URL |

**结果**：
- `custom_browser_connection_url` 用于连接远程浏览器
- `proxy_override` 用于配置浏览器的 `--proxy-server` 参数
- 两者**同时生效**，远程浏览器通过代理访问目标网站

#### 组合 2：extra_browser_* + 找到连接配置 + 是 PDF

| 步骤 | 变量值 |
|------|--------|
| 初始 `prefer_fetch_backend` | `extra_browser_my-grid` |
| 找到连接配置 | ✅ 是 |
| 改写后 `prefer_fetch_backend` | `html_webdriver` |
| `custom_browser_connection_url` | 远程浏览器地址 |
| 是否 PDF | ✅ 是 |
| PDF 改写后 `prefer_fetch_backend` | `html_requests` |
| 第 179 行检查 `startswith('extra_browser_')` | ❌ 否（已是 `html_requests`） |
| `proxy_url` (如果 `preferred_proxy_id` 非空) | ✅ 代理 URL |
| `proxy_override` | 代理 URL |

**结果**：
- 虽然配置了 extra_browser_*，但 PDF 优先，使用 Requests fetcher
- `custom_browser_connection_url` 被传递但被 Requests fetcher 忽略
- `proxy_override` 传递给 Requests fetcher

#### 组合 3：extra_browser_* + 未找到连接配置 + 非 PDF

| 步骤 | 变量值 |
|------|--------|
| 初始 `prefer_fetch_backend` | `extra_browser_unknown` |
| 找到连接配置 | ❌ 否 |
| `prefer_fetch_backend` (保持不变) | `extra_browser_unknown` |
| `custom_browser_connection_url` | `None` |
| 是否 PDF | ❌ 否 |
| 第 179 行检查 `startswith('extra_browser_')` | ✅ 是（仍是 `extra_browser_unknown`） |
| `proxy_url` | ❌ `None`（跳过代理） |
| `proxy_override` | `None` |

**结果**：
- `prefer_fetch_backend` 仍是 `extra_browser_unknown`，不是有效的 fetcher 名称
- 第 162 行 `hasattr(content_fetchers, prefer_fetch_backend)` 检查失败
- 第 174 行回退到默认的 `html_requests` fetcher
- `proxy_override=None`，Requests fetcher 可能使用环境变量代理

#### 组合 4：非 extra_browser_*（普通流程）

| 步骤 | 变量值 |
|------|--------|
| 初始 `prefer_fetch_backend` | `html_requests` 或其他 |
| extra_browser_* 处理 | 跳过 |
| `custom_browser_connection_url` | `None` |
| 第 179 行检查 `startswith('extra_browser_')` | ❌ 否 |
| `proxy_url` (如果 `preferred_proxy_id` 非空) | ✅ 代理 URL |
| `proxy_override` | 代理 URL |

**结果**：
- 正常流程，代理被注入

### 9.4 代码意图 vs 实际行为

**代码意图**（第 178 行注释）：
> "Custom browser endpoints should NOT have a proxy added"
> （自定义浏览器端点不应该添加代理）

**实际行为分析**：
- **组合 1（找到连接配置）**：代理**会被注入**（与注释意图相反）
- **组合 3（未找到连接配置）**：代理**不会被注入**（符合注释意图）
- **组合 2（PDF）**：代理**会被注入**（PDF 优先级更高）

**结论**：代码意图与实际行为不一致。注释说自定义浏览器端点不应该添加代理，但在"找到连接配置"的正常场景下，由于 `prefer_fetch_backend` 被提前改写为 `html_webdriver`，代理检查条件命中，代理实际上会被传递。

### 9.5 Selenium Fetcher 中的处理

在 `content_fetchers/webdriver_selenium.py:61-69` 中：
```python
if self.custom_browser_connection_url:
    self.driver = webdriver.Remote(
        command_executor=self.custom_browser_connection_url,
        options=options  # options 中包含 --proxy-server 参数
    )
```

这意味着：
- `custom_browser_connection_url` 用于连接到远程浏览器
- `proxy_override` 用于配置浏览器的代理设置（`--proxy-server` 参数）
- 两者**同时生效**，不是互斥关系
- 远程浏览器会通过指定的代理去访问目标网站

---

## 10. 失败时的处理机制

### 10.1 代理连接错误处理

在 `content_fetchers/requests.py:127-131` 中：

```python
except Exception as e:
    msg = str(e)
    if proxies and 'SOCKSHTTPSConnectionPool' in msg:
        msg = f"Proxy connection failed? {msg}"
    raise Exception(msg) from e
```

### 10.2 无代理自动切换

**重要：系统不支持代理失败自动切换到其他代理或直连。**

当代理失败时：
1. Requests 会根据重试策略重试指定次数（默认 6 次）
2. 重试失败后，异常会被抛出
3. 没有自动切换到其他代理或直连的机制
4. Watch 会记录错误信息到 `watch['last_error']`

### 10.3 Worker 层的错误处理

在 `worker.py:187-409` 中捕获各种异常并记录：

```python
except content_fetchers_exceptions.Non200ErrorCodeReceived as e:
    if e.status_code == 407:
        err_text = "Error - 407 (Proxy authentication required) received, did you need a username and password for the proxy?"
    datastore.update_watch(uuid=uuid, update_obj={'last_error': err_text})
```

---

## 11. 完整决策流程总结

### 11.1 代理选择总流程

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
┌─ 处理 extra_browser_* 类型（详见第 9 章）─────────────┐
│ 分 4 种组合情况处理
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 转换为 proxy_url ─────────────────────────────────────┐
│ if preferred_proxy_id:
│     if not prefer_fetch_backend.startswith('extra_browser_'):
│         proxy_url = proxy_list[preferred_proxy_id]['url']
│     else:
│         proxy_url = None （跳过代理）
│ else:
│     proxy_url = None （no-proxy 命中时走这里）
└─────────────────────────────────────────────────────────┘
    ↓
┌─ 传递给 Fetcher ───────────────────────────────────────┐
│ proxy_override=proxy_url
│ custom_browser_connection_url=连接地址 (如果是 extra_browser_*)
│ 根据 Fetcher 类型，按各自的独立规则处理
└─────────────────────────────────────────────────────────┘
    ↓
结束
```

### 11.2 各 Fetcher 的代理决策对比表

| Fetcher 类型 | 优先级 1 (最高) | 优先级 2 | 优先级 3 | 优先级 4 (最低) |
|-------------|-----------------|----------|----------|-----------------|
| **Requests** | proxy_override | HTTP_PROXY / HTTPS_PROXY 环境变量 | 直连 | - |
| **Playwright** | proxy_override | playwright_proxy_* 环境变量 | 直连 | - |
| **Puppeteer** | proxy_override | 直连 | - | - |
| **Selenium** | proxy_override (最后遍历) | webdriver_* 环境变量 | HTTP_PROXY / HTTPS_PROXY 环境变量 | 直连 |

### 11.3 特殊场景说明

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

#### 场景 6：extra_browser_* + 找到连接配置 + 非 PDF
```
prefer_fetch_backend = "extra_browser_my-grid"
    ↓
找到连接配置 → 改写为 "html_webdriver"
    ↓
custom_browser_connection_url = 远程浏览器地址
    ↓
非 PDF → 保持 "html_webdriver"
    ↓
第 179 行检查："html_webdriver" 不以 "extra_browser_" 开头
    ↓
proxy_url = 代理 URL (如果 preferred_proxy_id 非空)
    ↓
传递给 Selenium:
  - custom_browser_connection_url 用于连接远程浏览器
  - proxy_override 用于配置浏览器的 --proxy-server 参数
    ↓
两者同时生效，远程浏览器通过代理访问目标网站
```

#### 场景 7：extra_browser_* + 未找到连接配置
```
prefer_fetch_backend = "extra_browser_unknown"
    ↓
未找到连接配置 → 保持 "extra_browser_unknown"
    ↓
custom_browser_connection_url = None
    ↓
第 179 行检查："extra_browser_unknown" 以 "extra_browser_" 开头
    ↓
proxy_url = None（跳过代理）
    ↓
prefer_fetch_backend 不是有效 fetcher 名称 → 回退到 html_requests
    ↓
proxy_override=None，Requests fetcher 可能使用环境变量代理
```

---

## 12. 关键代码位置汇总

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 代理池初始化 | `store/__init__.py` | 825-853 |
| no-proxy 注入 | `store/__init__.py` | 850-851 |
| 代理选择逻辑 | `store/__init__.py` | 855-886 |
| no-proxy 判定 | `store/__init__.py` | 868-869 |
| 代理传递入口 | `processors/base.py` | 117-192 |
| extra_browser_* 改写 | `processors/base.py` | 146-152 |
| PDF 改写 | `processors/base.py` | 157-158 |
| proxy_url 转换 | `processors/base.py` | 176-185 |
| Requests 代理配置 | `content_fetchers/requests.py` | 19-57 |
| Requests 重试机制 | `content_fetchers/requests.py` | 61-80 |
| Playwright 代理配置 | `content_fetchers/playwright.py` | 183-216 |
| Puppeteer 代理配置 | `content_fetchers/puppeteer.py` | 197-225 |
| Selenium 代理配置 | `content_fetchers/webdriver_selenium.py` | 31-62 |
| Selenium 远程连接 | `content_fetchers/webdriver_selenium.py` | 61-69 |
| Watch 代理配置 UI | `blueprint/ui/edit.py` | 84-89, 202-203 |
| 全局代理配置 | `model/App.py` | 28-39 |
| Worker 错误处理 | `worker.py` | 187-409 |

---

## 13. 注意事项

### 13.1 代理池与配置

1. **全局代理池仅包含**：`proxies.json` 配置 + UI 配置的 `extra_proxies`
2. **系统环境变量 `HTTP_PROXY` / `HTTPS_PROXY` 不是代理池成员**，仅在 Requests 和 Selenium fetcher 中作为 fallback
3. **No-proxy 选项**：在代理池构建完成后注入，需要 `ENABLE_NO_PROXY_OPTION=True`（默认启用）

### 13.2 No-Proxy 行为

4. **No-proxy 命中后返回 `None`**，不会按 key 去 proxy_list 取 URL，直接传递 `proxy_override=None`
5. **当 `ENABLE_NO_PROXY_OPTION=False` 时**，Watch 配置 `"no-proxy"` 不会直连，会回落到代理池中的其他代理
6. **no-proxy 命中后**，各 fetcher 仍可能根据自己的规则使用环境变量代理，不一定真正直连

### 13.3 各 Fetcher 独立规则

7. **每个 fetcher 有独立的代理回退规则**，不要混成一套统一优先级
8. **Selenium 使用遍历覆盖机制**：最后一个非空值生效，`proxy_override` 放在最后确保优先级最高
9. **Puppeteer 没有其他回退**，不检查任何环境变量代理
10. **Playwright 不使用** `HTTP_PROXY` / `HTTPS_PROXY` 环境变量

### 13.4 Extra Browser 场景

11. **`extra_browser_*` 场景分 4 种组合**：
    - 找到连接配置 + 非 PDF：代理会被注入（与注释意图相反）
    - 找到连接配置 + 是 PDF：代理会被注入（PDF 优先级更高）
    - 未找到连接配置：代理不会被注入（符合注释意图）
    - 非 extra_browser_*：正常流程，代理被注入
12. **`custom_browser_connection_url` 和 `proxy_override` 同时生效**：前者用于连接远程浏览器，后者用于配置浏览器的代理设置
13. **如果未找到连接配置**，`prefer_fetch_backend` 保持 `extra_browser_*`，不是有效 fetcher 名称，会回退到 `html_requests`

### 13.5 失败处理

14. **代理失败不会自动切换**到其他代理或直连
15. **Requests 库的重试**：仅重试网络层错误，不重试 HTTP 状态码
