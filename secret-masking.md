# 敏感信息遮蔽策略分析报告

## 一、概述

本报告对 changedetection.io 项目中密码与敏感凭据在 **公开API、HTML配置页、持久化配置、备份文件** 四个层面的处理方式进行代码级审计，区分哪些信息对外暴露（API/HTML）、哪些仅保留在配置或备份中。

---

## 二、四层暴露矩阵总览

| 敏感字段 | 公开API（需Token） | HTML配置页（需登录） | 持久化配置（本地磁盘） | 备份文件（可下载） |
|----------|-------------------|---------------------|----------------------|-------------------|
| `llm.api_key` | ❌ 不暴露 | ✅ GET时清空（PasswordField） | ✅ 明文 | ✅ 明文 |
| `password` | ❌ 不暴露 | ✅ 不回显（SaltyPasswordField，存储为哈希） | ✅ 加盐哈希 | ✅ 加盐哈希 |
| `api_access_token` | ❌ 不暴露（无settings端点） | ✅ 完整明文展示 | ✅ 明文 | ✅ 明文 |
| `rss_access_token` | ❌ 不暴露（无settings端点） | ⚠️ 仅在HTML `<head>` 和 RSS 链接中明文 | ✅ 明文 | ✅ 明文 |
| `notification_urls[]`（全局） | ⚠️ 明文（`GET /api/v1/notifications`） | ✅ 完整明文（TextArea） | ✅ 明文 | ✅ 明文 |
| `notification_urls[]`（Watch级） | ⚠️ 明文（`GET /api/v1/watch/<uuid>`） | ✅ 完整明文（Watch编辑页） | ✅ 明文 | ✅ 明文 |
| `extra_proxies[].proxy_url` | ❌ 不暴露 | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |
| `proxies.json` 代理 | ❌ 不暴露 | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |
| `headers`（全局） | ❌ 不暴露 | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |
| `headers`（Watch级） | ⚠️ 明文（`GET /api/v1/watch/<uuid>`） | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |
| `proxy`（Watch级） | ⚠️ 明文（`GET /api/v1/watch/<uuid>`） | ✅ 完整明文 | ✅ 明文 | ✅ 明文 |

> **图例**: ✅ 明文 / ⚠️ 部分暴露 / ❌ 不暴露

---

## 三、公开 API 暴露分析

公开 API 均需通过 `@auth.check_token` 装饰器校验 `api_access_token`，但通过认证后，部分敏感信息以明文返回。

### 3.1 关键端点返回内容

#### `GET /api/v1/notifications` — 通知 URL 明文返回
**代码位置**: `changedetectionio/api/Notifications.py#L10-L19`

```python
def get(self):
    """Return Notification URL List."""
    notification_urls = self.datastore.data.get('settings', {}).get('application', {}).get('notification_urls', [])        
    return {
        'notification_urls': notification_urls,
    }, 200
```
**风险**: Apprise URL 通常包含明文密码，如 `mailto://user:pass@smtp.example.com`、`slack://xoxb-TOKEN@CHANNEL`。

---

#### `GET /api/v1/watch/<uuid>` — 单 Watch 完整配置
**代码位置**: `changedetectionio/api/Watch.py#L66-L131`

```python
def get(self, uuid):
    # ... 操作逻辑 ...
    with self.datastore.lock:
        watch = dict(watch_obj)
    # ... 添加额外字段 ...
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

**返回内容（部分敏感字段明文）**:
- `notification_urls[]` — Watch 级通知 URL
- `headers` — Watch 级 HTTP 请求头（可能含 `Authorization`）
- `proxy` — Watch 级代理配置
- **不包含**: `api_access_token`、`rss_access_token`、`llm.api_key`（这些在全局 settings 中）

**Watch 列表端点 `GET /api/v1/watch`** — `changedetectionio/api/Watch.py#L538-L559`，仅返回简化字段：
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
不包含 `notification_urls`、`headers`、`proxy` 等敏感配置。

---

#### `GET /api/v1/systeminfo` — 系统信息
**代码位置**: `changedetectionio/api/SystemInfo.py#L11-L40`

仅返回非敏感信息：
```python
return {
    'queue_size': self.update_q.qsize(),
    'overdue_watches': overdue_watches,
    'uptime': round(time.time() - self.datastore.start_time, 2),
    'watch_count': len(self.datastore.data.get('watching', {})),
    'version': main_version
}, 200
```

---

#### API 关键发现
1. **无全局 Settings 端点**: 没有 API 端点直接返回 `settings.application`，因此 `api_access_token`、`rss_access_token`、`llm.api_key` 等系统级密钥**不通过 API 暴露**。
2. **Watch 级信息暴露**: `GET /api/v1/watch/<uuid>` 返回 Watch 的完整配置，包括 `notification_urls`、`headers`、`proxy`，仅过滤 `__` 前缀和 LLM 运行时状态。
3. **通知 URL 专项暴露**: `GET /api/v1/notifications` 直接返回全局通知 URL 列表，无任何遮蔽。

---

## 四、HTML 配置页暴露分析

HTML 页面需通过 Flask-Login 登录认证（或密码保护），敏感信息展示策略不一致。

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
# PasswordField never re-renders, so a blank submitted value means
# "keep stored key" — drop it from the merge.
if not (llm_form_input.get('api_key') or '').strip():
    llm_form_input.pop('api_key', None)
```

**模板渲染** — `changedetectionio/blueprint/settings/templates/settings_llm_tab.html#L128`:
```jinja2
{{ render_field(form.llm.form.api_key) }}
```
> WTForms `PasswordField` 不会在 HTML 中渲染 `value` 属性，即使传值也不会显示。

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
**代码位置**: `changedetectionio/blueprint/settings/__init__.py#L237`（传值）+ `changedetectionio/blueprint/settings/templates/settings.html#L225`（渲染）

```python
# 视图传值
'api_key': datastore.data['settings']['application'].get('api_access_token'),

# 模板渲染
<span id="api-key">{{api_key}}</span>
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

**模板传递链路** — 视图函数注入 `app_rss_token`：
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

#### 代理 URL — 完整明文
**代码位置**: `changedetectionio/blueprint/settings/templates/settings.html#L99`

```jinja2
{{ render_field(form.requests.form.proxy, class="fetch-backend-proxy") }}
```
代理配置完整明文渲染。

---

#### HTML 配置页关键发现
| 字段 | 展示策略 | 代码证据 |
|------|----------|----------|
| LLM API Key | ✅ GET 清空，PasswordField | `changedetectionio/blueprint/settings/__init__.py#L36-L41` |
| 登录密码 | ✅ 不回显，加盐哈希存储 | `changedetectionio/forms.py#L92-L100` |
| API Access Token | ⚠️ 完整明文展示 + 重置按钮 | `changedetectionio/blueprint/settings/templates/settings.html#L225` |
| RSS Access Token | ⚠️ 仅在 HTML `<head>` 和链接中隐式暴露 | `changedetectionio/templates/base.html#L10-L17` |
| 通知 URL | ❌ 完整明文 | `changedetectionio/blueprint/settings/templates/settings.html#L110` |
| 代理 URL | ❌ 完整明文 | `changedetectionio/blueprint/settings/templates/settings.html#L99` |

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

**`changedetection.json` 包含的敏感信息（全部明文）**:
- `settings.application.api_access_token`
- `settings.application.rss_access_token`
- `settings.application.password`（加盐哈希，非明文）
- `settings.application.notification_urls[]`
- `settings.application.llm.api_key`
- `settings.requests.extra_proxies[]`（代理 URL，可能含密码）
- `settings.headers`（HTTP 头，可能含 `Authorization`）

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
    # ... 归一化 browser_steps ...
    return watch_dict
```

**`{uuid}/watch.json` 包含的敏感信息（全部明文）**:
- `notification_urls[]` — Watch 级通知 URL
- `headers` — Watch 级 HTTP 请求头
- `proxy` — Watch 级代理配置

---

### 5.3 Tag 配置 `{uuid}/tag.json`
**代码位置**: `changedetectionio/model/__init__.py#L607-L627`

```python
def _get_commit_data(self):
    """Prepare data for commit (can be overridden by subclasses)."""
    import copy
    # ... 获取快照 ...
    return {k: copy.deepcopy(v) for k, v in snapshot.items()}  # 完整保存所有字段
```

Tag 使用默认 `_get_commit_data()`，无字段过滤。

---

### 5.4 代理配置 `proxies.json`
**代码位置**: `changedetectionio/store/__init__.py#L828-L838`

```python
@property
def proxy_list(self):
    proxy_list = {}
    proxy_list_file = os.path.join(self.datastore_path, 'proxies.json')
    if path.isfile(proxy_list_file):
        if HAS_ORJSON:
            with open(os.path.join(self.datastore_path, "proxies.json"), 'rb') as f:
                proxy_list = orjson.loads(f.read())
        else:
            with open(os.path.join(self.datastore_path, "proxies.json"), encoding='utf-8') as f:
                proxy_list = json.load(f)
    # ... UI 配置映射 ...
    return proxy_list if len(proxy_list) else None
```

`proxies.json` 完整保存代理 URL，可能包含认证信息如 `http://user:pass@proxy.example.com:8080`。

---

### 5.5 持久化配置关键发现
1. **无加密**: 所有配置文件均为明文 JSON，磁盘层面无任何加密保护。
2. **无脱敏**: `_build_settings_data()` 和 `_get_commit_data()` 仅做字段排除（tags、processor_config_、__前缀），不做任何脱敏处理。
3. **独立文件**: `proxies.json` 独立于主配置，需单独保护。

---

## 六、备份文件暴露分析

备份文件是持久化配置的完整快照，可通过 Web 界面下载（需登录认证）。

### 6.1 备份创建流程
**代码位置**: `changedetectionio/blueprint/backups/__init__.py#L16-L91`

```python
def create_backup(datastore_path, watches: dict, tags: dict = None):
    # ... 创建 ZIP ...
    with zipfile.ZipFile(backup_filepath.replace('.zip', '.tmp'), "w", ...) as zipObj:
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
        
        # 4. URL 列表
        # ... 生成 url-list.txt 和 url-list-with-tags.txt ...
    # ... 重命名完成 ...
```

### 6.2 备份下载保护
**代码位置**: `changedetectionio/blueprint/backups/__init__.py#L144-L166`

```python
@backups_blueprint.route("/download/<string:filename>", methods=['GET'])
@login_optionally_required
def download_backup(filename):
    # 1. 文件名白名单校验（正则匹配）
    if not re.match(r"^" + backup_filename_regex + "$", filename):
        abort(400)
    # 2. 路径遍历防护（确保在 datastore_path 内）
    full_path = os.path.join(os.path.abspath(datastore.datastore_path), filename)
    if not full_path.startswith(os.path.abspath(datastore.datastore_path) + os.sep):
        abort(404)
    # 3. 发送文件
    return send_from_directory(os.path.abspath(datastore.datastore_path), filename, as_attachment=True)
```

### 6.3 备份文件关键发现
1. **零脱敏**: 备份是磁盘文件的完整复制，包含所有敏感信息明文。
2. **下载需认证**: 仅登录用户可下载，有文件名白名单和路径遍历防护。
3. **历史快照**: 备份包含 `{uuid}/history/*` 页面快照，可能包含敏感业务数据。
4. **无"安全备份"选项**: 无法导出不含敏感信息的配置用于分享。

---

## 七、统一遮蔽规则建议

### 7.1 敏感字段注册表

```python
SENSITIVE_FIELDS = frozenset({
    # 系统级密钥
    'api_access_token',
    'rss_access_token',
    'password',  # 哈希值，不暴露明文
    # LLM
    'llm.api_key',
    # 通知
    'notification_urls',
    # 代理
    'proxy_url',
    'extra_proxies',
    # HTTP 头
    'headers',
    'proxy',  # Watch 级代理
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
| `extra_proxies[].proxy_url` | 调用 `mask_url_password()`（若未来新增 settings API） |
| `headers`（全局 + Watch 级） | 移除 `Authorization`、`Proxy-Authorization`、`Cookie`、`Set-Cookie` |
| `proxy`（Watch 级） | 若是 URL 格式，调用 `mask_url_password()` |
| `api_access_token` | 只显示后 4 位，如 `...abcd`（若未来新增 settings API） |
| `rss_access_token` | 只显示后 4 位，如 `...abcd`（若未来新增 settings API） |
| `llm.api_key` | **不返回该字段**（已符合，因无 settings API） |
| `password` | **不返回该字段**（已符合） |

#### 规则 B：HTML 配置页
| 字段 | 处理方式 |
|------|----------|
| `notification_urls[]` | 输入框中遮蔽显示，提交时检测是否修改 |
| `extra_proxies[].proxy_url` | 同上 |
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
- 通知 URL API: `changedetectionio/api/Notifications.py#L10-L19`
- 单 Watch API: `changedetectionio/api/Watch.py#L66-L131`
- Watch 列表 API: `changedetectionio/api/Watch.py#L538-L559`
- 系统信息 API: `changedetectionio/api/SystemInfo.py#L11-L40`
- API 字段过滤: `changedetectionio/api/__init__.py#L153-L170`
- 系统管理字段: `changedetectionio/model/schema_utils.py#L21-L32`

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
3. 无全局 Settings API 端点，`api_access_token`、`rss_access_token`、`llm.api_key` 等系统级密钥**不通过 API 暴露**
4. API 有 `strip_internal_api_fields()` 过滤内部/临时字段和 LLM 运行时状态
5. 备份下载有登录认证 + 文件名白名单 + 路径遍历防护

### ❌ 主要缺失（按优先级）
| 优先级 | 缺失项 | 影响范围 | 代码证据 |
|--------|--------|----------|----------|
| 🔴 P0 | 通知 URL 密码无遮蔽（API + HTML + 日志） | 通知服务凭据泄露 | `changedetectionio/api/Notifications.py#L10-L19` |
| 🔴 P0 | RSS 令牌无重置接口 + HTML 隐式暴露 | 令牌泄露后无法快速轮换 | `changedetectionio/templates/base.html#L10-L17` |
| 🔴 P0 | 备份文件全部敏感信息明文 | 备份文件泄露即全盘泄露 | `changedetectionio/blueprint/backups/__init__.py#L16-L91` |
| 🟡 P1 | Watch 级 `notification_urls`、`headers`、`proxy` 在 API 中明文 | 单 Watch 配置泄露 | `changedetectionio/api/Watch.py#L131` |
| 🟡 P1 | 代理 URL 密码无遮蔽 | 代理服务凭据泄露 | `changedetectionio/store/__init__.py#L826-L853` |
| 🟡 P1 | HTTP 认证头无遮蔽 | `Authorization` 等头泄露 | `changedetectionio/api/Watch.py#L131` |
| 🟢 P2 | 持久化配置无加密 | 磁盘层面明文存储 | `changedetectionio/store/__init__.py#L349-L389` |
| 🟢 P2 | 缺少统一的敏感字段管理工具类 | 未来易新增泄露点 | — |

### 暴露范围区分
- **对外暴露（API + HTML）**: `notification_urls`（全局 + Watch 级）、Watch 级 `headers`/`proxy`、`api_access_token`（HTML 明文）、`rss_access_token`（HTML 隐式）
- **仅内部（持久化 + 备份）**: `llm.api_key`、`password`（哈希）、全局 `headers`/`extra_proxies`（无 API 端点暴露，但备份包含）
- **LLM API Key 例外**: HTML GET 时清空，API 不暴露，仅持久化和备份中明文
