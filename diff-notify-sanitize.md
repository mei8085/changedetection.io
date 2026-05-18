# Diff 通知与敏感字段处理流程分析

## 概述

本文档分析 changedetection.io 项目中内容差异（diff）生成、通知推送前的内容裁剪策略、敏感字段识别与遮蔽，以及模板渲染的完整处理链路。

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

#### 层级 1: API 响应字段过滤 (外部接口)

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

#### 层级 2: 持久化过滤 (磁盘存储)

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

代码位置: `api/Notifications.py:12-19
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

代码位置: `api/Notifications.py:46
```python
return {'notification_urls': added_urls}, 201
```

**证据 3: PUT /api/v1/notifications 回显替换后的 URL**

代码位置: `api/Notifications.py:68
```python
return {'notification_urls': clean_urls}, 200
```

**风险影响**:
- 任何拥有 API Token 的用户/攻击者可以读取所有通知 URL
- 通知 URL 中包含的 Token、密码、Webhook 密钥等全部明文暴露
- 例如: `tgram://123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11` 这样的 Telegram Bot Token 会被完整获取

#### 3.2.2 通知 URL 的日志暴露证据链

**证据 1: 添加通知 URL 时的 DEBUG 日志

代码位置: `store/__init__.py:1085
```python
logger.debug(f">>> Adding new notification_url - '{notification_url}'")
```

**证据 2: 发送通知时的 INFO 日志**

代码位置: `notification/handler.py:416
```python
logger.info(f">> Process Notification: AppRise start notifying '{url}'")
```

**风险影响**:
- 在 DEBUG 或 INFO 日志级别下，完整的通知 URL（包含敏感凭证）会被记录到日志系统
- 日志系统通常被多个团队/人员访问，存在严重的横向泄露风险
- 日志持久化后，敏感凭证可能在备份中永久留存

### 3.3 LLM API Key 的保护机制与真实防护边界

#### 3.3.1 真实防护机制（修正版）

LLM API Key 是特殊的敏感字段，其保护机制如下：

**层级 1: 根本没有读取 API 端点（最核心防护）

代码证据: `tests/test_llm_api_key_security.py:265-294
```python
def test_no_api_settings_endpoint_exists(
        client, live_server, measure_memory_usage, datastore_path):
    """
    There is currently no /api/v1/settings endpoint.
    If one is added in the future it must be covered by its own
    security tests before reaching production.
    """
    res_get = client.get('/api/v1/settings', headers={'x-api-key': api_token})
    assert res_get.status_code in (404, 405)
```

**关键事实**: 系统目前**没有 `/api/v1/settings` 端点**，LLM API Key 无法通过 API 直接读取。

**层级 2: 设置页面使用 PasswordField 保护**

代码证据: `tests/test_llm_api_key_security.py:301-317
```python
def test_settings_page_does_not_render_llm_api_key_in_plaintext(
        client, live_server, measure_memory_usage, datastore_path):
    """
    The settings page renders the API key form.  Because the field uses
    PasswordField, WTForms must NOT embed the current key value in the HTML
    (PasswordField intentionally omits the value attribute for security).
    """
    res = client.get(url_for('settings.settings_page'))
    assert res.status_code == 200
    body = res.data.decode('utf-8', errors='replace')
    assert CANARY_KEY not in body
```

**层级 3: 系统字段过滤机制

通过 `strip_internal_api_fields()` 过滤 `SYSTEM_MANAGED_NON_SPEC_FIELDS` 中的内部字段（如 `_llm_result`, `_llm_intent` 等 LLM 运行时数据。

**层级 4: 全面的安全测试保障

`tests/test_llm_api_key_security.py` 包含 9 个测试用例，确保 LLM API Key 不会出现在：
- GET/POST/PUT /api/v1/watch 响应
- GET /api/v1/tag 响应
- GET /api/v1/systeminfo 响应
- GET/POST/PUT /api/v1/notifications 响应
- GET /api/v1/search 响应
- GET /api/v1/full-spec 响应
- 设置页面 HTML 源码

#### 3.3.2 LLM API Key 的防护边界总结

| 防护层级 | 防护机制 | 代码位置 |
|---------|---------|----------|
| API 读取 | ❌ 无设置 API 端点（核心防护） | `tests/test_llm_api_key_security.py:265-294 |
| Web UI | PasswordField 不渲染值 | `tests/test_llm_api_key_security.py:301-317 |
| 系统字段过滤 | `strip_internal_api_fields()` | `api/__init__.py |
| 测试保障 | 9 个安全测试用例 | `tests/test_llm_api_key_security.py |

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

## 六、安全风险与建议

### 6.1 现存风险

1. **Diff 内容无敏感字段遮蔽**: 被监控页面的敏感信息会完整出现在通知中
2. **通知 URL 无遮蔽**: API 返回和日志中可能暴露通知服务的 Token
3. **简单截断可能破坏格式**: 超长内容被生硬截断，可能导致 Markdown/HTML 格式损坏

### 6.2 改进建议

1. **增加 Diff 内容敏感词过滤**:
   - 配置常见敏感字段正则（密码、Token、API Key、私钥等）
   - 在 Diff 渲染后、通知发送前进行匹配和遮蔽
   - 支持用户自定义敏感词列表

2. **通知 URL 遮蔽**:
   - 在 API 返回时对 URL 中的密码/Token 部分进行遮蔽（如 `****`）
   - 日志输出时同样进行遮蔽

3. **智能裁剪**:
   - 按词/句边界截断而非简单字节截断
   - 超长时添加截断提示（如 `... (内容过长，已截断)`）
