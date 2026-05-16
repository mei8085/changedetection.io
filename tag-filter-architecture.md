# 标签分组与过滤架构分析报告

## 1. 标签在存储层的组织方式

### 1.1 标签数据结构

**核心存储位置**：
- 内存中：`datastore.data['settings']['application']['tags']` - 字典结构，键为标签UUID
- 磁盘上：`{datastore_path}/{tag_uuid}/tag.json` - 每个标签独立目录和JSON文件

**代码依据**：
```python
# changedetectionio/store/__init__.py:974-977
self.__data['settings']['application']['tags'][new_uuid] = new_tag
# Save tag to its own tag.json file instead of settings
new_tag.commit()
```

**标签模型继承链**：
```
Tag.model → EntityPersistenceMixin → watch_base (dict子类)
```

**代码依据**：
```python
# changedetectionio/model/Tag.py:26
class model(EntityPersistenceMixin, watch_base):
```

### 1.2 Watch 与 Tag 关联机制

**关联存储方式**：Watch 对象通过 `watch['tags']` 数组存储关联的标签UUID列表

**代码依据**：
```python
# changedetectionio/store/__init__.py:513-514
for uuid, watch in self.__data['watching'].items():
    if watch.get('tags') and tag_uuid in watch['tags']:
```

**关联类型**：

| 类型 | 实现方式 | 代码位置 |
|------|----------|----------|
| **手动关联** | 用户在编辑页面直接选择标签 | `blueprint/ui/edit.py` |
| **URL自动匹配** | 通过 `url_match_pattern` 属性自动匹配 | `model/Tag.py:55-67` |

**URL自动匹配实现**：
```python
# changedetectionio/model/Tag.py:55-67
def matches_url(self, url: str) -> bool:
    pattern = self.get('url_match_pattern', '').strip()
    if not pattern or not url:
        return False
    if any(c in pattern for c in ('*', '?', '[')):
        return fnmatch.fnmatch(url.lower(), pattern.lower())
    return pattern.lower() in url.lower()
```

### 1.3 标签关联查询

**统一查询入口**：`get_all_tags_for_watch(uuid)` - 返回手动分配 + URL自动匹配的所有标签

**代码依据**：
```python
# changedetectionio/store/__init__.py:980-996
def get_all_tags_for_watch(self, uuid):
    watch = self.data['watching'].get(uuid)
    if not watch:
        return {}

    # Start with manually assigned tags
    result = dictfilt(self.__data['settings']['application']['tags'], watch.get('tags', []))

    # Additionally include any tag whose url_match_pattern matches this watch's URL
    watch_url = watch.get('url', '')
    if watch_url:
        for tag_uuid, tag in self.__data['settings']['application']['tags'].items():
            if tag_uuid not in result and tag.matches_url(watch_url):
                result[tag_uuid] = tag

    return result
```

**该方法调用位置**（用于过滤和展示）：
- API 列表过滤：`api/Watch.py:520`
- UI 页面展示：`blueprint/watchlist/templates/watch-overview.html:309`
- 通知服务：`notification_service.py:36`
- RSS 生成：`blueprint/rss/tag.py`

### 1.4 持久化与生命周期管理

**统一持久化机制**：`EntityPersistenceMixin` 为 Watch 和 Tag 提供统一的持久化能力

**代码依据**：
```python
# changedetectionio/model/persistence.py:52-84
def _save_to_disk(self, data_dict, uuid):
    # Determine entity type (cached at class level, not instance level)
    entity_type = _determine_entity_type(self.__class__)

    # Set filename and size limits based on entity type
    filename = f'{entity_type}.json'
    max_size_mb = 10 if entity_type == 'watch' else 1
```

**标签集合管理**：`TagsDict` 自定义字典，重写删除操作确保文件清理

**代码依据**：
```python
# changedetectionio/model/Tags.py:9-39
class TagsDict(dict):
    def __delitem__(self, key: str) -> None:
        super().__delitem__(key)
        tag_dir = self._datastore_path / key
        tag_json_file = tag_dir / "tag.json"
        # ... cleanup logic
```

---

## 2. 队列处理器调度时的标签感知机制

### 2.1 队列调度架构概述

**队列核心组件**：`RecheckPriorityQueue` - 基于优先级的线程安全队列

**关键发现**：队列调度器**本身不直接感知标签**，标签逻辑在更高业务层处理

### 2.2 标签感知的三个场景

#### 场景1：按标签批量重新检查

**触发入口**：`GET /api/v1/tag/<uuid>?recheck=true`

**实现逻辑**：
1. 遍历所有 Watch，找到包含该标签且未暂停的 Watch
2. 将匹配的 Watch 加入重新检查队列
3. 超过20个 Watch 时使用后台线程异步处理

**代码依据**：
```python
# changedetectionio/api/Tags.py:19-65
if request.args.get('recheck'):
    # Recheck all watches with this tag, including muted
    watches_to_queue = []
    for k in sorted(self.datastore.data['watching'].items(), key=lambda item: item[1].get('last_checked', 0)):
        watch_uuid = k[0]
        watch = k[1]
        if not watch['paused'] and tag['uuid'] in watch['tags']:
            watches_to_queue.append(watch_uuid)

    # Queue logic follows...
```

#### 场景2：标签配置变更强制重处理

**触发入口**：标签更新 API 和 UI 编辑页面

**实现逻辑**：
1. 标签配置更新后，清除所有关联 Watch 的校验和文件
2. 强制下一次检查时完整重新处理，而非使用校验和快速跳过

**代码依据**：
```python
# changedetectionio/api/Tags.py:154
cleared_count = self.datastore.clear_checksums_for_tag(uuid)
logger.info(f"Tag {uuid} updated via API, cleared {cleared_count} watch checksums")
```

```python
# changedetectionio/store/__init__.py:499-520
def clear_checksums_for_tag(self, tag_uuid):
    """
    Delete last-checksum.txt files for all watches using a specific tag.
    This should be called when a tag configuration is edited, since watches
    inherit tag settings and need to reprocess.
    """
    deleted_count = 0
    for uuid, watch in self.__data['watching'].items():
        if watch.get('tags') and tag_uuid in watch['tags']:
            if watch.data_dir:
                checksum_file = os.path.join(watch.data_dir, 'last-checksum.txt')
                # ... delete logic
```

#### 场景3：Worker 运行时的配置覆盖

**执行时机**：Worker 处理 Watch 时，在处理器执行前动态解析

**配置覆盖原则**：`Watch.field → Tag.field (if overrides_watch) → Global.field`

**代码依据**（来自文档注释）：
```python
# changedetectionio/model/Tag.py:7-14
Tags can override Watch settings when overrides_watch=True.
Current implementation requires manual checking in processors:

    for tag_uuid in watch.get('tags'):
        tag = datastore['settings']['application']['tags'][tag_uuid]
        if tag.get('overrides_watch'):
            restock_settings = tag.get('restock_settings', {})
            break
```

**可覆盖的配置类型**：
- LLM 相关配置（`llm_intent`、`llm_change_summary_prompt`）
- 处理器特定配置（如 `restock_settings`）
- 内容过滤配置（`include_filters`、`subtractive_selectors`）
- 通知静音状态

---

## 3. 差异计算模块与标签的关系

### 3.1 差异计算模块架构

**核心模块**：`changedetectionio/diff/__init__.py`

**主要函数**：
| 函数 | 职责 |
|------|------|
| `render_diff()` | 生成差异HTML输出 |
| `customSequenceMatcher()` | 序列比较核心逻辑 |
| `render_inline_word_diff()` | 单词级内联差异渲染 |
| `render_nested_line_diff()` | 嵌套行级差异渲染 |

### 3.2 关键结论：差异计算模块不直接感知标签

**纯函数设计原则**：差异计算模块是**无状态的纯函数**，不直接访问或感知标签信息

**代码依据**：
```python
# changedetectionio/diff/__init__.py:424-457
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
    # 仅接收文本内容和计算参数，无任何标签相关输入
```

### 3.3 标签对差异计算的间接影响路径

**影响链**：标签通过配置覆盖机制间接影响差异计算的**输入参数**

```
标签配置 → Watch 配置覆盖 → 处理器设置 → 差异计算输入参数 → 计算结果
```

**具体影响的配置项**：

| 配置项 | 影响方式 | 代码位置 |
|--------|----------|----------|
| `include_filters` | 控制提取哪些内容进行比较 | `model/Tag.py:30-38` |
| `subtractive_selectors` | 控制排除哪些内容 | `model/Tag.py:30-38` |
| `ignore_whitespace` | 是否忽略空白字符变化 | `diff/__init__.py:436` |
| `filter_text_added/removed/replaced` | 过滤特定类型的变化 | `diff/__init__.py:428-430` |
| `processor` | 选择不同的差异比较逻辑 | 各 processor 实现 |

**代码依据（标签覆盖检查）**：
```python
# changedetectionio/blueprint/ui/edit.py:39-43
def _watch_has_tag_options_set(watch):
    for tag_uuid, tag in datastore.data['settings']['application'].get('tags', {}).items():
        if tag_uuid in watch.get('tags', []) and (tag.get('include_filters') or tag.get('subtractive_selectors')):
            return True
```

### 3.4 差异计算的纯函数特性验证

**输入仅包含**：
1. 前一版本文本内容
2. 当前版本文本内容
3. 计算选项（布尔标志、字符串参数）

**输出仅包含**：
- 差异结果HTML或补丁格式

**无外部依赖**：不访问 datastore、不读取标签信息、无副作用

---

## 4. 架构总结与代码索引

### 4.1 标签功能分层架构

```
┌─────────────────────────────────────────────────────────┐
│  表现层 (UI/API)                                         │
│  - 标签过滤列表展示                                       │
│  - 按标签批量操作                                         │
│  - watchlist/__init__.py, api/Tags.py                    │
├─────────────────────────────────────────────────────────┤
│  业务逻辑层 (处理器/Worker)                               │
│  - 配置覆盖解析                                           │
│  - 标签组级操作                                           │
│  - worker.py, processors/*                               │
├─────────────────────────────────────────────────────────┤
│  存储层 (数据模型/持久化)                                  │
│  - Tag 模型                                              │
│  - Watch-Tag 关联                                        │
│  - model/Tag.py, model/Tags.py, store/__init__.py        │
└─────────────────────────────────────────────────────────┘
```

### 4.2 关键代码位置索引

| 功能点 | 文件路径 | 行号 |
|--------|----------|------|
| 标签模型定义 | `changedetectionio/model/Tag.py` | 1-71 |
| 标签集合管理 | `changedetectionio/model/Tags.py` | 1-39 |
| 统一持久化 Mixin | `changedetectionio/model/persistence.py` | 1-84 |
| 获取 Watch 的所有标签 | `changedetectionio/store/__init__.py` | 980-996 |
| 清除标签关联 Watch 的校验和 | `changedetectionio/store/__init__.py` | 499-520 |
| 按标签批量重新检查 | `changedetectionio/api/Tags.py` | 19-65 |
| Watch 列表按标签过滤 | `changedetectionio/api/Watch.py` | 517-522 |
| 差异计算核心函数 | `changedetectionio/diff/__init__.py` | 424-507 |
| Worker 主处理流程 | `changedetectionio/worker.py` | 46-743 |
| 标签覆盖配置检查 | `changedetectionio/blueprint/ui/edit.py` | 39-43 |
| 标签 URL 匹配方法 | `changedetectionio/model/Tag.py` | 55-67 |

### 4.3 设计特点总结

| 特点 | 说明 |
|------|------|
| **关注点分离** | 差异计算是纯函数，标签逻辑在上层处理，便于测试和维护 |
| **灵活的配置继承** | 标签可以覆盖 Watch 配置，实现组级别的设置管理 |
| **松耦合架构** | 队列调度器不需要理解标签，标签逻辑集中在业务层 |
| **统一持久化** | Tag 和 Watch 复用相同的持久化机制，减少代码重复 |
| **纯函数设计** | 差异计算无外部依赖，保证了可测试性和可预测性 |

---

**报告生成时间**：2026-05-16
**代码版本**：commit 130-changedetection.io
