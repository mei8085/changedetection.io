# RSS Feed 机制分析报告

## 1. 数据源汇总

RSS feed 的数据源来自 `datastore.data['watching']` 中的所有监控项（Watch）。每个 Watch 包含以下关键属性：

- **`uuid`**: 唯一标识符
- **`url`**: 监控的网页地址
- **`history`**: 历史快照集合（时间戳 → 快照文件路径）
- **`last_changed`**: 最后一次变更的时间戳
- **`last_viewed`**: 最后查看的时间戳
- **`tags`**: 标签集合
- **`notification_muted`**: 是否静音通知
- **`viewed`**: 是否已查看（计算属性，基于 last_viewed >= newest_history_key）

### 1.1 三种 Feed 类型

项目提供三种不同的 RSS feed：

1. **主 Feed** (`/rss`) - 汇总所有符合条件的 Watch
2. **单 Watch Feed** (`/rss/watch/<uuid>`) - 单个监控项的多次变更历史
3. **标签 Feed** (`/rss/tag/<tag_uuid>`) - 特定标签下的所有 Watch

## 2. 过滤逻辑

### 2.1 主 Feed 过滤（main_feed.py:53-59）

主 Feed 生成时会应用以下过滤条件：

```python
for uuid, watch in datastore.data['watching'].items():
    # 条件1: 隐藏静音的 Watch（如果启用）
    if datastore.data['settings']['application'].get('rss_hide_muted_watches') and watch.get('notification_muted'):
        continue
    # 条件2: 过滤特定标签
    if limit_tag and not limit_tag in watch['tags']:
        continue
    sorted_watches.append(watch)
```

**过滤条件汇总：**

| 条件 | 说明 | 可配置 |
|------|------|--------|
| 静音过滤 | 忽略 `notification_muted=True` 的 Watch | 是（`rss_hide_muted_watches`） |
| 标签过滤 | 只包含指定标签的 Watch | 是（URL 参数 `tag`） |
| 历史快照数 | 至少 2 个快照（`len(dates) >= 2`） | 否（硬编码） |
| 未查看 | 只包含 `not watch.viewed` 的 Watch | 否（硬编码） |

### 2.2 标签 Feed 过滤（tag.py:47-65）

```python
for uuid, watch in datastore.data['watching'].items():
    # 条件1: 必须包含指定标签
    if tag_uuid not in watch.get('tags', []):
        continue
    # 条件2: 静音过滤
    if datastore.data['settings']['application'].get('rss_hide_muted_watches') and watch.get('notification_muted'):
        continue
    # 条件3: 至少 2 个历史快照
    if len(dates) < 2:
        continue
    # 条件4: 未查看
    if not watch.viewed:
        ...
```

### 2.3 单 Watch Feed 过滤（single_watch.py:49-59）

```python
# 只检查：
# 1. Watch 存在
# 2. 至少 2 个历史快照
```

单 Watch Feed 没有额外过滤，只要 Watch 存在且有足够历史就生成。

## 3. 排序逻辑

### 3.1 主 Feed 排序（main_feed.py:61, 69-100）

**代码流程：**

```python
# 步骤 1: 按 last_changed 升序排序
sorted_watches.sort(key=lambda x: x.last_changed, reverse=False)

# 步骤 2: 按排序后的顺序依次添加 entry
for watch in sorted_watches:  # 从 last_changed 最小的（最旧）开始遍历
    if not watch.viewed:
        fe = fg.add_entry()
        # entry 的 pubDate 由 timestamp_to（dates[-1]）决定
        fe.pubDate(dt)
```

**实际输出顺序：**

| 环节 | 行为 |
|------|------|
| 排序阶段 | `sorted_watches` 按 `last_changed` **升序**排列（最旧的在前） |
| 添加阶段 | 按排序后的顺序依次调用 `fg.add_entry()`（先加最旧的 Watch，再加较新的） |
| 最终 XML 输出 | 与添加顺序一致，即 **Watch 级别按 last_changed 升序（最旧的 Watch 在前）** |

**关键说明：**
- 主 Feed 每个 Watch 只贡献 **1 个 entry**（最新一次未查看的变更）
- 排序是 **Watch 级别** 的，不是单 Watch 内多次变更的排序
- `reverse=False` 表示升序，即 `last_changed` 值较小的（较早变更的 Watch）排在前面

---

### 3.2 单 Watch Feed 排序（single_watch.py:77-109）

**代码流程：**

```python
# dates 是按时间顺序存储的，dates[-1] 是最新快照
# num_diffs = 可用的 diff 数量

# 注释明确说明：倒序添加，因为 feedgen 会反转
# "Add entries in reverse order because feedgen reverses them"
# "This way, the newest change appears first in the final RSS"
for i in range(num_diffs - 1, -1, -1):  # 从 num_diffs-1 递减到 0
    # i=num_diffs-1 → 最旧的 diff
    # i=0 → 最新的 diff (dates[-2] vs dates[-1])
    date_index_to = -(i + 1)
    timestamp_to = dates[date_index_to]
    
    fe = fg.add_entry()  # 先加最旧的 diff，后加最新的 diff
    fe.pubDate(dt)  # pubDate 与 timestamp_to 对应
```

**实际输出顺序（代码注释 + 测试验证）：**

| 环节 | 行为 | 结果 |
|------|------|------|
| 遍历顺序 | `i` 从 `num_diffs-1` 递减到 `0` | 从**最旧**的 diff 开始，到**最新**的 diff 结束 |
| 添加顺序 | 按遍历顺序调用 `fg.add_entry()` | 先添加**最旧**的 diff，后添加**最新**的 diff |
| 最终 XML | 代码注释 + `test_rss_single_watch_order` 验证 | **最新的变更在最前面** |

**测试验证（test_rss_single_watch_order）：**

```python
# 测试创建了 Version 1 → 2 → 3 → 4 → 5 共 5 个版本（4 个 diff）
# 验证：
assert "5 content" in descriptions[0]  # 第一个 item 是最新的（Version 5）
assert "4 content" in descriptions[1]  # 第二个 item 是 Version 4
assert "3 content" in descriptions[2]  # 第三个 item 是 Version 3
```

**结论：单 Watch Feed 最终输出顺序是最新的变更在前。**

---

### 3.3 标签 Feed 排序（tag.py:47-91）

**注意**: 标签 Feed **没有显式排序**，直接按 `datastore.data['watching'].items()` 的迭代顺序生成。

```python
# @todo  This is wrong, it needs to sort by most recently changed...
for uuid, watch in datastore.data['watching'].items():
    # 没有 sort() 调用
```

代码中已有 TODO 注释指出此问题。

## 4. 内容生成流程

### 4.1 核心流程（以主 Feed 为例）

```
1. 遍历所有 Watch
   ↓
2. 过滤（静音、标签、快照数、未查看）
   ↓
3. 排序（last_changed 升序）
   ↓
4. 为每个符合条件的 Watch 创建 RSS entry
   ├─ 取最后两个快照（dates[-1], dates[-2]）
   ├─ 构建通知上下文（NotificationContextData）
   ├─ 调用 notification_service 渲染内容
   ├─ 调用 process_notification 生成最终内容
   └─ 添加到 FeedGenerator
   ↓
5. 生成 RSS XML
```

### 4.2 模板系统

RSS 内容使用 Jinja2 模板渲染，模板选择逻辑在 `_util.py:71-82` 中定义。

**核心代码（get_rss_template 函数）：**

```python
def get_rss_template(datastore, watch, rss_content_format, default_html, default_plaintext):
    """Get the appropriate template for RSS content."""
    # 优先级 1: 如果 rss_template_type == 'notification_body'，命中即短路返回
    if datastore.data['settings']['application'].get('rss_template_type') == 'notification_body':
        return _check_cascading_vars(datastore=datastore, var_name='notification_body', watch=watch)

    # 优先级 2: 仅当优先级 1 不满足时，才判断 rss_template_override
    override = datastore.data['settings']['application'].get('rss_template_override')
    if override and override.strip():
        return override

    # 优先级 3: 以上都不满足时，根据 rss_content_format 使用默认模板
    elif 'text' in rss_content_format:
        return default_plaintext
    else:
        return default_html
```

---

#### 模板选择优先级与条件分支（从高到低）

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 优先级 1: rss_template_type == 'notification_body'  [短路返回]          │
│   条件: application.rss_template_type 的值等于字符串 'notification_body' │
│   命中则: 立即调用 _check_cascading_vars 获取模板并 return，不再继续     │
│                                                                         │
│   └─→ _check_cascading_vars 级联查找（notification_service.py:17-54）:   │
│       ├─ Watch 级别: watch.get('notification_body')                     │
│       ├─ Tag 级别: tag.get('notification_body')（第一个匹配）           │
│       └─ 全局级别: datastore['settings']['application']['notification_body'] │
├─────────────────────────────────────────────────────────────────────────┤
│ 优先级 2: rss_template_override [仅优先级 1 不满足时才判断]              │
│   条件: application.rss_template_type 不等于 'notification_body'，且     │
│         application.rss_template_override 存在且非空                     │
│   命中则: 直接 return override 字符串，不再继续                          │
├─────────────────────────────────────────────────────────────────────────┤
│ 优先级 3: 默认模板 [以上都不满足时]                                      │
│   条件: application.rss_template_type 不等于 'notification_body'，且     │
│         application.rss_template_override 不存在或为空                   │
│   命中则: 根据 application.rss_content_format 选择默认模板               │
│       ├─ 包含 'text' → RSS_TEMPLATE_PLAINTEXT_DEFAULT                   │
│       └─ 否则 → RSS_TEMPLATE_HTML_DEFAULT                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

#### 条件分支依据说明

**1. 优先级 1 短路返回的依据（_util.py:73-74）：**

```python
if datastore.data['settings']['application'].get('rss_template_type') == 'notification_body':
    return _check_cascading_vars(...)  # 直接 return，函数在此结束
```

- **行为**: 函数第一条 `if` 语句直接 `return`，命中后不会执行后续任何代码
- **效果**: 即使同时设置了 `rss_template_override`，只要 `rss_template_type == 'notification_body'`，`override` 就会被完全忽略

**2. 优先级 2 仅在优先级 1 不满足时判断的依据（_util.py:76-78）：**

```python
override = datastore.data['settings']['application'].get('rss_template_override')
if override and override.strip():
    return override
```

- **行为**: 这是 `if ... return ...` 之后的第二段代码
- **效果**: 只有当优先级 1 的 `if` 条件不成立时，代码才会执行到这里

**3. 优先级 3 的依据（_util.py:79-82）：**

```python
elif 'text' in rss_content_format:
    return default_plaintext
else:
    return default_html
```

- **行为**: 这是 `if ... return ...` 之后的 `elif/else` 分支
- **效果**: 只有当前面两个优先级都不满足时，才会使用默认模板

**4. `rss_template_type` 其他值的处理（model/App.py:64）：**

```python
'rss_template_type': 'system_default',  # 默认值
```

- **默认值**: `'system_default'`
- **行为**: 当 `rss_template_type` 为 `'system_default'` 或其他不等于 `'notification_body'` 的值时，优先级 1 条件不成立，继续判断优先级 2 和 3

---

#### 级联优先级详解（_check_cascading_vars，notification_service.py:17-54）

当命中优先级 1 时，`_check_cascading_vars` 内部还会进行三级级联查找：

```python
def _check_cascading_vars(datastore, var_name, watch):
    """
    Check notification variables in cascading priority:
    Individual watch settings > Tag settings > Global settings
    """
    # 第 1 级: Watch 级别
    v = watch.get(var_name)
    if v and not watch.get('notification_muted'):
        return v

    # 第 2 级: Tag 级别（按顺序查找，第一个匹配的返回）
    tags = datastore.get_all_tags_for_watch(uuid=watch.get('uuid'))
    if tags:
        for tag_uuid, tag in tags.items():
            v = tag.get(var_name)
            if v and not tag.get('notification_muted'):
                return v

    # 第 3 级: 全局级别
    if datastore.data['settings']['application'].get(var_name):
        return datastore.data['settings']['application'].get(var_name)

    # 默认值
    if var_name == 'notification_body':
        return default_notification_body
    return None
```

---

#### 测试验证（test_rss_single_watch_follow_notification_body）

测试验证了当 `rss_template_type == 'notification_body'` 时，级联优先级为 **Watch 级别 > Tag 级别 > 全局级别**：

```python
# 步骤 1: 设置全局 notification_body，rss_template_type = 'notification_body'
"application-notification_body": 'Boo yeah hello from main settings'
"application-rss_template_type": 'notification_body'
# 结果: 使用全局模板

# 步骤 2: 设置 Tag 级别 notification_body
data={"name": "rss-custom", "notification_body": 'Hello from the group/tag level'}
# 结果: Tag 级覆盖全局级

# 步骤 3: 设置 Watch 级别 notification_body
data={"notification_body": "RSS body set from watch level"}
# 结果: Watch 级覆盖 Tag 级
```

---

#### 默认模板定义（__init__.py:23-27）

```python
RSS_TEMPLATE_PLAINTEXT_DEFAULT = "<pre>{{watch_label}} had a change.\n\n{{diff}}\n</pre>"
RSS_TEMPLATE_HTML_DEFAULT = """<html><body>
<h4><a href="{{watch_url}}">{{watch_label}}</a></h4>
<p>{{diff}}</p>
</body></html>
"""
```

## 5. Feed 与界面的内容片段一致性

### 5.1 核心机制：共享 Diff 引擎

Feed 和 Web 界面使用**完全相同的 diff 渲染引擎**，确保内容片段一致性：

```
                    ┌─────────────────┐
                    │  render_diff()  │  ← 核心 diff 引擎
                    │  (diff/__init__.py) │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │   Web 界面      │           │   RSS Feed      │
    │  (text_json_diff│           │  (notification  │
    │   /difference.py)│          │   /handler.py)  │
    └─────────────────┘           └─────────────────┘
```

### 5.2 Placemarker 机制

Diff 引擎使用**占位符（Placemarker）**而非直接生成 HTML：

**Placemarker 常量**（diff/__init__.py:33-43）：

```python
REMOVED_PLACEMARKER_OPEN = '@removed_PLACEMARKER_OPEN'
REMOVED_PLACEMARKER_CLOSED = '@removed_PLACEMARKER_CLOSED'
ADDED_PLACEMARKER_OPEN = '@added_PLACEMARKER_OPEN'
ADDED_PLACEMARKER_CLOSED = '@added_PLACEMARKER_CLOSED'
CHANGED_PLACEMARKER_OPEN = '@changed_PLACEMARKER_OPEN'
CHANGED_PLACEMARKER_CLOSED = '@changed_PLACEMARKER_CLOSED'
CHANGED_INTO_PLACEMARKER_OPEN = '@changed_into_PLACEMARKER_OPEN'
CHANGED_INTO_PLACEMARKER_CLOSED = '@changed_into_PLACEMARKER_CLOSED'
```

### 5.3 渲染路径对比

#### Web 界面渲染路径（text_json_diff/difference.py:183-196）

```python
# 1. 生成带 placemarkers 的 diff
content = diff.render_diff(
    previous_version_file_contents=from_version_file_contents,
    newest_version_file_contents=to_version_file_contents,
    include_replaced=diff_prefs['replaced'],
    include_added=diff_prefs['added'],
    include_removed=diff_prefs['removed'],
    include_equal=diff_prefs['changesOnly'],
    ignore_junk=diff_prefs['ignoreWhitespace'],
    word_diff=diff_prefs['type'] == 'diffWords',
)

# 2. 转换为带颜色的 HTML
content = apply_html_color_to_body(n_body=content)
```

#### RSS Feed 渲染路径（notification/handler.py:360-424）

```python
# 1. 生成带 placemarkers 的 diff（通过 add_rendered_diff_to_notification_vars）
n_object.update(add_rendered_diff_to_notification_vars(
    notification_scan_text=...,
    current_snapshot=...,
    prev_snapshot=...,
    word_diff=...  # 与界面相同的 word_diff 参数
))

# 2. 渲染 Jinja2 模板
n_body = jinja_render(template_str=n_object.get('notification_body', ''), **notification_parameters)

# 3. 根据格式转换 placemarkers
(url, n_body, n_title) = apply_service_tweaks(
    url=url, 
    n_body=n_body, 
    n_title=n_title, 
    requested_output_format=requested_output_format_original
)
```

### 5.4 格式转换函数

不同输出格式使用不同的 placemarker 转换：

| 函数 | 文件 | 用途 |
|------|------|------|
| `apply_html_color_to_body()` | notification/handler.py:86-101 | Web 界面、HTML 通知 |
| `apply_discord_markdown_to_body()` | notification/handler.py:103-125 | Discord 通知 |
| `apply_standard_markdown_to_body()` | notification/handler.py:127-150 | Markdown 通知 |
| `replace_placemarkers_in_text()` | notification/handler.py:153-207 | 通用转换（Telegram 等） |

**关键**: 所有转换函数都从**相同的 diff 输出**开始，只是最终表示形式不同。

### 5.5 数据一致性保障

#### 快照来源一致性

- **Web 界面**: `watch.get_history_snapshot(timestamp=from_version)` / `to_version`
- **RSS Feed**: `watch.get_history_snapshot(timestamp=dates[-1])` / `dates[-2]`

两者都使用相同的 `get_history_snapshot()` 方法读取历史快照。

#### Diff 参数一致性

```python
# Web 界面（用户可配置）
word_diff=diff_prefs['type'] == 'diffWords'
include_replaced=diff_prefs['replaced']
include_added=diff_prefs['added']
include_removed=diff_prefs['removed']

# RSS Feed（固定默认）
word_diff=False if requested_output_format_original == 'text' else True
# include_replaced/add/removed 使用 render_diff 默认值（True）
```

**注意**: RSS Feed 使用的是固定的 diff 参数，而 Web 界面允许用户动态调整（`changesOnly`、`ignoreWhitespace` 等）。这意味着：

- **基本的 added/removed 高亮**: 一致
- **高级过滤选项**: 可能不一致（如用户在界面隐藏了 removed 行，但 RSS 仍显示）

### 5.6 FormattableDiff 类

为了在 Jinja2 模板中灵活使用 diff，系统提供了 `FormattableDiff` 类（notification_service.py:114-166）：

```python
class FormattableDiff(str):
    def __call__(self, lines=None, added_only=False, removed_only=False, 
                 context=0, word_diff=None, case_insensitive=False, ignore_junk=False):
        # 允许在模板中动态调整 diff 渲染
        result = diff_module.render_diff(...)
        return result
```

这允许模板中这样使用：

```jinja2
{{ diff }}                    # 默认 diff
{{ diff(lines=5) }}           # 只显示前 5 行
{{ diff(added_only=true) }}   # 只显示新增内容
{{ diff(word_diff=false) }}   # 行级 diff 而非词级
```

## 6. 关键文件索引

| 文件 | 职责 | 关键行号 |
|------|------|----------|
| `blueprint/rss/blueprint.py` | RSS 蓝图路由注册 | 9-26 |
| `blueprint/rss/main_feed.py` | 主 Feed 生成逻辑 | 23-105 |
| `blueprint/rss/single_watch.py` | 单 Watch Feed 生成 | 13-115 |
| `blueprint/rss/tag.py` | 标签 Feed 生成 | 11-95 |
| `blueprint/rss/_util.py` | RSS 工具函数 | 1-155 |
| `blueprint/rss/__init__.py` | 默认模板定义 | 20-27 |
| `notification/handler.py` | 通知处理（含 diff 渲染） | 307-496 |
| `notification_service.py` | 通知服务（diff 变量构建） | 171-299 |
| `diff/__init__.py` | 核心 diff 引擎 | 424-507 |
| `processors/text_json_diff/difference.py` | Web 界面 diff 渲染 | 104-250 |
| `model/Watch.py` | Watch 模型（history 管理） | 429-435, 442-491 |

## 7. 总结

### 7.1 数据源汇总
- 来自 `datastore.data['watching']` 中的所有 Watch 对象
- 按 `last_changed` 升序排序（主 Feed）
- 支持按标签过滤（URL 参数 `tag`）

### 7.2 过滤机制
- **静音过滤**: 可配置 `rss_hide_muted_watches`
- **标签过滤**: URL 参数 `tag` 支持
- **快照数**: 至少 2 个才显示
- **未查看**: 主 Feed 和标签 Feed 只包含未查看的变更

### 7.3 排序方式
- **主 Feed**: 先按 `last_changed` 升序排序，再按排序顺序依次添加，最终输出 Watch 级别按 `last_changed` 升序（最旧的 Watch 在前）
- **单 Watch Feed**: 从最旧的 diff 开始倒序遍历添加，代码注释 + 测试验证最终**最新的变更在前**
- **标签 Feed**: 无显式排序（已知问题，代码有 TODO 注释）

### 7.4 模板选择逻辑
优先级从高到低：

| 优先级 | 条件 | 行为 |
|--------|------|------|
| 1 | `rss_template_type == 'notification_body'` | 调用 `_check_cascading_vars` 级联查找：**Watch 级别 > Tag 级别 > 全局级别** |
| 2 | `rss_template_override` 存在且非空 | 直接使用 override 模板 |
| 3 | 以上都不满足 | 根据 `rss_content_format` 选择默认模板 |

**测试依据**: `test_rss_single_watch_follow_notification_body` 验证了 Watch > Tag > Global 的级联优先级。

### 7.5 内容一致性保障
1. **共享 diff 引擎**: `render_diff()` 是唯一的 diff 生成源
2. **Placemarker 机制**: 先生成带占位符的文本，再按输出格式转换
3. **统一快照读取**: 使用相同的 `get_history_snapshot()` 方法
4. **注意**: 高级过滤选项（changesOnly、ignoreWhitespace 等）在 Web 界面可配置，但 RSS 使用固定默认值

