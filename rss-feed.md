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

### 3.1 主 Feed 排序（main_feed.py:61）

```python
sorted_watches.sort(key=lambda x: x.last_changed, reverse=False)
```

- **排序键**: `watch.last_changed`（最后一次变更的时间戳）
- **方向**: 升序（`reverse=False`），即**最旧的变更在前**
- **注意**: 这是 Watch 级别的排序，不是单 Watch 内多次变更的排序

### 3.2 单 Watch Feed 排序（single_watch.py:77-109）

```python
# 遍历顺序（倒序创建，因为 feedgen 会反转）
for i in range(num_diffs - 1, -1, -1):
    # 从最旧到最新依次添加
    # feedgen 最终会按 pubDate 排序
    fe = fg.add_entry()
    fe.pubDate(dt)  # 使用 timestamp_to 设置
```

- 单 Watch 内的多次变更按时间戳排序
- **最终 RSS 结果**: 由 feedgen 根据 `pubDate` 决定，通常最新的在前

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

RSS 内容使用 Jinja2 模板渲染，模板优先级：

```
1. 自定义覆盖模板（rss_template_override）
2. 通知正文模板（rss_template_type = 'notification_body'）
   → 使用 _check_cascading_vars 级联查找：
     - Watch 级别 → Tag 级别 → 全局级别
3. 默认模板（根据 rss_content_format）
   - HTML 默认: RSS_TEMPLATE_HTML_DEFAULT
   - 纯文本默认: RSS_TEMPLATE_PLAINTEXT_DEFAULT
```

**默认模板定义**（__init__.py:23-27）：

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
- 按 `last_changed` 排序（主 Feed）
- 支持按标签过滤

### 7.2 过滤机制
- **静音过滤**: 可配置 `rss_hide_muted_watches`
- **标签过滤**: URL 参数 `tag` 支持
- **快照数**: 至少 2 个才显示
- **未查看**: 主 Feed 和标签 Feed 只包含未查看的变更

### 7.3 排序方式
- **主 Feed**: `last_changed` 升序（旧的在前）
- **单 Watch Feed**: 按 `pubDate` 由 feedgen 排序
- **标签 Feed**: 无显式排序（已知问题）

### 7.4 内容一致性保障
1. **共享 diff 引擎**: `render_diff()` 是唯一的 diff 生成源
2. **Placemarker 机制**: 先生成带占位符的文本，再按输出格式转换
3. **统一快照读取**: 使用相同的 `get_history_snapshot()` 方法
4. **注意**: 高级过滤选项（changesOnly、ignoreWhitespace 等）在 Web 界面可配置，但 RSS 使用固定默认值

