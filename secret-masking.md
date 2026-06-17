# 敏感信息遮蔽策略分析报告

## 一、概述

本报告对 changedetection.io 项目中密码与敏感凭据在**配置、通知、导出**三个环节的遮蔽策略进行代码级审计，补充梳理 **RSS 访问令牌**的完整生命周期，并识别统一规则与未覆盖角落。

---

## 二、敏感字段全量清单

### 2.1 系统级敏感字段

| 字段名 | 存储路径 | 类型 | 生成方式 |
|--------|----------|------|----------|
| `password` | `settings.application.password` | 加盐哈希 | 用户设置 / `SaltyPasswordField` |
| `api_access_token` | `settings.application.api_access_token` | 16 字节 hex 令牌 | `secrets.token_hex(16)` |
| `rss_access_token` | `settings.application.rss_access_token` | 16 字节 hex 令牌 | `secrets.token_hex(16)` |
| `llm.api_key` | `settings.application.llm.api_key` | 用户提供的 API Key | 用户输入 / 环境变量 `LLM_API_KEY` |
| `notification_urls[]` | `settings.application.notification_urls` | Apprise URL（可能内嵌密码） | 用户输入 |
| `extra_proxies[].proxy_url` | `settings.requests.extra_proxies` | 代理 URL（可能内嵌凭据） | 用户输入 |
| `headers` | `settings.headers` | HTTP 请求头（可能含 Authorization） | 用户输入 |

### 2.2 Watch 级敏感字段

| 字段名 | 存储路径 | 类型 |
|--------|----------|------|
| `notification_urls[]` | `watch.notification_urls` | Apprise URL（可能内嵌密码） |
| `headers` | `watch.headers` | HTTP 请求头（可能含 Authorization） |
| `proxy` | `watch.proxy` | 代理配置（可能内嵌凭据） |

---

## 三、RSS 访问令牌机制详解

RSS 访问令牌是 changedetection.io 中一个独特的安全设计：它以**明文凭据**形式出现在 URL query 参数中，用于绕过 Flask-Login 认证直接访问 Feed。本节完整梳理其生命周期。

### 3.1 生成与存储

令牌在首次安装或数据加载时自动生成，存储于 `settings.application.rss_access_token`。

**首次安装生成** — `changedetectionio/store/__init__.py#L255-L256`：
```python
# Generate RSS access token
self.__data['settings']['application']['rss_access_token'] = secrets.token_hex(16)
```

**存量数据兜底** — `changedetectionio/store/__init__.py#L303-L307`：
```python
# Ensure RSS access token exists
if not self.__data['settings']['application'].get('rss_access_token'):
    secret = secrets.token_hex(16)
    self.__data['settings']['application']['rss_access_token'] = secret
    self.commit()
```

同样的机制也适用于 `api_access_token` — `changedetectionio/store/__init__.py#L309-L313`。

### 3.2 登录豁免

RSS 路由在全局 `@login_optionally_required` 装饰器中被豁免，不经过 Flask-Login 认证，完全依赖 URL token。

**豁免逻辑** — `changedetectionio/flask_app.py#L555-L557`：
```python
# RSS access with token is allowed
elif request.endpoint and 'rss.feed' in request.endpoint:
    return None
```

被豁免的端点包括：
- `rss.feed` — 主 Feed (`/rss`)
- `rss.rss_single_watch` — 单 Watch Feed (`/rss/watch/<uuid>`)
- `rss.rss_tag_feed` — Tag 维度 Feed (`/rss/tag/<tag_uuid>`)

### 3.3 模板传递链路

`app_rss_token` 通过多个视图函数注入模板上下文：

| 页面 | 传递位置 | 代码引用 |
|------|----------|----------|
| Watch 列表页 | `watchlist/index` 渲染参数 | `changedetectionio/blueprint/watchlist/__init__.py#L94` |
| Watch 编辑页 | `ui/edit` 渲染参数 | `changedetectionio/blueprint/ui/edit.py#L312`, `#L333` |
| Tag 管理页 | `tags/tags_overview_page` 渲染参数 | `changedetectionio/blueprint/tags/__init__.py#L28` |
| 设置页 | — | **RSS token 不在设置页单独展示**（仅 API key 展示） |

Watch 编辑页还额外构造了单 Watch RSS 链接 — `changedetectionio/blueprint/ui/edit.py#L334-L337`：
```python
'rss_uuid_feed' : {
    'label': watch.label,
    'url': url_for('rss.rss_single_watch', uuid=watch['uuid'], token=app_rss_token)
},
```

### 3.4 页面订阅链接渲染

令牌在多个模板中被直接渲染为明文 URL：

**全局 `<head>` 自动发现链接** — `changedetectionio/templates/base.html#L10-L17`：
```jinja2
{% if app_rss_token %}
    <link rel="alternate" type="application/rss+xml" 
          href="{{ url_for('rss.feed', tag=active_tag_uuid, token=app_rss_token, _external=True )}}" >
    {% if rss_uuid_feed %}
    <link rel="alternate" type="application/rss+xml" 
          href="{{ rss_uuid_feed['url'] }}" >
    {% endif %}
{% endif %}
```

**页面可见的 RSS 图标链接**：
- Watch 列表页：`changedetectionio/blueprint/watchlist/templates/watch-overview.html#L402`
- Tag 管理页：`changedetectionio/blueprint/tags/templates/groups-overview.html#L91`
- Watch 编辑页：`changedetectionio/blueprint/ui/templates/edit.html#L557`

### 3.5 Feed 访问校验

所有三个 RSS 端点均使用同一校验函数 `validate_rss_token()`，逻辑为简单的**字符串精确匹配**。

**校验函数** — `changedetectionio/blueprint/rss/_util.py#L55-L68`：
```python
def validate_rss_token(datastore, request):
    app_rss_token = datastore.data['settings']['application'].get('rss_access_token')
    rss_url_token = request.args.get('token')

    if rss_url_token != app_rss_token:
        return False, ("Access denied, bad token", 403)

    return True, None
```

**各端点调用位置**：
- 主 Feed：`changedetectionio/blueprint/rss/main_feed.py#L37-L39`
- 单 Watch Feed：`changedetectionio/blueprint/rss/single_watch.py#L35-L37`
- Tag Feed：`changedetectionio/blueprint/rss/tag.py#L27-L29`

### 3.6 RSS 令牌遮蔽状态

| 场景 | 是否遮蔽 | 说明 |
|------|----------|------|
| HTML 页面 `<head>` link | ❌ 明文 | URL query 参数中完整暴露 |
| 页面可见 RSS 图标链接 | ❌ 明文 | `href` 属性中完整暴露 |
| 设置页面 | — | 不在设置页 UI 中单独展示 |
| API 响应 | ❌ 明文 | `settings.application.rss_access_token` 未被过滤 |
| 备份文件 | ❌ 明文 | `changedetection.json` 中完整存储 |
| 重置机制 | ❌ 无 | 与 API key 不同，RSS token **没有重置接口** |

> **重要遗漏**：RSS 访问令牌缺少独立的重置功能，一旦泄露无法通过 UI 快速轮换，只能删除 `changedetection.json` 中的字段让系统重新生成。

---

## 四、三类入口遮蔽规则

### 4.1 配置入口

配置入口涵盖 Web 表单渲染、表单提交、以及 API 响应三个子环节。

#### 4.1.1 Web 表单渲染

| 字段 | 表单控件 | 渲染策略 | 代码引用 |
|------|----------|----------|----------|
| LLM API Key | `PasswordField` | GET 时显式清空 `api_key=''`；空提交不覆盖存储值 | `changedetectionio/blueprint/settings/__init__.py#L36-L41`, `#L108-L111` |
| 登录密码 | `SaltyPasswordField` | 空值或 `False` 提交时从更新字典删除，保留原密码 | `changedetectionio/forms.py#L92-L100`, `changedetectionio/blueprint/settings/__init__.py#L86-L88` |
| API Access Token | 纯文本 `<span>` | 设置页明文显示完整 token，支持一键复制 | `changedetectionio/blueprint/settings/templates/settings.html#L225-L227` |
| RSS Access Token | — | 不在设置页 UI 展示 | — |
| 通知 URL | 普通 TextArea | 完整明文渲染，无遮蔽 | — |
| 代理 URL | 普通输入框 | 完整明文渲染，无遮蔽 | — |

**API Access Token 重置接口** — `changedetectionio/blueprint/settings/__init__.py#L263-L270`：
```python
@settings_blueprint.route("/reset-api-key", methods=['GET'])
def settings_reset_api_key():
    secret = secrets.token_hex(16)
    datastore.data['settings']['application']['api_access_token'] = secret
    datastore.commit()
    flash(gettext("API Key was regenerated."))
    return redirect(url_for('settings.settings_page')+'#api')
```

#### 4.1.2 API 响应过滤

**统一过滤函数 `strip_internal_api_fields()`** — `changedetectionio/api/__init__.py#L153-L170`：
```python
def strip_internal_api_fields(data):
    if not isinstance(data, dict):
        return data
    from changedetectionio.model.schema_utils import SYSTEM_MANAGED_NON_SPEC_FIELDS
    return {
        k: v for k, v in data.items()
        if not (isinstance(k, str) and (k.startswith('__') or k in SYSTEM_MANAGED_NON_SPEC_FIELDS))
    }
```

**被过滤字段 `SYSTEM_MANAGED_NON_SPEC_FIELDS`** — `changedetectionio/model/schema_utils.py#L21-L32`：
```python
SYSTEM_MANAGED_NON_SPEC_FIELDS = frozenset({
    'last_check_status', 'last_filter_config_hash', 'restock',
    '_llm_result', '_llm_intent', '_llm_change_summary',
    'llm_prefilter', 'llm_evaluation_cache',
    'llm_last_tokens_used', 'llm_tokens_used_cumulative',
})
```

**通知 URL API** — `changedetectionio/api/Notifications.py#L10-L19`，无任何遮蔽：
```python
def get(self):
    notification_urls = self.datastore.data.get('settings', {}).get('application', {}).get('notification_urls', [])
    return {'notification_urls': notification_urls}, 200
```

#### 4.1.3 配置入口遮蔽遗漏点

| 遗漏点 | 风险等级 | 说明 |
|--------|----------|------|
| `notification_urls` 未过滤 | 🔴 高 | API 直接返回含密码的 Apprise URL |
| `rss_access_token` 未过滤 | 🔴 高 | API 响应中完整暴露 |
| `extra_proxies[].proxy_url` 未过滤 | 🟡 中 | 代理 URL 中可能含认证信息 |
| `headers` 未过滤 | 🟡 中 | `Authorization`、`Cookie` 等头原样返回 |
| `api_access_token` 未过滤 | 🟡 中 | API 响应完整返回，与设置页展示策略不一致 |

### 4.2 通知入口

通知入口涵盖 URL 处理、日志记录、调试日志三个子环节。

#### 4.2.1 通知发送流程

通知 URL 在发送前会经过 Jinja2 渲染，然后被 Apprise 消费。

**核心发送逻辑** — `changedetectionio/notification/handler.py#L416-L510`：
```python
for url in n_object['notification_urls']:
    url = jinja_render(template_str=url, **notification_parameters)
    logger.info(f">> Process Notification: AppRise start notifying '{url}'")
    # ... Apprise 发送 ...
```

#### 4.2.2 日志记录点

| 日志位置 | 记录内容 | 代码引用 | 风险 |
|----------|----------|----------|------|
| 应用日志（loguru INFO） | 完整渲染后 URL | `changedetectionio/notification/handler.py#L430` | URL 含密码明文 |
| 通知调试日志 | 完整 `sent_obj` JSON（含 url、title、body、original_context） | `changedetectionio/flask_app.py#L1098-L1107` | 完整通知上下文 |
| 通知错误日志 | `str(e)` 异常信息 | `changedetectionio/flask_app.py#L1090-L1099` | 可能包含敏感上下文 |
| Apprise 内部日志 | DEBUG 级别捕获 | `changedetectionio/notification/handler.py` | 取决于 Apprise 实现 |

**调试日志写入** — `changedetectionio/flask_app.py#L1104-L1107`：
```python
notification_debug_log += ["{} - SENDING - {}".format(
    now.strftime("%c"), 
    json.dumps(sent_obj)  # 包含完整 url、title、body、original_context
)]
notification_debug_log = notification_debug_log[-100:]  # 保留最近 100 条
```

调试日志可通过 `/settings/notification-logs` 查看 — `changedetectionio/blueprint/settings/__init__.py#L272-L278`。

#### 4.2.3 通知入口遮蔽遗漏点

| 遗漏点 | 风险等级 | 说明 |
|--------|----------|------|
| 通知 URL 在应用日志中明文 | 🔴 高 | `logger.info` 直接输出含密码的 URL |
| 调试日志记录完整 `sent_obj` | 🔴 高 | 含 URL 密码、通知正文、完整上下文 |
| Apprise DEBUG 日志无过滤 | 🟡 中 | 内部日志可能包含敏感信息 |
| 异常信息无敏感扫描 | 🟡 中 | `str(e)` 可能泄露认证失败等信息 |

### 4.3 导出入口

导出入口即备份功能，涵盖备份创建、下载、导入三个子环节。

#### 4.3.1 备份内容清单

备份通过 ZIP 格式创建，包含以下文件 — `changedetectionio/blueprint/backups/__init__.py#L16-L91`：

| 文件 | 内容 | 敏感信息 |
|------|------|----------|
| `changedetection.json` | 全局配置 | 所有系统级敏感字段**完整明文** |
| `{uuid}/watch.json` | 单 Watch 配置 | Watch 级 `notification_urls`、`headers`、`proxy` |
| `{uuid}/tag.json` | Tag 配置 | 一般无敏感信息 |
| `{uuid}/history/*` | 历史快照/截图 | 页面内容可能含敏感数据 |
| `{uuid}/processed.txt` | 处理后文本 | 同上 |
| `url-list.txt` | URL 列表 | 一般无敏感信息 |
| `url-list-with-tags.txt` | 带 Tag 的 URL 列表 | 一般无敏感信息 |

#### 4.3.2 备份下载与导入

**下载** — `changedetectionio/blueprint/backups/__init__.py#L144-L166`：
- 需要 Flask-Login 登录认证
- 文件名白名单校验防止路径遍历

**导入** — `changedetectionio/blueprint/imports/importer.py`：
- 直接还原所有配置，包括敏感信息明文

#### 4.3.3 导出入口遮蔽遗漏点

| 遗漏点 | 风险等级 | 说明 |
|--------|----------|------|
| 备份 ZIP 全部明文 | 🔴 高 | 所有密钥、密码、令牌完整存储 |
| 无"安全备份"选项 | 🔴 高 | 无法导出不含敏感信息的配置用于分享 |
| 导入不校验敏感来源 | 🟡 中 | 导入任意备份都会覆盖现有密钥 |

---

## 五、专项保护机制盘点

### 5.1 LLM API Key 专项保护

LLM 密钥拥有项目中最完善的安全保障，覆盖 **11 个 API 端点 + 设置页面 HTML**。

**金丝雀测试** — `changedetectionio/tests/test_llm_api_key_security.py`：
```python
CANARY_KEY = 'sk-CANARY-SECRET-DO-NOT-EXPOSE-12345'
def _key_in_response(response, key=CANARY_KEY) -> bool:
    body = response.data.decode('utf-8', errors='replace')
    return key in body
```

**测试覆盖的端点**：
- `GET /api/v1/watch/<uuid>`、`GET /api/v1/watch`、`PUT /api/v1/watch/<uuid>`
- `GET /api/v1/tag/<uuid>`、`GET /api/v1/tags`
- `GET /api/v1/systeminfo`、`GET /api/v1/search`、`GET /api/v1/full-spec`
- `GET/POST/PUT /api/v1/notifications`
- 设置页面 HTML 源码

**CSRF 凭据泄露防护（GHSA-g36r-fm2p-87xm）**：
当 `api_base` 与存储值不匹配时，必须显式提供 `api_key`，防止 CSRF 将密钥发送到攻击者服务器。

**SSRF 防护（GHSA-jrxm-qjfh-g54f）**：
默认拒绝 `127.0.0.1`、`localhost`、`10.0.0.0/8`、`192.168.0.0/16`、`169.254.169.254` 等内网地址，可通过 `ALLOW_IANA_RESTRICTED_ADDRESSES=true` 绕过。

### 5.2 现有机制局限性

| 机制 | 覆盖范围 | 未覆盖 |
|------|----------|--------|
| `PasswordField` + GET 清空 | LLM API Key、登录密码（Web 表单） | API 响应、通知 URL、代理 URL |
| `strip_internal_api_fields()` | `__` 前缀字段、LLM 运行时状态 | `notification_urls`、`headers`、`proxy_url`、`rss_access_token`、`api_access_token` |
| LLM 金丝雀测试 | LLM `api_key` | 其他所有敏感字段 |
| SSRF 防护 | LLM `api_base` | 通知 URL、代理 URL |
| 凭据泄露防护 | LLM `api_key` + `api_base` | 其他场景 |
| 登录认证 | 备份下载、设置页 | 备份文件内容本身、RSS 端点 |

---

## 六、统一遮蔽规则建议

### 6.1 建议的敏感字段注册表

```python
SENSITIVE_FIELDS = frozenset({
    # 系统级令牌
    'api_access_token',
    'rss_access_token',
    'password',
    # LLM
    'llm.api_key',
    # 通知
    'notification_urls',
    # 代理
    'proxy_url',
    'extra_proxies',
    # HTTP 头
    'headers',
})
```

### 6.2 URL 密码遮蔽工具函数

```python
from urllib.parse import urlparse, urlunparse

def mask_url_password(url: str) -> str:
    """
    遮蔽 URL 中的用户信息部分（密码置为 ****，保留用户名）
    例: http://user:pass@host.com:8080/path -> http://user:****@host.com:8080/path
    """
    try:
        parsed = urlparse(url)
        if not parsed.password:
            return url
        netloc = ""
        if parsed.username:
            netloc = f"{parsed.username}:****"
            if parsed.hostname:
                netloc += f"@{parsed.hostname}"
        elif parsed.hostname:
            netloc = parsed.hostname
        if parsed.port:
            netloc += f":{parsed.port}"
        return urlunparse(parsed._replace(netloc=netloc))
    except Exception:
        return url
```

### 6.3 三类入口遮蔽规则

#### 规则 A：配置入口（API + UI）

| 字段 | API 响应 | UI 展示 |
|------|----------|---------|
| `notification_urls[]` | 逐个 URL 调用 `mask_url_password()` | 同左 |
| `extra_proxies[].proxy_url` | 调用 `mask_url_password()` | 同左 |
| `headers` | 移除 `Authorization`、`Proxy-Authorization`、`Cookie` | 同左 |
| `api_access_token` | 只显示后 4 位，如 `...abcd` | 保持现状（完整显示 + 重置按钮） |
| `rss_access_token` | 只显示后 4 位，如 `...abcd` | 在设置页新增展示 + 重置接口 |
| `llm.api_key` | **不返回该字段**（已实现） | GET 时清空（已实现） |
| `password` | **不返回该字段** | 空值保留（已实现） |

#### 规则 B：通知入口（日志）

| 日志类型 | 处理方式 |
|----------|----------|
| 应用日志（`logger.info`） | 通知 URL 先遮蔽再记录 |
| 通知调试日志 | `sent_obj['url']` 遮蔽；移除 `original_context` 或脱敏 |
| 错误日志 | 对异常信息运行敏感信息正则扫描 |
| Apprise 内部日志 | 重定向到遮蔽后的 logger wrapper |

#### 规则 C：导出入口（备份）

| 方案 | 说明 |
|------|------|
| **方案 1（最小改动）** | 备份下载时增加警告："备份包含所有密钥，请妥善保管" |
| **方案 2（推荐）** | 新增 `?safe=true` 参数，创建时自动脱敏所有敏感字段 |
| **方案 3（完整）** | 备份加密（需引入密码派生 + AES-GCM） |

---

## 七、代码参考索引

> 所有路径均为仓库根目录相对路径

### RSS 访问令牌
- 生成与兜底: `changedetectionio/store/__init__.py#L255-L256`, `#L303-L313`
- 登录豁免: `changedetectionio/flask_app.py#L555-L557`
- 访问校验: `changedetectionio/blueprint/rss/_util.py#L55-L68`
- 主 Feed 端点: `changedetectionio/blueprint/rss/main_feed.py#L37-L39`
- 单 Watch Feed: `changedetectionio/blueprint/rss/single_watch.py#L35-L37`
- Tag Feed: `changedetectionio/blueprint/rss/tag.py#L27-L29`
- 模板传递: `changedetectionio/blueprint/watchlist/__init__.py#L94`, `changedetectionio/blueprint/ui/edit.py#L312-L337`, `changedetectionio/blueprint/tags/__init__.py#L28`
- `<head>` link 渲染: `changedetectionio/templates/base.html#L10-L17`
- UI 可见链接: `changedetectionio/blueprint/watchlist/templates/watch-overview.html#L402`, `changedetectionio/blueprint/tags/templates/groups-overview.html#L91`, `changedetectionio/blueprint/ui/templates/edit.html#L557`

### 配置入口
- LLM API Key 表单: `changedetectionio/blueprint/settings/__init__.py#L36-L41`, `#L108-L111`
- 密码表单处理: `changedetectionio/forms.py#L92-L100`, `changedetectionio/blueprint/settings/__init__.py#L86-L88`
- API Key 展示与重置: `changedetectionio/blueprint/settings/templates/settings.html#L225-L231`, `changedetectionio/blueprint/settings/__init__.py#L263-L270`
- API 字段过滤: `changedetectionio/api/__init__.py#L153-L170`
- 系统管理字段: `changedetectionio/model/schema_utils.py#L21-L32`
- 通知 URL API: `changedetectionio/api/Notifications.py#L10-L19`

### 通知入口
- 通知发送: `changedetectionio/notification/handler.py#L416-L510`
- 调试日志写入: `changedetectionio/flask_app.py#L1098-L1107`
- 调试日志查看: `changedetectionio/blueprint/settings/__init__.py#L272-L278`

### 导出入口
- 备份创建: `changedetectionio/blueprint/backups/__init__.py#L16-L91`
- 备份下载: `changedetectionio/blueprint/backups/__init__.py#L144-L166`
- 导入逻辑: `changedetectionio/blueprint/imports/importer.py`

### 安全测试
- LLM API Key 专项测试: `changedetectionio/tests/test_llm_api_key_security.py`
- API 内部字段过滤测试: `changedetectionio/tests/test_api.py#L409-L457`

---

## 八、总结

### ✅ 现有优势
1. LLM API Key 有完善的专项保护（金丝雀测试覆盖 11 端点 + SSRF 防护 + CSRF 凭据泄露防护）
2. 内部/临时字段有统一过滤机制 `strip_internal_api_fields()`
3. Web 表单密码字段使用 `PasswordField`，GET 清空、空提交不覆盖
4. 备份下载需要登录认证
5. RSS 端点独立 token 认证，与会话隔离

### ❌ 主要缺失（按优先级）
| 优先级 | 缺失项 | 影响范围 |
|--------|--------|----------|
| 🔴 P0 | 通知 URL 密码无遮蔽（API + 日志 + 调试日志） | 通知服务凭据泄露 |
| 🔴 P0 | RSS 令牌无重置接口 + API 明文暴露 | 令牌泄露后无法快速轮换 |
| 🔴 P0 | 备份文件全部敏感信息明文 | 备份文件泄露即全盘泄露 |
| 🟡 P1 | 代理 URL 密码无遮蔽 | 代理服务凭据泄露 |
| 🟡 P1 | HTTP 认证头无遮蔽 | `Authorization` 等头泄露 |
| 🟡 P1 | API 响应中 `api_access_token`、`rss_access_token` 完整返回 | 与 UI 展示策略不一致 |
| 🟢 P2 | 缺少统一的敏感字段管理工具类 | 未来易新增泄露点 |
