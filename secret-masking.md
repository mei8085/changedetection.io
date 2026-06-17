# 敏感信息遮蔽策略分析报告

## 一、概述

本报告对 changedetection.io 项目中密码与敏感凭据在 **公开API、HTML配置页、持久化配置、备份文件** 四个层面的处理方式进行代码级审计。重点区分：

1. **公开 API 的鉴权机制** 与 **敏感信息返回范围**（含 OpenAPI spec 与数据 API 的差异）
2. **Watch 级 proxy 选择名** 与 **额外代理 URL 凭据** 的本质区别
3. 各层面哪些信息对外暴露，哪些仅保留在配置或备份中

---

## 二、四层暴露矩阵总览

| 敏感字段 | 公开 API（需Token¹） | HTML配置页（需登录） | 持久化配置（本地磁盘） | 备份文件（可下载） |
|----------|---------------------|---------------------|----------------------|-------------------|
| `llm.api_key` | ❌ 不暴露（无settings端点） | ✅ GET时清空（PasswordField） | ✅ 明文 | ✅ 明文 |
| `password` | ❌ 不暴露 | ✅ 不回显（存储为加盐哈希） | ✅ 加盐哈希 | ✅ 加盐哈希 |
| `api_access_token` | ❌ 不暴露（无settings端点） | ✅ 完整明文展示 + 重置按钮 | ✅ 明文 | ✅ 明文 |
| `rss_access_token` | ❌ 不暴露（无settings端点） | ⚠️ 仅在HTML `<head>` 和 RSS 链接中明文 | ✅ 明文 | ✅ 明文 |
| `notification_urls[]`（全局） | ⚠️ 明文（`/api/v1/notifications`） | ✅ 完整明文（TextArea） | ✅ 明文 | ✅ 明文 |
| `notification_urls[]`（Watch级） | ⚠️ 明文（`/api/v1/watch/<uuid>`） | ✅ 完整明文（Watch编辑页） | ✅ 明文 | ✅ 明文 |
| `proxy`（Watch级选择名²） | ⚠️ 明文（仅 key 名称，不含凭据） | ✅ 下拉选择项，展示名称 | ✅ 明文 | ✅ 明文 |
| `extra_proxies[].proxy_url` | ❌ 不暴露（无settings端点） | ✅ 完整明文（StringField，含凭据） | ✅ 明文 | ✅ 明文 |
| `proxies.json` 代理 URL | ❌ 不暴露 | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |
| `headers`（全局） | ❌ 不暴露 | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |
| `headers`（Watch级） | ⚠️ 明文（`/api/v1/watch/<uuid>`） | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |
| OpenAPI spec | 🌐 **公开无鉴权** | — | — | — |

> **图例**: ✅ 明文 / ⚠️ 部分暴露 / ❌ 不暴露 / 🌐 完全公开

> **注1**: API 鉴权可通过 `api_access_token_enabled` 开关关闭，关闭后所有数据 API 完全公开。
>
> **注2**: Watch 级 `proxy` 字段存储的是 **proxy 选择名/key**（如 `"ui-0myproxy"`），不是完整代理 URL，因此不直接包含凭据。但通过 key 可在前端或 API 客户端反查对应的代理 URL。

---

## 三、公开 API 暴露分析

### 3.1 API 鉴权机制

所有数据 API 通过 `@auth.check_token` 装饰器保护，校验逻辑如下：

**鉴权函数** — `changedetectionio/api/auth.py#L8-L25`

```python
def check_token(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        datastore = args[0].datastore
        config_api_token_enabled = datastore.data['settings']['application'].get('api_access_token_enabled')
        config_api_token = datastore.data['settings']['application'].get('api_access_token')

        # config_api_token_enabled - a UI option in settings if access should obey the key or not
        if config_api_token_enabled:
            if request.headers.get('x-api-key') != config_api_token:
                return make_response(
                    jsonify("Invalid access - API key invalid."), 403
                )

        return f(*args, **kwargs)
    return decorated
```

**关键特性**:
- 通过 HTTP Header `x-api-key` 传递令牌
- 令牌可通过设置页开关（`api_access_token_enabled`）关闭，关闭后所有数据 API 完全公开
- 令牌存储于 `settings.application.api_access_token`，16 字节 hex

**受保护的端点**（22 个，均带 `@auth.check_token`）:
- Watch CRUD: `GET/PUT/DELETE /api/v1/watch/<uuid>`、`POST /api/v1/watch`、`GET /api/v1/watch`
- Watch 历史: `GET /api/v1/watch/<uuid>/history` 等 4 个历史端点
- Tag CRUD: `GET/PUT/POST /api/v1/tag(s)` 等 5 个端点
- 通知: `GET/POST/PUT/DELETE /api/v1/notifications`
- 其他: `GET /api/v1/systeminfo`、`GET /api/v1/search`、`POST /api/v1/import`

---

### 3.2 OpenAPI spec 端点 — 公开无鉴权

**重要发现**: `GET /api/v1/full-spec` **不需要鉴权**，完全公开。

**注册位置** — `changedetectionio/flask_app.py#L606`

```python
watch_api.add_resource(Spec, '/api/v1/full-spec')
```

**实现代码** — `changedetectionio/api/Spec.py#L14-L21`

```python
class Spec(Resource):
    def get(self):
        """Return the merged OpenAPI spec including all registered processor extensions."""
        return make_response(
            _get_spec_yaml(),
            200,
            {'Content-Type': 'application/yaml'}
        )
```

**暴露内容**:
- 所有 API 端点路径与参数定义
- 所有 Schema 定义（Watch、Tag、Notification 等的字段结构）
- 处理器扩展的 schema（如 `processor_config_restock_diff`）
- x-code-samples 示例代码

**风险评估**:
- 低风险：仅暴露 API 结构文档，不包含实际数据
- 但可帮助攻击者了解 API 接口，为后续攻击提供便利
- 与 `api_access_token_enabled` 开关无关，始终公开

---

### 3.3 数据 API 敏感信息返回范围

#### `/api/v1/notifications` — 全局通知 URL 明文返回
**代码位置**: `changedetectionio/api/Notifications.py#L10-L19`

```python
def get(self):
    notification_urls = self.datastore.data.get('settings', {}).get('application', {}).get('notification_urls', [])        
    return {'notification_urls': notification_urls}, 200
```
**风险**: Apprise URL 通常包含明文密码/令牌，如 `mailto://user:pass@smtp.example.com`、`slack://xoxb-TOKEN@CHANNEL`。

---

#### `/api/v1/watch/<uuid>` — 单 Watch 完整配置
**代码位置**: `changedetectionio/api/Watch.py#L66-L131`

```python
def get(self, uuid):
    # ... 操作逻辑 ...
    with self.datastore.lock:
        watch = dict(watch_obj)
    # ... 添加额外字段（history_n, last_changed, viewed, link, processor_config 等） ...
    return strip_internal_api_fields(watch)
```

**过滤逻辑 `strip_internal_api_fields()`** — `changedetectionio/api/__init__.py#L153-L170`:
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

**被过滤字段 `SYSTEM_MANAGED_NON_SPEC_FIELDS`** — `changedetectionio/model/schema_utils.py#L21-L32`:
```python
SYSTEM_MANAGED_NON_SPEC_FIELDS = frozenset({
    'last_check_status', 'last_filter_config_hash', 'restock',
    '_llm_result', '_llm_intent', '_llm_change_summary',
    'llm_prefilter', 'llm_evaluation_cache',
    'llm_last_tokens_used', 'llm_tokens_used_cumulative',
})
```

**返回的敏感字段（明文）**:
| 字段 | 类型 | 说明 |
|------|------|------|
| `notification_urls[]` | list[string] | Watch 级通知 URL，可能含密码/令牌 |
| `headers` | object | Watch 级 HTTP 请求头，可能含 `Authorization` |
| `proxy` | string | **Proxy 选择名/key**（非完整 URL，见 3.4 节） |
| `browser_steps[]` | list | 浏览器步骤，可能含表单密码等敏感操作 |

**不返回的系统级敏感字段**:
- `api_access_token`、`rss_access_token`、`llm.api_key`、`password` — 这些在全局 settings 中，无 settings API 端点

---

#### `/api/v1/watch` — Watch 列表（精简版）
**代码位置**: `changedetectionio/api/Watch.py#L538-L559`

仅返回简化字段，不含敏感配置：
```python
list[uuid] = {
    'last_changed': watch.last_changed,
    'last_checked': watch['last_checked'],
    'last_error': watch['last_error'],
    'link': watch.link,
    'page_title': watch['page_title'],
    'tags': [*tags],
    'title': watch['title'],
    'url': watch['url'],
    'viewed': watch.viewed
}
```

---

#### `/api/v1/systeminfo` — 系统信息
**代码位置**: `changedetectionio/api/SystemInfo.py#L11-L40`

仅返回非敏感信息（队列大小、超期 watch 数、运行时间、watch 数量、版本号）。

---

### 3.4 Watch 级 proxy 选择名 vs 代理 URL 凭据

这是两个不同层次的概念，容易混淆。

#### Watch 级 `proxy` 字段 — 选择名/Key

**存储内容**: 字符串 key，如 `"ui-0my-proxy"`、`"no-proxy"`、`"proxiesjson-key1"`

**用途**: 作为索引，从 `proxy_list` 字典中查找实际代理配置。

**API 返回**: 明文返回 key 名称，**不直接包含凭据**。

**代码证据**:
- Watch 存储: `watch['proxy']` 是字符串 key
- 代理列表获取: `changedetectionio/store/__init__.py#L826-L853`
- Watch 级代理解析: `changedetectionio/store/__init__.py#L855-L879`

```python
def get_preferred_proxy_for_watch(self, uuid):
    """Returns the preferred proxy by ID key"""
    if self.proxy_list is None:
        return None
    watch = self.data['watching'].get(uuid)
    if strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')) and watch.get('proxy') == "no-proxy":
        return None
    if watch.get('proxy') and watch.get('proxy') in list(self.proxy_list.keys()):
        return watch.get('proxy')
    # ... 否则使用系统默认代理
```

#### `proxy_list` 字典 — 实际代理配置

**数据结构**:
```python
proxy_list = {
    "ui-0my-proxy": {
        'label': "My Proxy",           # 显示名称
        'url': "http://user:pass@..."  # 完整代理 URL（含凭据）
    },
    "no-proxy": {
        'label': "No proxy",
        'url': ''
    },
    # ... 来自 proxies.json 的更多条目
}
```

**两个来源**:
1. **外部文件 `proxies.json`** — 磁盘上独立的配置文件
2. **UI 配置 `extra_proxies`** — 存储在 `changedetection.json` 的 `settings.requests.extra_proxies` 中

#### 关键区别总结

| 维度 | Watch `proxy` 字段 | `extra_proxies[].proxy_url` |
|------|-------------------|---------------------------|
| 存储内容 | 选择名/key 字符串 | 完整代理 URL（可能含 user:pass） |
| API 暴露 | 明文暴露 key 名称 | 不直接暴露（无 settings API） |
| 敏感程度 | 低（仅名称） | 高（含凭据） |
| 安全性 | ⚠️ 通过 key 可反查 URL | ❌ URL 中凭据明文存储 |

> **注意**: 虽然 Watch API 只返回 proxy key，但如果攻击者能访问设置页或配置文件，仍可通过 key 找到对应的完整 URL。

---

## 四、HTML 配置页暴露分析

HTML 页面需通过 Flask-Login 登录认证（或密码保护）。

### 4.1 设置页敏感字段展示

#### LLM API Key — GET 时清空
**代码位置**: `changedetectionio/blueprint/settings/__init__.py#L36-L41`

```python
# api_key is intentionally blanked on GET — PasswordField never re-renders
# its value, and a blank submission preserves the stored key.
default['llm'] = LLMSettings.model_validate(
    datastore.data['settings']['application'].get('llm') or {}
).model_dump()
default['llm']['api_key'] = ''  # 显式清空
```

**POST 处理** — `changedetectionio/blueprint/settings/__init__.py#L108-L111`:
```python
if not (llm_form_input.get('api_key') or '').strip():
    llm_form_input.pop('api_key', None)
```

**模板** — `changedetectionio/blueprint/settings/templates/settings_llm_tab.html#L128`:
```jinja2
{{ render_field(form.llm.form.api_key) }}
```
> WTForms `PasswordField` 不在 HTML 中渲染 `value` 属性。

---

#### 登录密码 — SaltyPasswordField
**代码位置**: `changedetectionio/forms.py#L92-L100`

使用自定义 `SaltyPasswordField`，空值或 `False` 提交时从更新字典删除：
```python
if 'password' in app_update and not app_update['password']:
    del (app_update['password'])
```
> 存储为加盐哈希，非明文。

---

#### API Access Token — 完整明文展示
**代码位置**: `changedetectionio/blueprint/settings/templates/settings.html#L225-L227`

```jinja2
<span id="api-key">{{api_key}}</span>
<a href="{{url_for('settings.settings_reset_api_key')}}" class="pure-button button-small button-cancel">
    {{ _('Regenerate API key') }}
</a>
```

**重置接口** — `changedetectionio/blueprint/settings/__init__.py#L263-L270`:
```python
@settings_blueprint.route("/reset-api-key", methods=['GET'])
def settings_reset_api_key():
    secret = secrets.token_hex(16)
    datastore.data['settings']['application']['api_access_token'] = secret
    datastore.commit()
    flash(gettext("API Key was regenerated."))
    return redirect(url_for('settings.settings_page')+'#api')
```

---

#### RSS Access Token — 隐式暴露
RSS token **不在设置页 UI 中单独展示**，但通过多个模板隐式暴露：

**全局 `<head>` 自动发现链接** — `changedetectionio/templates/base.html#L10-L17`:
```jinja2
{% if app_rss_token %}
    <link rel="alternate" type="application/rss+xml" 
          href="{{ url_for('rss.feed', tag=active_tag_uuid, token=app_rss_token, _external=True )}}" >
{% endif %}
```

**页面可见 RSS 图标链接**:
- Watch 列表页: `changedetectionio/blueprint/watchlist/templates/watch-overview.html#L402`
- Tag 管理页: `changedetectionio/blueprint/tags/templates/groups-overview.html#L91`
- Watch 编辑页: `changedetectionio/blueprint/ui/templates/edit.html#L557`

**模板传递链路**（视图函数注入 `app_rss_token`）:
- Watch 列表: `changedetectionio/blueprint/watchlist/__init__.py#L94`
- Watch 编辑: `changedetectionio/blueprint/ui/edit.py#L312`, `#L333`
- Tag 管理: `changedetectionio/blueprint/tags/__init__.py#L28`

> **遗漏**: RSS token 没有重置接口，泄露后只能删除 `changedetection.json` 中的字段让系统重新生成。

---

#### 通知 URL — 完整明文
**代码位置**: `changedetectionio/blueprint/settings/templates/settings.html#L110`

```jinja2
{{ render_common_settings_form(form.application.form, emailprefix, settings_application, extra_notification_token_placeholder_info) }}
```
通知 URL 以 TextArea 完整明文渲染，无任何遮蔽。

---

#### 代理配置 — 完整明文（StringField，非 PasswordField）

代理配置有两部分：

**1. 默认代理选择（RadioField）** — `changedetectionio/forms.py#L1007`

```python
proxy = RadioField(_l('Default proxy'))
```
展示代理名称列表，选择的是 **proxy key**，不直接显示 URL 凭据。

**2. 额外代理配置（FieldList + FormField）** — `changedetectionio/forms.py#L973-L984`

```python
class SingleExtraProxy(Form):
    proxy_name = StringField(_l('Name'), ...)
    proxy_url = StringField(_l('Proxy URL'), [
        validators.Optional(),
        ValidateStartsWithRegex(regex=r'^(https?|socks5)://', ...),
        ValidateSimpleURL()
    ], render_kw={"placeholder": "socks5:// or regular proxy http://user:pass@...:3128", "size":50})
```

**模板渲染** — `changedetectionio/blueprint/settings/templates/settings.html#L360-L363`:
```jinja2
<div class="pure-control-group" id="extra-proxies-setting">
    {{ render_fieldlist_with_inline_errors(form.requests.form.extra_proxies) }}
    <span class="pure-form-message-inline">{{ _('SOCKS5 proxies with authentication are only supported...') }}</span>
</div>
```

**关键问题**: `proxy_url` 使用 **`StringField`** 而非 `PasswordField`，代理 URL 中的用户名和密码（如 `http://user:pass@proxy.example.com:8080`）**以完整明文形式展示在 HTML 页面中**，包括：
- 页面 HTML 源码
- 浏览器开发者工具
- 可能的浏览器历史缓存

> **修正说明**: 之前的分析笼统说"代理 URL 明文"，现在明确：Watch 级 `proxy` 字段只是选择名（低敏感），而 `extra_proxies[].proxy_url` 是完整 URL（高敏感，含凭据），且使用 StringField 明文展示。

---

#### HTML 配置页关键发现总结

| 字段 | 展示策略 | 敏感程度 | 代码证据 |
|------|----------|----------|----------|
| LLM API Key | ✅ GET 清空，PasswordField | — | `changedetectionio/blueprint/settings/__init__.py#L36-L41` |
| 登录密码 | ✅ 不回显，加盐哈希 | — | `changedetectionio/forms.py#L92-L100` |
| API Access Token | ⚠️ 完整明文 + 重置 | 高 | `changedetectionio/blueprint/settings/templates/settings.html#L225` |
| RSS Access Token | ⚠️ HTML `<head>` 和链接中隐式暴露 | 高 | `changedetectionio/templates/base.html#L10-L17` |
| 通知 URL | ❌ 完整明文（TextArea） | 高 | `changedetectionio/blueprint/settings/templates/settings.html#L110` |
| 默认代理选择 | ⚠️ RadioField，仅展示名称 | 低 | `changedetectionio/forms.py#L1007` |
| 额外代理 URL | ❌ StringField，完整明文（含凭据） | 高 | `changedetectionio/forms.py#L973-L984` |

---

## 五、持久化配置暴露分析

所有敏感信息均以明文存储在服务器本地磁盘，无任何加密。

### 5.1 全局配置 `changedetection.json`
**代码位置**: `changedetectionio/store/__init__.py#L349-L389`

```python
def _build_settings_data(self):
    import copy
    settings_copy = copy.deepcopy(self.__data['settings'])  # 完整深拷贝
    settings_copy['application']['tags'] = {}  # 仅清空 tags（存独立文件）
    return {
        'note': 'Settings file - watches are in {uuid}/watch.json, tags are in {uuid}/tag.json',
        'app_guid': self.__data.get('app_guid'),
        'settings': settings_copy,  # 包含所有敏感字段
        'build_sha': self.__data.get('build_sha'),
        'version_tag': self.__data.get('version_tag')
    }

def _save_settings(self):
    settings_data = self._build_settings_data()
    changedetection_json = os.path.join(self.datastore_path, "changedetection.json")
    save_json_atomic(changedetection_json, settings_data, label="settings")
```

**包含的敏感信息（全部明文）**:
- `settings.application.api_access_token`
- `settings.application.rss_access_token`
- `settings.application.password`（加盐哈希，非明文）
- `settings.application.notification_urls[]`
- `settings.application.llm.api_key`
- `settings.requests.extra_proxies[]`（`proxy_name` + `proxy_url`，可能含凭据）
- `settings.headers`（可能含 `Authorization`）

---

### 5.2 Watch 配置 `{uuid}/watch.json`
**代码位置**: `changedetectionio/model/Watch.py#L1064-L1093`

```python
def _get_commit_data(self):
    """
    Prepare watch data for commit.
    Excludes processor_config_* keys (stored in separate files).
    Excludes __-prefixed keys (transient in-memory state).
    """
    import copy
    # ... 获取快照 ...
    watch_dict = {
        k: copy.deepcopy(v) for k, v in snapshot.items()
        if not k.startswith('processor_config_') and not k.startswith('__')
    }
    return watch_dict
```

**包含的敏感信息（全部明文）**:
- `notification_urls[]` — Watch 级通知 URL
- `headers` — Watch 级 HTTP 请求头
- `proxy` — Watch 级代理选择名（key）
- `browser_steps[]` — 可能含表单密码等

---

### 5.3 Tag 配置 `{uuid}/tag.json`
**代码位置**: `changedetectionio/model/__init__.py#L607-L627`

使用默认 `_get_commit_data()`，完整保存所有字段，无过滤。

---

### 5.4 代理配置 `proxies.json`
**代码位置**: `changedetectionio/store/__init__.py#L826-L853`

```python
@property
def proxy_list(self):
    proxy_list = {}
    proxy_list_file = os.path.join(self.datastore_path, 'proxies.json')
    if path.isfile(proxy_list_file):
        # 加载 JSON 文件 ...
    # UI 配置映射 ...
    extras = self.data['settings']['requests'].get('extra_proxies')
    if extras:
        for proxy in extras:
            if proxy.get('proxy_name') and proxy.get('proxy_url'):
                k = "ui-" + str(i) + proxy.get('proxy_name')
                proxy_list[k] = {'label': proxy.get('proxy_name'), 'url': proxy.get('proxy_url')}
    return proxy_list if len(proxy_list) else None
```

`proxies.json` 独立于主配置文件，完整保存代理 URL，可能包含认证信息。

---

### 5.5 持久化配置关键发现
1. **零加密**: 所有配置文件均为明文 JSON
2. **零脱敏**: 仅排除特定字段（tags、processor_config_、__前缀），不做任何脱敏
3. **多文件**: `changedetection.json` + `{uuid}/watch.json` + `{uuid}/tag.json` + `proxies.json`，需全部保护

---

## 六、备份文件暴露分析

备份文件是持久化配置的完整快照，可通过 Web 界面下载（需登录认证）。

### 6.1 备份创建流程
**代码位置**: `changedetectionio/blueprint/backups/__init__.py#L16-L91`

```python
def create_backup(datastore_path, watches: dict, tags: dict = None):
    with zipfile.ZipFile(...) as zipObj:
        # 1. 全局配置 - 直接复制原始文件
        changedetection_json = os.path.join(datastore_path, "changedetection.json")
        if os.path.isfile(changedetection_json):
            zipObj.write(changedetection_json, arcname="changedetection.json")
        
        # 2. Tag 数据目录 - 直接复制所有文件
        for uuid, tag in (tags or {}).items():
            for f in Path(tag.data_dir).glob('*'):
                zipObj.write(f, arcname=os.path.join(f.parts[-2], f.parts[-1]))
        
        # 3. Watch 数据目录 - 直接复制所有文件（包括历史快照）
        for uuid, w in watches.items():
            for f in Path(w.data_dir).glob('*'):
                zipObj.write(f, arcname=os.path.join(f.parts[-2], f.parts[-1]))
        
        # 4. URL 列表文件
        # ... url-list.txt / url-list-with-tags.txt ...
```

**包含的敏感信息（全部明文）**:
- `changedetection.json` — 所有系统级密钥、通知 URL、代理 URL
- `{uuid}/watch.json` — Watch 级通知 URL、headers、proxy 选择名
- `{uuid}/tag.json` — Tag 级配置
- `{uuid}/history/*` — 页面历史快照（可能含敏感业务数据）
- `proxies.json` — 如存在也会被包含吗？**需确认** — 实际上 `proxies.json` 在 datastore_path 根目录，但备份函数只显式添加了 changedetection.json，**未添加 proxies.json**！

> **重要发现**: 备份函数 `create_backup()` 没有包含 `proxies.json` 文件。这可能是一个遗漏 — 外部代理配置在备份中丢失。

---

### 6.2 备份下载保护
**代码位置**: `changedetectionio/blueprint/backups/__init__.py#L144-L166`

```python
@backups_blueprint.route("/download/<string:filename>", methods=['GET'])
@login_optionally_required
def download_backup(filename):
    # 1. 文件名白名单校验（正则匹配 changedetection-backup-\d+.zip）
    if not re.match(r"^" + backup_filename_regex + "$", filename):
        abort(400)
    # 2. 路径遍历防护（确保在 datastore_path 内）
    full_path = os.path.join(os.path.abspath(datastore.datastore_path), filename)
    if not full_path.startswith(os.path.abspath(datastore.datastore_path) + os.sep):
        abort(404)
    # 3. 发送文件
    return send_from_directory(os.path.abspath(datastore.datastore_path), filename, as_attachment=True)
```

---

### 6.3 备份文件关键发现
1. **零脱敏**: 备份是磁盘文件的完整复制，包含所有敏感信息明文
2. **下载需认证**: 仅登录用户可下载，有文件名白名单和路径遍历防护
3. **历史快照**: 备份包含 `{uuid}/history/*` 页面快照，可能包含敏感业务数据
4. **可能遗漏 `proxies.json`**: 外部代理配置文件可能未被包含在备份中（需验证）

---

## 七、统一遮蔽规则建议

### 7.1 敏感字段注册表

```python
SENSITIVE_FIELDS = frozenset({
    # 系统级密钥
    'api_access_token',
    'rss_access_token',
    'password',  # 哈希值
    # LLM
    'llm.api_key',
    # 通知
    'notification_urls',
    # 代理
    'proxy_url',         # 完整 URL（含凭据）
    'extra_proxies',     # 列表，每项含 proxy_url
    # HTTP 头
    'headers',
    # Watch 级
    'browser_steps',     # 可能含表单密码
})
```

### 7.2 URL 密码遮蔽工具函数

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

### 7.3 四层遮蔽规则

#### 规则 A：公开 API

| 字段 | 处理方式 |
|------|----------|
| `notification_urls[]`（全局 + Watch 级） | 逐个 URL 调用 `mask_url_password()` |
| `proxy`（Watch 级 key） | 保持现状（仅名称，低敏感） |
| `headers`（Watch 级） | 移除/遮蔽 `Authorization`、`Proxy-Authorization`、`Cookie`、`Set-Cookie` |
| `browser_steps[]` | 扫描并遮蔽表单中的密码字段值 |
| OpenAPI spec | 保持公开（低风险，仅文档） |
| `api_access_token` / `rss_access_token` / `llm.api_key` | 不返回（已符合，因无 settings API） |

#### 规则 B：HTML 配置页

| 字段 | 处理方式 |
|------|----------|
| `notification_urls[]` | 输入框中遮蔽显示，提交时检测是否修改 |
| `extra_proxies[].proxy_url` | **改用 PasswordField 或遮蔽显示**，避免明文在 HTML 中暴露 |
| `api_access_token` | 保持现状（完整显示 + 重置按钮） |
| `rss_access_token` | 在设置页新增展示 + 重置接口，与 API key 一致 |
| `headers` | 渲染时遮蔽敏感头值 |

#### 规则 C：持久化配置（可选增强）

| 方案 | 说明 |
|------|------|
| 方案 1（最小改动） | 保持现状，文档化安全要求 |
| 方案 2（推荐） | 新增配置加密选项，使用用户密码派生密钥加密敏感字段 |

#### 规则 D：备份文件

| 方案 | 说明 |
|------|------|
| 方案 1（最小改动） | 下载时增加警告："备份包含所有密钥，请妥善保管" |
| 方案 2（推荐） | 新增 `?safe=true` 参数，创建时自动脱敏所有敏感字段 |
| 方案 3（完整） | 备份加密（需引入密码派生 + AES-GCM） |

---

## 八、代码参考索引

> 所有路径均为仓库根目录相对路径

### 公开 API
- API 鉴权: `changedetectionio/api/auth.py#L8-L25`
- 通知 URL API: `changedetectionio/api/Notifications.py#L10-L19`
- 单 Watch API: `changedetectionio/api/Watch.py#L66-L131`
- Watch 列表 API: `changedetectionio/api/Watch.py#L538-L559`
- 系统信息 API: `changedetectionio/api/SystemInfo.py#L11-L40`
- OpenAPI Spec API: `changedetectionio/api/Spec.py#L14-L21`
- Spec 端点注册: `changedetectionio/flask_app.py#L606`
- API 字段过滤: `changedetectionio/api/__init__.py#L153-L170`
- 系统管理字段: `changedetectionio/model/schema_utils.py#L21-L32`

### Proxy 机制
- Watch 级 proxy 解析: `changedetectionio/store/__init__.py#L855-L879`
- proxy_list 属性: `changedetectionio/store/__init__.py#L826-L853`
- SingleExtraProxy 表单: `changedetectionio/forms.py#L973-L984`
- extra_proxies 字段: `changedetectionio/forms.py#L1022`
- 默认代理 RadioField: `changedetectionio/forms.py#L1007`
- 代理设置模板: `changedetectionio/blueprint/settings/templates/settings.html#L360-L363`

### HTML 配置页
- LLM API Key 表单: `changedetectionio/blueprint/settings/__init__.py#L36-L41`, `#L108-L111`
- 登录密码表单: `changedetectionio/forms.py#L92-L100`
- API Key 展示与重置: `changedetectionio/blueprint/settings/templates/settings.html#L225-L231`, `changedetectionio/blueprint/settings/__init__.py#L263-L270`
- RSS 链接渲染: `changedetectionio/templates/base.html#L10-L17`
- 通知 URL 渲染: `changedetectionio/blueprint/settings/templates/settings.html#L110`

### 持久化配置
- 全局配置保存: `changedetectionio/store/__init__.py#L349-L389`
- Watch 配置保存: `changedetectionio/model/Watch.py#L1064-L1093`
- Tag 配置保存: `changedetectionio/model/__init__.py#L607-L627`
- 代理配置加载: `changedetectionio/store/__init__.py#L826-L853`

### 备份文件
- 备份创建: `changedetectionio/blueprint/backups/__init__.py#L16-L91`
- 备份下载: `changedetectionio/blueprint/backups/__init__.py#L144-L166`

### RSS 访问令牌
- 生成与兜底: `changedetectionio/store/__init__.py#L255-L256`, `#L303-L313`
- 登录豁免: `changedetectionio/flask_app.py#L555-L557`
- 访问校验: `changedetectionio/blueprint/rss/_util.py#L55-L68`
- 模板传递: `changedetectionio/blueprint/watchlist/__init__.py#L94`, `changedetectionio/blueprint/ui/edit.py#L312-L337`, `changedetectionio/blueprint/tags/__init__.py#L28`

### 安全测试
- LLM API Key 专项测试: `changedetectionio/tests/test_llm_api_key_security.py`
- API 内部字段过滤测试: `changedetectionio/tests/test_api.py#L409-L457`

---

## 九、总结

### ✅ 现有优势
1. LLM API Key 有完善的专项保护（GET 清空 + 11 端点金丝雀测试 + SSRF 防护 + CSRF 凭据泄露防护）
2. 登录密码使用加盐哈希存储，Web 表单不回显
3. 无全局 Settings API 端点 → `api_access_token`、`rss_access_token`、`llm.api_key` 等系统级密钥不通过 API 直接暴露
4. API 有 `strip_internal_api_fields()` 过滤内部/临时字段和 LLM 运行时状态
5. 备份下载有登录认证 + 文件名白名单 + 路径遍历防护
6. Watch 级 `proxy` 字段仅存选择名，不直接暴露完整 URL 凭据

### ❌ 主要缺失（按优先级）

| 优先级 | 缺失项 | 影响范围 | 代码证据 |
|--------|--------|----------|----------|
| 🔴 P0 | 通知 URL 密码无遮蔽（API + HTML + 日志） | 通知服务凭据泄露 | `changedetectionio/api/Notifications.py#L10-L19` |
| 🔴 P0 | 备份文件全部敏感信息明文 | 备份文件泄露即全盘泄露 | `changedetectionio/blueprint/backups/__init__.py#L16-L91` |
| 🔴 P0 | RSS 令牌无重置接口 + HTML 隐式暴露 | 令牌泄露后无法快速轮换 | `changedetectionio/templates/base.html#L10-L17` |
| 🟡 P1 | `extra_proxies[].proxy_url` 使用 StringField 明文展示 | 代理凭据在 HTML 中明文可见 | `changedetectionio/forms.py#L973-L984` |
| 🟡 P1 | Watch 级 `notification_urls`、`headers` 在 API 中明文 | 单 Watch 配置泄露 | `changedetectionio/api/Watch.py#L131` |
| 🟡 P1 | HTTP 认证头无遮蔽 | `Authorization` 等头泄露 | `changedetectionio/api/Watch.py#L131` |
| 🟡 P1 | OpenAPI spec 完全公开 | 帮助攻击者了解 API 结构 | `changedetectionio/api/Spec.py#L14-L21` |
| 🟢 P2 | 持久化配置无加密 | 磁盘层面明文存储 | `changedetectionio/store/__init__.py#L349-L389` |
| 🟢 P2 | 备份可能遗漏 `proxies.json` | 外部代理配置备份丢失 | `changedetectionio/blueprint/backups/__init__.py#L16-L91` |
| 🟢 P2 | 缺少统一的敏感字段管理工具类 | 未来易新增泄露点 | — |

### 暴露范围区分
- **完全公开（无需任何认证）**: OpenAPI spec
- **对外暴露（API + HTML，需认证）**: `notification_urls`（全局 + Watch 级）、Watch 级 `headers`、`api_access_token`（HTML 明文）、`rss_access_token`（HTML 隐式）、Watch 级 `proxy` 选择名
- **仅内部（持久化 + 备份）**: `llm.api_key`、`password`（哈希）、全局 `headers`/`extra_proxies`、完整代理 URL（API 不直接暴露）
- **LLM API Key 例外**: HTML GET 时清空，API 不暴露，仅持久化和备份中明文
