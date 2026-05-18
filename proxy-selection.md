# 代理选择策略完整说明

## 1. 全局代理池配置

### 1.1 代理配置来源

代理池由以下几部分组成，在 `store/__init__.py:815-853` 中构建：

1. **环境变量配置的代理**：通过 `HTTP_PROXY`、`HTTPS_PROXY` 等环境变量配置的系统代理
2. **UI配置的额外代理**：在设置页面通过 `extra_proxies` 配置的自定义代理（`forms.py:963-974`）
3. **no-proxy 选项**：当 `ENABLE_NO_PROXY_OPTION` 环境变量为 `True` 时自动添加（`store/__init__.py:850-851`）

### 1.2 代理数据结构

```python
proxy_list = {
    "proxy-key": {
        "label": "代理显示名称",
        "url": "socks5://user:pass@host:port"  # 实际代理URL
    },
    "no-proxy": {
        "label": "No proxy",
        "url": ""
    }
}
```

### 1.3 全局默认代理

在 `settings/requests/proxy` 中配置全局默认代理，在 `blueprint/settings/__init__.py:52-60` 中初始化。

---

## 2. Watch 级别代理覆盖

### 2.1 配置方式

每个 Watch 可以在编辑页面单独选择代理，存储在 `watch['proxy']` 字段中：

- **空字符串 `''`**：使用系统默认代理
- **`"no-proxy"`**：不使用任何代理
- **具体代理 key**：使用指定的代理

### 2.2 表单处理

在 `blueprint/ui/edit.py:171-177` 中构建代理选择下拉框：
- 第一个选项为 `('Default', '')`
- 后续选项为所有可用代理

在 `blueprint/ui/edit.py:202-203` 中保存时，如果代理值为空字符串，则设置为 `None`。

---

## 3. 代理选择判定顺序

### 3.1 核心选择逻辑

代理选择的核心逻辑在 `store/__init__.py:855-886` 的 `get_preferred_proxy_for_watch()` 方法中，优先级从高到低：

```
1. 如果代理池为空 → 返回 None
2. 如果 ENABLE_NO_PROXY_OPTION=True 且 watch['proxy'] == "no-proxy" → 返回 None (不使用代理)
3. 如果 watch['proxy'] 存在且在代理列表中 → 返回该代理 key
4. 尝试使用全局默认代理 (settings['requests']['proxy'])
5. 如果全局默认代理有效 → 返回该代理 key
6. 否则返回代理列表中的第一个可用代理
```

### 3.2 no-proxy 判定

no-proxy 的判定在 `store/__init__.py:868-869`：
```python
if strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')) and watch.get('proxy') == "no-proxy":
    return None
```

返回 `None` 表示不使用任何代理。

---

## 4. 抓取层代理传递

### 4.1 代理选择到代理 URL 的转换

在 `processors/base.py:117-192` 的 `call_browser()` 方法中完成代理选择和传递：

1. 调用 `get_preferred_proxy_for_watch()` 获取代理 key
2. 如果代理 key 存在且不是自定义浏览器端点，从 `proxy_list` 获取实际代理 URL
3. 将代理 URL 作为 `proxy_override` 参数传递给 fetcher 构造函数

```python
# processors/base.py:176-189
proxy_url = None
if preferred_proxy_id:
    if not prefer_fetch_backend.startswith('extra_browser_'):
        proxy_url = self.datastore.proxy_list.get(preferred_proxy_id).get('url')

self.fetcher = fetcher_obj(proxy_override=proxy_url,
                           custom_browser_connection_url=custom_browser_connection_url,
                           screenshot_format=self.screenshot_format)
```

### 4.2 各 Fetcher 的代理处理

#### 4.2.1 Requests Fetcher (`content_fetchers/requests.py`)

```python
# requests.py:45-57
if self.proxy_override:
    proxies = {'http': self.proxy_override, 'https': self.proxy_override, 'ftp': self.proxy_override}
else:
    if self.system_http_proxy:
        proxies['http'] = self.system_http_proxy
    if self.system_https_proxy:
        proxies['https'] = self.system_https_proxy
```

直接将 `proxy_override` 设置为 requests 的 `proxies` 字典。

#### 4.2.2 Playwright Fetcher (`content_fetchers/playwright.py`)

```python
# playwright.py:196-215
if proxy_override:
    self.proxy = {'server': proxy_override}

# 解析用户名密码
if self.proxy:
    parsed = urlparse(self.proxy.get('server'))
    if parsed.username:
        self.proxy['username'] = parsed.username
        self.proxy['password'] = parsed.password
```

在创建浏览器上下文时传入：
```python
context = await browser.new_context(
    proxy=self.proxy,
    # ... 其他参数
)
```

#### 4.2.3 Puppeteer Fetcher (`content_fetchers/puppeteer.py`)

```python
# puppeteer.py:210-225
if proxy_override:
    parsed = urlparse(proxy_override)
    if parsed:
        self.proxy = {'username': parsed.username, 'password': parsed.password}
        # 将代理服务器地址附加到浏览器连接URL
        proxy_url = parsed.scheme + "://" if parsed.scheme else 'http://'
        proxy_url += f"{parsed.hostname}{port}{parsed.path}{q}"
        self.browser_connection_url += f"{r}--proxy-server={proxy_url}"
```

#### 4.2.4 Selenium WebDriver Fetcher (`content_fetchers/webdriver_selenium.py`)

```python
# webdriver_selenium.py:45-62
proxy_sources = [
    self.system_http_proxy,
    self.system_https_proxy,
    # ... 其他环境变量代理
    proxy_override,  # 最后一个会覆盖前面的
]
for k in filter(None, proxy_sources):
    self.proxy_url = k.strip()

# 传递给 Chrome
if self.proxy_url:
    options.add_argument(f'--proxy-server={self.proxy_url}')
```

---

## 5. 失败回退机制

### 5.1 代理失败处理

**当前代码库中没有代理失败自动回退机制**。当代理失败时：

1. **异常捕获**：在 `worker.py` 中捕获各种异常（如 `ProxyError`、`ConnectionError`、`BrowserConnectError` 等）
2. **错误记录**：将错误信息记录到 `watch['last_error']` 中
3. **不自动切换代理**：不会自动尝试其他代理

### 5.2 Requests 重试机制

Requests fetcher 有内置的重试机制（`content_fetchers/requests.py:68-80`），但仅限于：
- 连接超时
- 读取超时
- 连接重置

**注意**：这是网络级别的重试，不会切换到其他代理。重试策略：
- 最大重试次数：`REQUESTS_RETRY_MAX_COUNT` 环境变量，默认 6 次
- 退避因子：0.5 秒
- 重试方法：HEAD、GET、OPTIONS、POST

### 5.3 代理认证失败

当遇到 407 错误时（`worker.py:243-244`），会给出明确的错误提示：
> "Error - 407 (Proxy authentication required) received, did you need a username and password for the proxy?"

---

## 6. 完整流程图

```
抓取任务开始
    ↓
调用 get_preferred_proxy_for_watch(uuid)
    ├─ 代理池为空? → 无代理
    ├─ watch.proxy == "no-proxy"? → 无代理
    ├─ watch.proxy 在代理列表中? → 使用该代理
    ├─ 全局默认代理有效? → 使用全局默认
    └─ 否则 → 使用代理列表第一个
    ↓
获取代理 URL (proxy_url)
    ↓
根据 fetch_backend 选择 fetcher
    ├─ html_requests → requests fetcher
    │   └─ 设置 proxies 字典
    ├─ html_webdriver → playwright/puppeteer/selenium
    │   ├─ playwright: 传入 context proxy 参数
    │   ├─ puppeteer: 附加 --proxy-server 到连接URL
    │   └─ selenium: 添加 --proxy-server Chrome 参数
    └─ 其他自定义 fetcher
    ↓
执行抓取
    ├─ 成功 → 正常处理
    └─ 失败 → 记录错误，不回退到其他代理
```

---

## 7. 关键代码位置汇总

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 代理池构建 | `store/__init__.py` | 815-853 |
| 代理选择逻辑 | `store/__init__.py` | 855-886 |
| no-proxy 判定 | `store/__init__.py` | 868-869 |
| 代理传递到 fetcher | `processors/base.py` | 176-192 |
| Requests 代理处理 | `content_fetchers/requests.py` | 45-57 |
| Playwright 代理处理 | `content_fetchers/playwright.py` | 196-215 |
| Puppeteer 代理处理 | `content_fetchers/puppeteer.py` | 210-225 |
| Selenium 代理处理 | `content_fetchers/webdriver_selenium.py` | 45-103 |
| Watch 代理表单 | `blueprint/ui/edit.py` | 171-177, 202-203 |
| 全局代理设置 | `blueprint/settings/__init__.py` | 52-80 |
