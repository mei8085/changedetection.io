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

## 2. 标签覆盖规则详解

### 2.1 核心规则区分

标签配置覆盖分为**三种完全不同的机制**，各自有不同的触发条件和行为：

| 机制 | 触发条件 | 行为 | 适用属性 |
|------|----------|------|----------|
| **列表直接合并** | 不检查 `overrides_watch`，只要标签有该属性就生效 | 所有匹配标签的列表值合并为一个列表 | 选择器类属性（列表类型） |
| **单值级联匹配** | 不检查 `overrides_watch`，只要标签有该属性就生效 | Watch 有值则用 Watch，否则取第一个有值的标签 | LLM 配置（字符串类型） |
| **条件覆盖** | 必须满足 `overrides_watch=True` 才生效 | 第一个匹配标签的属性值完全替换 Watch 配置 | restock 处理器配置 |

---

### 2.2 机制一：get_tag_overrides_for_watch 列表直接合并

**函数实现**：
```python
# changedetectionio/store/__init__.py:936-945
def get_tag_overrides_for_watch(self, uuid, attr):
    tags = self.get_all_tags_for_watch(uuid=uuid)
    ret = []

    if tags:
        for tag_uuid, tag in tags.items():
            if attr in tag and tag[attr]:
                ret = [*ret, *tag[attr]]  # 直接合并所有匹配标签的属性

    return ret
```

**关键特征**：
1. ❌ **不检查** `overrides_watch` 属性
2. ✅ 合并**所有**关联标签的该属性值
3. ⚠️ 只适用于**列表类型**的属性（使用 `*` 展开合并）

**直接合并的属性列表**：

| 属性名 | 调用位置 | 合并策略 |
|--------|----------|----------|
| `include_filters` | `processors/text_json_diff/processor.py:60` | Watch + 所有标签 → 去重合并 |
| `subtractive_selectors` | `processors/text_json_diff/processor.py:83` | 所有标签 + Watch + 全局 → 顺序合并 |
| `extract_lines_containing` | `processors/text_json_diff/processor.py:90` | Watch + 所有标签 → 去重合并 |
| `extract_text` | `processors/text_json_diff/processor.py:94` | Watch + 所有标签 → 去重合并 |
| `ignore_text` | `processors/text_json_diff/processor.py:98` | Watch + 所有标签 + 全局 → 去重合并 |

**代码依据（合并逻辑）**：
```python
# changedetectionio/processors/text_json_diff/processor.py:57-67
def _get_merged_rules(self, attr, include_global=False):
    """Merge rules from watch, tags, and optionally global settings."""
    watch_rules = self.watch.get(attr, [])
    tag_rules = self.datastore.get_tag_overrides_for_watch(uuid=self.watch_uuid, attr=attr)
    rules = list(dict.fromkeys(watch_rules + tag_rules))  # Watch + Tags 合并后去重

    if include_global:
        global_rules = self.datastore.data['settings']['application'].get(f'global_{attr}', [])
        rules = list(dict.fromkeys(rules + global_rules))

    return rules
```

---

### 2.3 机制二：LLM 单值级联匹配（不检查 overrides_watch）

**优先级链路**：`Watch.field → 第一个有值的 Tag.field → 空值`

**关键特征**：
1. ❌ **不检查** `overrides_watch` 属性
2. 🎯 **第一个匹配优先**：遍历标签列表，第一个有非空值的标签获胜
3. 🔄 **回退机制**：Watch 有值则直接返回，否则才回退到标签查找
4. 📋 适用于**单值字符串**配置（llm_intent, llm_change_summary）

**LLM 标签覆盖的属性列表**：

| 属性名 | 解析函数 | 覆盖策略 |
|--------|----------|----------|
| `llm_intent` | `resolve_intent()` | Watch 有值则用 Watch，否则取第一个有值的标签 |
| `llm_change_summary` | `resolve_llm_field()` | Watch 有值则用 Watch，否则取第一个有值的标签 |

**代码依据（resolve_llm_field 通用解析）**：
```python
# changedetectionio/llm/evaluator.py:156-173
def resolve_llm_field(watch, datastore, field: str) -> tuple[str, str]:
    """
    Generic cascade resolver for any LLM per-watch field.
    Returns (value, source) where source is 'watch' or tag title.
    Returns ('', '') if not set anywhere.
    """
    value = (watch.get(field) or '').strip()
    if value:
        return value, 'watch'  # Watch 优先级最高

    # 不检查 overrides_watch，直接遍历标签找第一个有值的
    for tag_uuid in watch.get('tags', []):
        tag = datastore.data['settings']['application'].get('tags', {}).get(tag_uuid)
        if tag:
            tag_value = (tag.get(field) or '').strip()
            if tag_value:
                return tag_value, tag.get('title', 'tag')  # 第一个有值的标签获胜

    return '', ''
```

**代码依据（resolve_intent 专用解析）**：
```python
# changedetectionio/llm/evaluator.py:176-192
def resolve_intent(watch, datastore) -> tuple[str, str]:
    intent = (watch.get('llm_intent') or '').strip()
    if intent:
        return intent, 'watch'  # Watch 优先级最高

    # 不检查 overrides_watch，直接遍历标签找第一个有值的
    for tag_uuid in watch.get('tags', []):
        tag = datastore.data['settings']['application'].get('tags', {}).get(tag_uuid)
        if tag:
            tag_intent = (tag.get('llm_intent') or '').strip()
            if tag_intent:
                return tag_intent, tag.get('title', 'tag')  # 第一个有值的标签获胜

    return '', ''
```

---

### 2.4 机制三：restock 条件覆盖（需要 overrides_watch=True）

**覆盖原则**：`Watch.field → Tag.field (if overrides_watch) → 默认值`

**关键特征**：
1. ✅ **必须检查** `overrides_watch == True` 才生效
2. 🎯 **第一个匹配优先**：遍历标签列表，第一个满足条件的标签获胜
3. 🔄 **完全替换**：标签值完全替换 Watch 值，不是合并
4. 📋 适用于**复杂对象**配置（processor_config_restock_diff）

**需要 overrides_watch 的属性列表**：

| 属性名 | 检查位置 | 覆盖策略 |
|--------|----------|----------|
| `processor_config_restock_diff` | `processors/restock_diff/processor.py:464` | 第一个 `overrides_watch=True` 标签的配置完全替换 Watch 配置 |
| `processor_config_restock_diff` | `api/Watch.py:122` | GET /watch 时注入标签覆盖配置 |

**代码依据（restock_diff 条件覆盖）**：
```python
# changedetectionio/processors/restock_diff/processor.py:461-467
# See if any tags have 'activate for individual watches in this tag/group?' enabled and use the first we find
for tag_uuid in watch.get('tags'):
    tag = self.datastore.data['settings']['application']['tags'].get(tag_uuid, {})
    if tag.get('overrides_watch'):  # 必须检查 overrides_watch
        restock_settings = tag.get('processor_config_restock_diff') or {}
        logger.info(f"Watch {watch.get('uuid')} - Tag '{tag.get('title')}' selected for restock settings override")
        break  # 第一个匹配即停止
```

**代码依据（API 层注入）**：
```python
# changedetectionio/api/Watch.py:119-127
tags = self.datastore.data['settings']['application'].get('tags', {})
for tag_uuid in (watch_obj.get('tags') or []):
    tag = tags.get(tag_uuid, {})
    if tag.get('overrides_watch'):  # 必须检查 overrides_watch
        restock_config = dict(tag.get('processor_config_restock_diff') or {})
        restock_source = f'tag:{tag_uuid}'
        break
```

---

### 2.5 三种覆盖路径完整对照表

| 对比维度 | get_tag_overrides_for_watch 列表合并 | LLM 单值级联匹配 | restock 条件覆盖 |
|---------|-------------------------------------|-----------------|------------------|
| **检查 overrides_watch** | ❌ 不检查 | ❌ 不检查 | ✅ 必须检查且为 True |
| **标签匹配策略** | 所有标签都参与合并 | 第一个有非空值的标签获胜 | 第一个 `overrides_watch=True` 的标签获胜 |
| **数据类型** | 列表类型（数组） | 单值字符串 | 复杂对象、字典 |
| **Watch 与标签关系** | Watch 列表 + 所有标签列表，去重合并 | Watch 有值则用 Watch，否则回退标签 | 标签完全覆盖 Watch 配置 |
| **生效标签数量** | 所有关联标签都生效 | 仅第一个有值标签生效 | 仅第一个满足条件标签生效 |
| **典型属性** | include_filters, subtractive_selectors, ignore_text | llm_intent, llm_change_summary | processor_config_restock_diff |
| **实现位置** | store/__init__.py:936 | llm/evaluator.py:156-192 | processors/restock_diff/processor.py:461 |
| **合并/覆盖模式** | 合并（Union） | 级联回退（Fallback） | 覆盖（Override） |

---

## 3. 队列处理链路标签感知四步分析

### 3.0 队列架构总览

```
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │   入 队     │────▶│   取 队     │────▶│ Worker 处理 │────▶│  Diff 参数  │
  │   Enqueue   │     │   Dequeue   │     │  Process    │     │  Resolve   │
  └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
         │                   │                   │                   │
         ▼                   ▼                   ▼                   ▼
   🔍 按标签筛选        📦 队列无标签感知    🎯 动态解析配置        ⚙️  合并选择器
      仅过滤，不修改         纯优先级队列         overrides_watch        get_tag_overrides
```

---

### 3.1 第一步：入队（Enqueue）- 标签用于过滤筛选

**触发场景**：
1. 按标签批量重新检查（`GET /api/v1/tag/<uuid>?recheck=true`）
2. Watch 列表按标签过滤后批量重检查

**入队时标签的作用**：
- ✅ **仅用于过滤**：决定哪些 Watch 进入队列
- ❌ **不修改队列项**：入队的 `PrioritizedItem` 不含标签信息
- ❌ **不影响优先级**：优先级由其他因素决定，与标签无关

**代码依据**：
```python
# changedetectionio/api/Tags.py:19-65
if request.args.get('recheck'):
    # Recheck all watches with this tag, including muted
    watches_to_queue = []
    for k in sorted(self.datastore.data['watching'].items(), key=lambda item: item[1].get('last_checked', 0)):
        watch_uuid = k[0]
        watch = k[1]
        # 🔍 仅在这里用标签过滤，标签信息不进入队列
        if not watch['paused'] and tag['uuid'] in watch['tags']:
            watches_to_queue.append(watch_uuid)

    # 入队的只有 watch_uuid，没有标签信息
    for watch_uuid in watches_to_queue:
        worker_pool.queue_item_async_safe(self.update_q, queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```

**关键结论**：入队阶段标签**仅作为过滤条件**，不进入队列数据结构。

---

### 3.2 第二步：取队（Dequeue）- 队列完全不感知标签

**队列核心组件**：`RecheckPriorityQueue` - 基于优先级的线程安全队列

**取队行为**：
- 📦 **纯优先级调度**：完全根据 `item.priority` 决定出队顺序
- ❌ **不感知标签**：队列内部逻辑与标签系统完全解耦
- 🔒 **无标签过滤**：取队时不再检查标签，也不会因标签改变优先级

**代码依据**：
```python
# changedetectionio/queue_handlers.py:64-100
def put(self, item, block: bool = True, timeout: Optional[float] = None):
    """Thread-safe sync put with priority ordering"""
    logger.trace(f"RecheckQueue.put() called for item: {self._get_item_uuid(item)}, block={block}, timeout={timeout}")
    try:
        # CRITICAL: Add to both priority storage AND notification queue atomically
        # 仅用 priority 排序，完全忽略标签信息
        with self._lock:
            heapq.heappush(self._priority_items, item)  # 🔑 只看 priority
```

```python
# changedetectionio/queue_handlers.py:102-134
def get(self, block: bool = True, timeout: Optional[float] = None):
    """Thread-safe sync get with priority ordering"""
    # 等待通知后直接取最高优先级项，不检查标签
    with self._lock:
        if not self._priority_items:
            raise Exception("Priority queue inconsistency")
        item = heapq.heappop(self._priority_items)  # 🔑 只看 priority
```

**关键结论**：队列调度器**完全不感知标签**，标签系统与队列系统在这一层完全解耦。

---

### 3.3 第三步：Worker 处理 - 动态解析标签配置

**执行时机**：Worker 从队列获取 Watch UUID 后，加载 Watch 对象时开始解析标签

**标签感知点**：
1. 🔄 **配置覆盖解析**：检查 `overrides_watch` 条件覆盖
2. 🧹 **校验和清理**：标签配置变更后清理校验和强制重处理
3. 📋 **处理器配置注入**：restock_diff 等处理器获取标签级配置

**代码依据（Worker 主流程）**：
```python
# changedetectionio/worker.py:147-180
if uuid in list(datastore.data['watching'].keys()) and datastore.data['watching'][uuid].get('url'):
    changed_detected = False
    contents = b''
    process_changedetection_results = True
    update_obj = {}

    watch = datastore.data['watching'].get(uuid)

    # Processor is what we are using for detecting the "Change"
    processor = watch.get('processor', 'text_json_diff')

    # Init a new 'difference_detection_processor'
    from changedetectionio.processors import get_processor_module
    processor_module = get_processor_module(processor)

    if not processor_module:
        error_msg = f"Processor module '{processor}' not found."
        logger.error(error_msg)
        raise ModuleNotFoundError(error_msg)

    update_handler = processor_module.perform_site_check(datastore=datastore,
                                                         watch_uuid=uuid)
    # 🎯 处理器初始化后，在其内部动态解析标签配置
```

**代码依据（标签变更触发重处理）**：
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
                if os.path.isfile(checksum_file):
                    try:
                        os.remove(checksum_file)
                        deleted_count += 1
                        logger.debug(f"Cleared checksum for watch {uuid}")
                    except OSError as e:
                        logger.warning(f"Failed to delete checksum file for {uuid}: {e}")
```

**关键结论**：Worker 处理阶段是标签配置**真正生效**的地方，通过处理器内部动态解析实现。

---

### 3.4 第四步：Diff 参数 - 标签选择器最终生效

**执行时机**：差异计算前，构建内容提取规则时

**标签在 Diff 阶段的作用**：
- ⚙️ **选择器合并**：通过 `get_tag_overrides_for_watch` 合并所有标签的选择器
- ❌ **不直接调用 diff**：标签不直接参与 diff 算法本身
- 📥 **间接影响输入**：标签通过改变提取规则间接影响 diff 的输入内容

**代码依据（选择器合并）**：
```python
# changedetectionio/processors/text_json_diff/processor.py:79-86
@property
def subtractive_selectors(self):
    if self._subtractive_selectors_cache is None:
        watch_selectors = self.watch.get("subtractive_selectors", [])
        # ⚙️  这里调用 get_tag_overrides_for_watch，直接合并所有标签的选择器
        tag_selectors = self.datastore.get_tag_overrides_for_watch(uuid=self.watch_uuid, attr='subtractive_selectors')
        global_selectors = self.datastore.data["settings"]["application"].get("global_subtractive_selectors", [])
        # 标签选择器 → Watch 选择器 → 全局选择器，按顺序合并
        self._subtractive_selectors_cache = [*tag_selectors, *watch_selectors, *global_selectors]
    return self._subtractive_selectors_cache
```

**代码依据（include_filters 合并）**：
```python
# changedetectionio/processors/text_json_diff/processor.py:69-77
@property
def include_filters(self):
    if self._include_filters_cache is None:
        # ⚙️ _get_merged_rules 内部调用 get_tag_overrides_for_watch
        filters = self._get_merged_rules('include_filters')
        # Inject LD+JSON price tracker rule if enabled
        if self.watch.get('track_ldjson_price_data', '') == PRICE_DATA_TRACK_ACCEPT:
            filters += html_tools.LD_JSON_PRODUCT_OFFER_SELECTORS
        self._include_filters_cache = filters
    return self._include_filters_cache
```

**差异计算纯函数特性验证**：
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
    # 🔑 纯函数：仅接收文本内容和计算参数，完全不感知标签存在
    # 标签已在上层通过改变选择器影响了输入文本内容
```

**关键结论**：Diff 模块本身是**纯函数**，标签通过**影响输入内容**（选择器过滤）间接发挥作用。

---

### 3.5 队列链路标签感知总结

| 阶段 | 标签感知程度 | 核心作用 | 关键机制 |
|------|-------------|----------|----------|
| **入队** | ⭐ 轻度 | 过滤筛选 | 检查 `tag in watch['tags']`，标签不进入队列 |
| **取队** | ❌ 无感知 | 纯优先级调度 | 队列与标签解耦，只按 priority 排序 |
| **Worker 处理** | ⭐⭐⭐ 重度 | 配置覆盖解析 | 检查 `overrides_watch`，处理器级配置覆盖 |
| **Diff 参数** | ⭐⭐ 中度 | 选择器合并 | `get_tag_overrides_for_watch` 合并过滤规则 |

**标签信息流转路径**：
```
Watch['tags'] = [uuid1, uuid2]
       ↓ (入队过滤)
  PrioritizedItem = {uuid: watch_uuid}
       ↓ (取队无标签)
  Worker 加载 Watch 对象
       ↓ (处理阶段)
  ├─→ restock_diff: 检查 overrides_watch，覆盖配置
  └─→ text_json_diff: get_tag_overrides_for_watch 合并选择器
       ↓ (选择器过滤文本)
  纯文本内容 → render_diff() → 差异结果
```

---

## 4. 差异计算模块与标签的关系

### 4.1 差异计算模块架构

**核心模块**：`changedetectionio/diff/__init__.py`

**主要函数**：
| 函数 | 职责 |
|------|------|
| `render_diff()` | 生成差异HTML输出 |
| `customSequenceMatcher()` | 序列比较核心逻辑 |
| `render_inline_word_diff()` | 单词级内联差异渲染 |
| `render_nested_line_diff()` | 嵌套行级差异渲染 |

### 4.2 关键结论：差异计算模块不直接感知标签

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

### 4.3 标签对差异计算的间接影响路径

**影响链**：标签通过配置覆盖机制间接影响差异计算的**输入参数**

```
标签配置 → Watch 配置覆盖 → 处理器设置 → 差异计算输入参数 → 计算结果
```

**具体影响的配置项**：

| 配置项 | 影响方式 | 代码位置 |
|--------|----------|----------|
| `include_filters` | 控制提取哪些内容进行比较 | `processors/text_json_diff/processor.py:72` |
| `subtractive_selectors` | 控制排除哪些内容 | `processors/text_json_diff/processor.py:83` |
| `ignore_text` | 忽略特定文本变化 | `processors/text_json_diff/processor.py:98` |
| `extract_lines_containing` | 仅提取包含特定内容的行 | `processors/text_json_diff/processor.py:90` |
| `extract_text` | 自定义文本提取规则 | `processors/text_json_diff/processor.py:94` |

**代码依据（标签覆盖检查）**：
```python
# changedetectionio/blueprint/ui/edit.py:39-43
def _watch_has_tag_options_set(watch):
    for tag_uuid, tag in datastore.data['settings']['application'].get('tags', {}).items():
        if tag_uuid in watch.get('tags', []) and (tag.get('include_filters') or tag.get('subtractive_selectors')):
            return True
```

### 4.4 差异计算的纯函数特性验证

**输入仅包含**：
1. 前一版本文本内容
2. 当前版本文本内容
3. 计算选项（布尔标志、字符串参数）

**输出仅包含**：
- 差异结果HTML或补丁格式

**无外部依赖**：不访问 datastore、不读取标签信息、无副作用

---

## 5. 架构总结与代码索引

### 5.1 标签功能分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│  表现层 (UI/API)                                                  │
│  - 标签过滤列表展示                                                │
│  - 按标签批量操作                                                  │
│  - watchlist/__init__.py, api/Tags.py                             │
├─────────────────────────────────────────────────────────────────┤
│  业务逻辑层 (处理器/Worker)                                        │
│  - 配置覆盖解析：overrides_watch 检查                             │
│  - 选择器合并：get_tag_overrides_for_watch                        │
│  - worker.py, processors/*                                        │
├─────────────────────────────────────────────────────────────────┤
│  存储层 (数据模型/持久化)                                           │
│  - Tag 模型 / TagsDict 集合管理                                    │
│  - Watch-Tag 关联 (watch['tags'] = [UUIDs])                       │
│  - model/Tag.py, model/Tags.py, store/__init__.py                │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 关键代码位置索引

| 功能点 | 文件路径 | 行号 |
|--------|----------|------|
| **标签核心模型** | | |
| 标签模型定义 | `changedetectionio/model/Tag.py` | 1-71 |
| 标签集合管理 | `changedetectionio/model/Tags.py` | 1-39 |
| 统一持久化 Mixin | `changedetectionio/model/persistence.py` | 1-84 |
| 获取 Watch 的所有标签 | `changedetectionio/store/__init__.py` | 980-996 |
| **标签覆盖规则** | | |
| 标签属性直接合并函数 | `changedetectionio/store/__init__.py` | 936-945 |
| 选择器合并逻辑 | `changedetectionio/processors/text_json_diff/processor.py` | 57-98 |
| restock 条件覆盖 | `changedetectionio/processors/restock_diff/processor.py` | 461-467 |
| API 注入标签覆盖配置 | `changedetectionio/api/Watch.py` | 119-127 |
| **队列处理链路** | | |
| 按标签批量重新检查 | `changedetectionio/api/Tags.py` | 19-65 |
| 队列入队实现 | `changedetectionio/queue_handlers.py` | 64-100 |
| 队列取队实现 | `changedetectionio/queue_handlers.py` | 102-134 |
| 清除标签关联 Watch 校验和 | `changedetectionio/store/__init__.py` | 499-520 |
| Worker 主处理流程 | `changedetectionio/worker.py` | 46-743 |
| **差异计算** | | |
| 差异计算核心函数 | `changedetectionio/diff/__init__.py` | 424-507 |
| 标签覆盖配置检查 | `changedetectionio/blueprint/ui/edit.py` | 39-43 |
| 标签 URL 匹配方法 | `changedetectionio/model/Tag.py` | 55-67 |

### 5.3 设计特点总结

| 特点 | 说明 |
|------|------|
| **关注点分离** | 差异计算是纯函数，标签逻辑在上层处理，便于测试和维护 |
| **双轨覆盖机制** | 列表类属性直接合并，复杂配置需要 overrides_watch 条件覆盖 |
| **松耦合架构** | 队列调度器不需要理解标签，标签逻辑集中在处理器业务层 |
| **统一持久化** | Tag 和 Watch 复用相同的持久化机制，减少代码重复 |
| **纯函数设计** | 差异计算无外部依赖，保证了可测试性和可预测性 |
| **分层生效** | 标签在队列链路各阶段逐步发挥作用，而非集中处理 |

---

**报告生成时间**：2026-05-16
**代码版本**：commit 130-changedetection.io
**分析深度**：覆盖存储层、队列链路、标签覆盖规则、差异计算模块
