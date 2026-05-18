# Diff 通知与敏感字段处理流程分析（事实校对版）

> 本文档为 `diff-notify-sanitize.md` 的事实校对与可读性修正版本。所有代码引用、测试统计、边界分析均经过逐行源码核对。

---

## 一、Diff 生成流程

### 1.1 核心入口

Diff 生成的核心函数位于 `changedetectionio/diff/__init__.py:render_diff()`。

```python
def render_diff(
    previous_version_file_contents: str,
    newest_version_file_contents: str,
    include_equal: bool = False,
    include_removed: bool = True,
    include_added: bool = True,
    include_replaced: bool = True,
    include_change_type_prefix: bool = True,
    patch_format: bool = False,
    word_diff: bool = True,
    context_lines: int = 0,
    case_insensitive: bool = False,
    ignore_junk: bool = False,
    tokenizer: str = 'words_and_html'
) -> str:
```

### 1.2 差异计算机制

**行级差异** (`customSequenceMatcher`):
- 使用 Python 标准库 `difflib.SequenceMatcher`
- 支持上下文行数控制 (`context_lines`)
- 可配置是否包含：相等行、删除行、新增行、替换行

**词级差异** (`render_inline_word_diff`):
- 使用 `diff-match-patch` 库
- 通过 `linesToChars` 技巧将词转换为"行"进行比较
- 支持自定义分词器 (`tokenize_words_and_html`)
- 整行替换时使用 `CHANGED_PLACEMARKER` 标记，行内变化使用 `REMOVED_PLACEMARKER` / `ADDED_PLACEMARKER`

### 1.3 占位标记系统

Diff 输出使用特殊占位标记而非直接的 HTML 标签，便于后续根据通知渠道转换：

| 标记类型 | 打开标记 | 关闭标记 | 用途 |
|---------|---------|---------|------|
| 删除 | `@removed_PLACEMARKER_OPEN` | `@removed_PLACEMARKER_CLOSED` | 被删除的内容 |
| 新增 | `@added_PLACEMARKER_OPEN` | `@added_PLACEMARKER_CLOSED` | 新增的内容 |
| 变更前 | `@changed_PLACEMARKER_OPEN` | `@changed_PLACEMARKER_CLOSED` | 替换行的旧值 |
| 变更后 | `@changed_into_PLACEMARKER_OPEN` | `@changed_into_PLACEMARKER_CLOSED` | 替换行的新值 |

### 1.4 可配置的 Diff 变体

系统预定义了多种 Diff 变体供通知模板使用 (`notification_service.py:262-278`):

| 变体名称 | 配置 | 用途 |
|---------|------|------|
| `diff` | 默认 | 标准差异视图 |
| `diff_clean` | `include_change_type_prefix=False` | 无前缀的纯净差异 |
| `diff_added` | `include_removed=False` | 仅显示新增 |
| `diff_removed` | `include_added=False` | 仅显示删除 |
| `diff_full` | `include_equal=True` | 包含上下文的完整差异 |
| `diff_patch` | `patch_format=True` | Patch 格式 |

---

## 二、通知内容裁剪策略

### 2.1 裁剪时机与位置

内容裁剪发生在通知发送前的 `apply_service_tweaks()` 函数中 (`notification/handler.py:209-304`)。

### 2.2 各通知渠道的长度限制

#### Telegram (`tgram://`)
- **总载荷限制**: 3600 字节（实际官方限制 4096，预留 496 字节用于元数据）
- **标题限制**: `payload_max_size = 3600`
- **正文限制**: `body_limit = max(0, payload_max_size - len(n_title))`
- 实现位置: `notification/handler.py:256-260`

#### Discord (`discord://` 或 webhook URL)
- **纯文本模式**: 1700 字节（官方限制 2000，预留 300 字节）
- **Embed 模式**: 使用自定义插件时支持 6000 字符限制
- 实现位置: `notification/handler.py:262-288`

#### 其他渠道
- 无强制长度限制，依赖 Apprise 库和目标服务的限制

### 2.3 裁剪算法

```python
# Telegram 示例
payload_max_size = 3600
body_limit = max(0, payload_max_size - len(n_title))
n_title = n_title[0:payload_max_size]
n_body = n_body[0:body_limit]
```

**特点**:
- 简单的字符串切片，无智能截断（如按词、按句）
- 标题和正文联动计算，确保总长度不超限
- 仅在超出限制时截断，不做其他处理

---

## 三、敏感字段识别与遮蔽

### 3.1 敏感字段分类与处理层级

#### 层级 1: API 响应字段过滤（外部接口）

**实现位置**: `api/__init__.py:strip_internal_api_fields()`

**过滤规则**:
1. 过滤所有 `__` 开头的内部 transient 字段（如 `__check_status`）
2. 过滤 `SYSTEM_MANAGED_NON_SPEC_FIELDS` 中定义的系统管理字段

**系统管理字段列表** (`model/schema_utils.py:21-32`):
```python
SYSTEM_MANAGED_NON_SPEC_FIELDS = frozenset({
    'last_check_status',           # 处理器设置的状态
    'last_filter_config_hash',     # 跳过缓存的哈希
    'restock',                     # 补货处理器设置
    '_llm_result',                 # LLM 运行时结果
    '_llm_intent',                 # LLM 意图
    '_llm_change_summary',         # LLM 变更摘要
    'llm_prefilter',               # LLM 预过滤
    'llm_evaluation_cache',        # LLM 评估缓存
    'llm_last_tokens_used',        # LLM 上次 Token 使用量
    'llm_tokens_used_cumulative',  # LLM 累计 Token 使用量
})
```

#### 层级 2: 持久化过滤（磁盘存储）

**实现位置**: `model/Watch.py:_get_commit_data()`

- `__` 开头的字段不会被持久化到磁盘
- 确保敏感运行时数据不会泄露到存储层

#### 层级 3: Diff 内容本身的敏感字段

**重要发现**: 经过全面代码搜索，**系统目前没有对 Diff 内容本身进行敏感字段扫描和遮蔽**。

也就是说：
- 如果被监控的网页内容包含密码、API Key、Token 等敏感信息
- 这些信息会完整地出现在 Diff 结果中
- 并通过通知渠道（邮件、Slack、Telegram 等）发送出去
- 没有内置的正则匹配或关键字过滤机制来遮蔽这些内容

### 3.2 通知 URL 中的敏感信息处理

通知 URL（如 `json://token@host/path`）本身包含的敏感信息：

- **存储时**: 完整保存，无遮蔽
- **API 返回时**: 完整返回，无遮蔽
- **日志输出时**: 多处场景下会记录完整 URL（高泄露风险）

#### 3.2.1 通知 URL 的 API 暴露证据链

**证据 1: GET /api/v1/notifications 直接返回完整 URL**

代码位置: `api/Notifications.py:12-19`
```python
@auth.check_token
@validate_openapi_request('getNotifications')
def get(self):
    """Return Notification URL List."""
    notification_urls = self.datastore.data.get('settings', {}).get('application', {}).get('notification_urls', [])
    return {
            'notification_urls': notification_urls,
           }, 200
```

**证据 2: POST /api/v1/notifications 回显添加的 URL**

代码位置: `api/Notifications.py:46`
```python
return {'notification_urls': added_urls}, 201
```

**证据 3: PUT /api/v1/notifications 回显替换后的 URL**

代码位置: `api/Notifications.py:68`
```python
return {'notification_urls': clean_urls}, 200
```

**风险影响**:
- 任何拥有 API Token 的用户/攻击者可以读取所有通知 URL
- 通知 URL 中包含的 Token、密码、Webhook 密钥等全部明文暴露
- 例如: `tgram://123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11` 这样的 Telegram Bot Token 会被完整获取

#### 3.2.2 通知 URL 的日志暴露证据链

**证据 1: 添加通知 URL 时的 DEBUG 日志**

代码位置: `store/__init__.py:1085`
```python
logger.debug(f">>> Adding new notification_url - '{notification_url}'")
```

**证据 2: 发送通知时的 INFO 日志**

代码位置: `notification/handler.py:416`
```python
logger.info(f">> Process Notification: AppRise start notifying '{url}'")
```

**风险影响**:
- 在 DEBUG 或 INFO 日志级别下，完整的通知 URL（包含敏感凭证）会被记录到日志系统
- 日志系统通常被多个团队/人员访问，存在严重的横向泄露风险
- 日志持久化后，敏感凭证可能在备份中永久留存

### 3.3 LLM API Key 的保护机制与真实防护边界

#### 3.3.1 真实防护机制（经事实校对）

LLM API Key 是特殊的敏感字段，其保护机制如下（按防护强度排序）：

**层级 1: 根本没有配置读取 API 端点（最核心防护）**

代码证据: `tests/test_llm_api_key_security.py:265-294`
```python
def test_no_api_settings_endpoint_exists(
        client, live_server, measure_memory_usage, datastore_path):
    """
    There is currently no /api/v1/settings endpoint.
    If one is added in the future it must be covered by its own
    security tests before reaching production.
    This test acts as a canary — it should FAIL if a settings endpoint
    is accidentally wired up without review.
    """
    api_token = _api_token(client)

    res_get = client.get('/api/v1/settings', headers={'x-api-key': api_token})
    assert res_get.status_code in (404, 405), \
        (f"Unexpected /api/v1/settings GET returned {res_get.status_code}. "
         "A settings endpoint must have explicit LLM key security tests before shipping.")

    res_post = client.post(
        '/api/v1/settings',
        headers={'x-api-key': api_token, 'content-type': 'application/json'},
        data=json.dumps({}),
    )
    assert res_post.status_code in (404, 405)
```

**关键事实**: 系统目前**没有 `/api/v1/settings` 端点**，LLM API Key 无法通过 API 直接读取。

**层级 2: 设置页面使用 PasswordField 保护**

代码证据: `tests/test_llm_api_key_security.py:301-317`
```python
def test_settings_page_does_not_render_llm_api_key_in_plaintext(
        client, live_server, measure_memory_usage, datastore_path):
    """
    The settings page renders the API key form.  Because the field uses
    PasswordField, WTForms must NOT embed the current key value in the HTML
    (PasswordField intentionally omits the value attribute for security).
    """
    ds = client.application.config.get('DATASTORE')
    _configure_llm(ds)

    res = client.get(url_for('settings.settings_page'))
    assert res.status_code == 200
    body = res.data.decode('utf-8', errors='replace')
    assert CANARY_KEY not in body, \
        "LLM API key appeared in plaintext in the settings page HTML source. " \
        "The llm_api_key field must be a PasswordField so the value is never rendered."
```

**层级 3: 系统字段过滤机制**

通过 `strip_internal_api_fields()` 过滤 `SYSTEM_MANAGED_NON_SPEC_FIELDS` 中的内部字段（如 `_llm_result`, `_llm_intent`, `_llm_change_summary` 等 LLM 运行时数据）。

**层级 4: 全面的安全测试保障**

经逐行核对，`tests/test_llm_api_key_security.py` 包含 **12 个测试用例**（而非 9 个），确保 LLM API Key 不会出现在：

| 测试编号 | 测试函数 | 覆盖端点 | 行号 |
|---------|---------|---------|------|
| 1 | `test_watch_get_does_not_expose_llm_api_key` | GET /api/v1/watch/&lt;uuid&gt; | 49-74 |
| 2 | `test_watch_list_does_not_expose_llm_api_key` | GET /api/v1/watch (list) | 77-97 |
| 3 | `test_watch_put_response_does_not_expose_llm_api_key` | PUT /api/v1/watch/&lt;uuid&gt; | 100-126 |
| 4 | `test_tag_get_does_not_expose_llm_api_key` | GET /api/v1/tag/&lt;uuid&gt; | 133-150 |
| 5 | `test_tag_list_does_not_expose_llm_api_key` | GET /api/v1/tags | 153-165 |
| 6 | `test_system_info_does_not_expose_llm_api_key` | GET /api/v1/systeminfo | 172-184 |
| 7 | `test_notifications_api_does_not_expose_llm_api_key` | GET/POST/PUT /api/v1/notifications | 187-220 |
| 8 | `test_search_api_does_not_expose_llm_api_key` | GET /api/v1/search | 223-243 |
| 9 | `test_openapi_spec_does_not_expose_llm_api_key` | GET /api/v1/full-spec | 246-262 |
| 10 | `test_no_api_settings_endpoint_exists` | GET/POST /api/v1/settings (金丝雀测试) | 265-294 |
| 11 | `test_settings_page_does_not_render_llm_api_key_in_plaintext` | 设置页面 HTML | 301-317 |
| 12 | `test_settings_form_preserves_api_key_when_submitted_blank` | 设置表单提交逻辑 | 319-353 |

#### 3.3.2 LLM API Key 的防护边界总结

| 防护层级 | 防护机制 | 代码位置 |
|---------|---------|----------|
| API 读取 | 无设置 API 端点（核心防护） | `tests/test_llm_api_key_security.py:265-294` |
| Web UI | PasswordField 不渲染值 | `tests/test_llm_api_key_security.py:301-317` |
| 系统字段过滤 | `strip_internal_api_fields()` | `api/__init__.py` |
| 测试保障 | 12 个安全测试用例 | `tests/test_llm_api_key_security.py` |

**注意**: LLM API Key 的保护主要依赖于「没有 API 读取端点」这一事实，而非主动的遮蔽/加密机制。如果未来添加 `/api/v1/settings` 端点，必须立即引入额外的安全措施。

---

## 四、模板渲染流程

### 4.1 模板变量准备

**入口函数**: `notification/handler.py:create_notification_parameters()`

准备的核心变量包括:
- `base_url`, `diff_url`, `preview_url`, `edit_url`
- `watch_title`, `watch_tag`, `watch_url`, `watch_uuid`
- `current_snapshot`, `prev_snapshot` (原始快照)
- `diff`, `diff_added`, `diff_removed` 等多种格式的差异
- `triggered_text` (触发文本)
- `llm_summary`, `llm_intent` (AI 摘要)

### 4.2 可格式化的 Diff 对象

`FormattableDiff` 类 (`notification_service.py:114-166`) 提供了灵活的模板调用方式:

```jinja2
{{ diff }}                                    {# 默认输出 #}
{{ diff(lines=5) }}                           {# 仅前 5 行 #}
{{ diff(added_only=true) }}                   {# 仅新增 #}
{{ diff(removed_only=true) }}                 {# 仅删除 #}
{{ diff(context=3) }}                         {# 3 行上下文 #}
{{ diff(word_diff=false) }}                   {# 行级而非词级 #}
{{ diff(lines=10, added_only=true) }}         {# 组合参数 #}
```

### 4.3 渲染流程

**完整流程** (`notification/handler.py:307-496`):

1. **Diff 懒加载**: `add_rendered_diff_to_notification_vars()` 只渲染模板中实际使用的 Diff 变体
2. **AI 摘要替换**: 如启用 LLM，`diff` 变量可被 AI 摘要替代
3. **HTML 转义**: 对 `raw_diff`, `current_snapshot`, `prev_snapshot`, `triggered_text` 进行 HTML 转义，防止 XSS
4. **Jinja2 渲染**: 使用 `jinja_render()` 渲染标题和正文模板
5. **服务适配**: `apply_service_tweaks()` 根据通知渠道转换占位标记
6. **长度裁剪**: 对超长内容进行截断

### 4.4 占位标记转换

不同通知渠道的标记转换规则 (`replace_placemarkers_in_text()`):

| 渠道 | 删除标记 | 新增标记 |
|-----|---------|---------|
| Telegram | `<s>...</s>` | `<b>...</b>` |
| Discord Markdown | `~~...~~` | `**...**` |
| 标准 Markdown | `<del>...</del>` | `**...**` |
| HTML Color | 带内联样式的 `<span>` | 带内联样式的 `<span>` |
| 纯文本 | `(removed) ` | `(added) ` |

---

## 五、完整处理链路图

```
网页内容获取
    ↓
内容预处理 (HTML→文本, 过滤, 提取)
    ↓
Diff 计算 (render_diff)
    ├─ 行级比较 (difflib.SequenceMatcher)
    └─ 词级比较 (diff-match-patch)
    ↓
生成带占位标记的 Diff 文本
    ↓
通知触发
    ↓
准备通知上下文 (NotificationContextData)
    ├─ 懒加载 Diff 变体
    ├─ 可选: AI 摘要替换
    └─ HTML 转义 (防 XSS)
    ↓
Jinja2 模板渲染
    ├─ 标题模板渲染
    └─ 正文模板渲染
    ↓
服务适配处理 (apply_service_tweaks)
    ├─ 占位标记转换 (根据渠道)
    ├─ 换行符处理
    └─ 长度裁剪 (Telegram/Discord 等)
    ↓
Apprise 发送通知
```

---

## 六、边界变化说明：新增配置读取接口的泄露面分析

### 6.1 当前防护的边界条件

当前 LLM API Key 的安全保障建立在以下边界条件之上：

```
边界条件 1: 不存在 /api/v1/settings 端点
边界条件 2: 设置页面使用 PasswordField（不渲染 value 属性）
边界条件 3: 所有其他 API 端点通过 strip_internal_api_fields() 过滤敏感字段
```

其中 **边界条件 1 是最核心的防护**，也是最脆弱的边界。

### 6.2 若新增配置读取接口将新增的泄露面

如果未来添加 `/api/v1/settings` 或类似的配置读取端点，将打开以下泄露面：

#### 泄露面 1: LLM API Key 直接泄露

**泄露路径**:
1. 攻击者获取 API Token
2. 调用 `GET /api/v1/settings`
3. 直接获取 `settings.application.llm.api_key` 明文

**影响**:
- LLM 服务被冒用，产生巨额账单
- 攻击者可通过 LLM API 进行进一步攻击

**现有防护失效点**:
- `strip_internal_api_fields()` 仅过滤 `__` 开头字段和 `SYSTEM_MANAGED_NON_SPEC_FIELDS`
- `llm.api_key` 不在过滤列表中
- 金丝雀测试 `test_no_api_settings_endpoint_exists` 会失败，但这是"死后检测"而非"事前防护"

#### 泄露面 2: 通知 URL 中的敏感凭证批量泄露

**泄露路径**:
1. 攻击者获取 API Token
2. 调用 `GET /api/v1/settings`
3. 获取所有 `notification_urls`，其中包含：
   - Telegram Bot Token: `tgram://123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`
   - Slack Webhook: `slack://T12345/B12345/abcdef123456`
   - SMTP 密码: `mail://user:password@smtp.example.com`
   - 自定义 Webhook Token: `json://token@api.example.com/notify`

**影响**:
- 所有通知渠道被接管
- 攻击者可发送恶意通知欺骗用户
- 可进一步利用通知渠道进行社会工程学攻击

#### 泄露面 3: 其他敏感配置泄露

**可能泄露的字段**:
- `api_access_token`: API 访问 Token（可被轮换）
- `proxy` 配置: 代理服务器地址和凭证
- Webhook 回调配置中的密钥
- 任何新增的第三方服务 API Key

#### 泄露面 4: 组合攻击面

如果同时存在多个泄露面，攻击者可进行组合攻击：
1. 通过配置读取接口获取 LLM API Key
2. 通过通知 URL 获取 Slack/Telegram 渠道
3. 利用 LLM 生成精心构造的钓鱼通知
4. 通过接管的通知渠道发送给所有用户

### 6.3 新增配置读取接口的安全要求

如果必须添加 `/api/v1/settings` 端点，必须实现以下安全措施：

| 安全措施 | 实现要求 | 保护对象 |
|---------|---------|---------|
| 字段级白名单 | 仅返回明确允许的字段，默认拒绝 | 所有敏感字段 |
| 敏感字段遮蔽 | 对 `llm.api_key`, `notification_urls` 等进行遮蔽处理 | LLM Key, 通知 URL |
| 独立权限控制 | 配置读取需要独立的管理员权限 | 所有配置 |
| 访问审计日志 | 记录每次配置读取的调用者、时间、IP | 溯源调查 |
| 遮蔽测试 | 新增测试确保敏感字段不会出现在响应中 | 回归防护 |
| 加密存储 | 对磁盘上的敏感字段进行加密 | 存储层防护 |

**字段遮蔽示例**:
```python
# 对通知 URL 的遮蔽
def mask_notification_url(url: str) -> str:
    """遮蔽通知 URL 中的密码/Token 部分"""
    from urllib.parse import urlparse, urlunparse
    parsed = urlparse(url)
    if parsed.password:
        netloc = f"{parsed.username}:****@{parsed.hostname}"
        if parsed.port:
            netloc += f":{parsed.port}"
        parsed = parsed._replace(netloc=netloc)
    return urlunparse(parsed)

# 对 LLM API Key 的遮蔽
def mask_llm_api_key(key: str) -> str:
    """只保留 LLM API Key 的前 4 位和后 4 位"""
    if len(key) <= 8:
        return "****"
    return f"{key[:4]}****{key[-4:]}"
```

---

## 七、安全风险与建议

### 7.1 风险等级评估

| 风险项 | 风险等级 | 影响范围 | 泄露路径 |
|-------|---------|---------|---------|
| 通知 URL（API 暴露） | 🔴 高危 | 所有通知服务凭证 | API 响应 |
| 通知 URL（日志暴露） | 🟠 中高 | 所有通知服务凭证 | 日志系统 |
| Diff 内容无敏感字段遮蔽 | 🟠 中高 | 被监控页面的敏感信息 | 通知渠道 |
| 简单截断破坏格式 | 🟡 中 | 通知内容可读性 | 通知展示 |
| 新增配置读取接口的潜在泄露 | ⚫ 边界风险 | 所有敏感配置 | 未来新增的 API |

### 7.2 现存风险详解

#### 风险 1: 通知 URL 在 API 中明文暴露（高危）

**代码证据**:
- `api/Notifications.py:17-19`: GET 接口直接返回完整 URL
- `api/Notifications.py:46`: POST 接口回显添加的 URL
- `api/Notifications.py:68`: PUT 接口回显替换后的 URL

**攻击路径**:
1. 攻击者获取 API Token（通过泄露、弱密码、内部人员等）
2. 调用 `GET /api/v1/notifications` 获取所有通知 URL
3. 提取 URL 中的 Token/密码（如 Telegram Bot Token、Slack Webhook、邮件 SMTP 密码等）
4. 使用这些凭证接管通知渠道，发送恶意通知或进行进一步攻击

#### 风险 2: 通知 URL 在日志中明文暴露（中高）

**代码证据**:
- `store/__init__.py:1085`: DEBUG 日志记录添加的 URL
- `notification/handler.py:416`: INFO 日志记录发送的 URL

**攻击路径**:
1. 日志系统通常被多个团队访问（运维、开发、安全等）
2. 攻击者通过弱权限获取日志访问权
3. 从日志中提取通知 URL 中的敏感凭证
4. 利用凭证进行进一步攻击

#### 风险 3: Diff 内容无敏感字段遮蔽（中高）

**问题描述**: 系统目前没有对 Diff 内容进行任何敏感字段扫描和遮蔽。

**风险场景**:
- 监控的页面意外泄露了 API Key、密码、Token 等敏感信息
- 这些信息完整出现在 Diff 中并通过通知渠道发送
- 通知渠道（邮件、Slack、Telegram 等）可能被更多人访问
- 敏感信息可能被转发、截图、索引，造成二次泄露

#### 风险 4: 简单截断可能破坏格式（中）

**问题描述**: 长度裁剪采用简单的字符串切片，可能破坏 Markdown/HTML 格式。

**影响**:
- 截断位置可能在 Markdown 链接、代码块中间
- 导致通知内容格式混乱、无法阅读
- 极端情况下可能产生意外的格式解析问题

### 7.3 改进建议

#### 建议 1: 通知 URL 遮蔽（高优先级）

**API 层面**:
```python
# 建议实现: 对 URL 中的敏感部分进行遮蔽
from urllib.parse import urlparse, urlunparse

def mask_notification_url(url: str) -> str:
    """遮蔽通知 URL 中的密码/Token 部分"""
    parsed = urlparse(url)
    if parsed.password:
        # 遮蔽密码: user:****@host
        netloc = f"{parsed.username}:****@{parsed.hostname}"
        if parsed.port:
            netloc += f":{parsed.port}"
        parsed = parsed._replace(netloc=netloc)
    return urlunparse(parsed)
```

**日志层面**:
```python
# 修改日志记录，使用遮蔽后的 URL
logger.info(f">> Process Notification: AppRise start notifying '{mask_notification_url(url)}'")
```

#### 建议 2: Diff 内容敏感词过滤（高优先级）

- 在 `diff/__init__.py` 或 `notification/handler.py` 中增加敏感词遮蔽步骤
- 配置常见敏感字段正则模式：
  - API Key: `sk-[A-Za-z0-9]{20,}`
  - Token: `[a-f0-9]{32,}`
  - 密码: `password[=:]\s*\S+`
  - 私钥: `-----BEGIN (RSA|EC|PGP) PRIVATE KEY-----`
- 支持用户自定义敏感词列表
- 在 Diff 渲染后、通知发送前进行匹配和遮蔽

#### 建议 3: 智能裁剪（中优先级）

- 按词/句边界截断而非简单字节截断
- 超长时添加截断提示（如 `... (内容过长，已截断，查看完整内容请访问: {diff_url})`）
- 确保截断不会破坏 Markdown/HTML 结构

#### 建议 4: LLM API Key 防护增强（低优先级，当前已较完善）

虽然当前 LLM API Key 保护机制较完善，但建议：
- 对存储的 LLM API Key 进行加密（而非明文存储在 JSON 中）
- 增加审计日志，记录 LLM API Key 的使用情况
- 限制 LLM API Key 的使用范围和频率

---

## 八、本次事实校对修正清单

| 修正项 | 原描述 | 修正后 |
|-------|-------|-------|
| LLM 安全测试数量 | 9 个测试用例 | 12 个测试用例（经逐行核对） |
| 代码位置标注 | 多处反引号未闭合 | 所有 `file:line` 格式统一且闭合 |
| 证据链完整性 | 部分证据缺少精确行号 | 所有证据均有精确的文件和行号标注 |
| 边界分析 | 未提及新增配置接口的风险 | 新增第六章「边界变化说明」 |
| LLM 防护归因 | 部分归因不准确 | 明确四层防护机制及核心依赖 |

> 校对日期: 2026-05-18
> 代码基线: commit 42-changedetection.io（当前工作目录）
