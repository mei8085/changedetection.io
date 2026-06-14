# changedetection.io —— processor_config_* 从表单到磁盘的真实路径

> 聚焦 UI POST 提交链路，搞清楚 Watch 重建、二次 update、was_edited 污染、引用窗口期这几个核心细节。

---

## 1. watch_class 重建时 __init__ 二次 update(default) 的作用

### 1.1 为什么每次保存都要重建 Watch？

在 [edit.py#L233-L235](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L233-L235)：

```python
# Recast it if need be to right data Watch handler
watch_class = processors.get_custom_watch_obj_for_processor(form.data.get('processor'))
datastore.data['watching'][uuid] = watch_class(
    datastore_path=datastore.datastore_path,
    __datastore=datastore.data,
    default=datastore.data['watching'][uuid]
)
```

**每次保存都重建**，即使 processor 没变。原因是：
- 不同 processor 可能有不同的 Watch 子类（如 restock_diff 可能有自己的 Watch.model）
- 子类可能覆盖了 `_get_commit_data()`、`history`、`clear_watch()` 等方法
- 必须保证运行时的 Watch 对象是正确的类型

但在实际代码中，大部分 processor 没有自定义 Watch 类，`get_custom_watch_obj_for_processor` 返回的就是默认的 `Watch.model`，所以重建前后是同一个类——仍然会重建一遍。

### 1.2 构造函数中的两次 update

Watch 对象的构造经过了**两层继承链**，每层 `__init__` 都做了一次 `update()`：

```
Watch.model.__init__
    └─ watch_base.__init__
        └─ dict.__init__
```

#### 第一次 update（watch_base.__init__）——默认值打底

[watch_base.__init__ #L174-L306](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/__init__.py#L174-L306)：

```python
self.update({
    'body': None,
    'browser_steps': [],
    'check_count': 0,
    'consecutive_filter_failures': 0,
    # ... 50+ 个字段的默认值
    'time_between_check': {'weeks': None, 'days': None, ...},
    'processor': 'text_json_diff',
    # ...
})
```

这一步用一个包含所有字段默认值的大字典 update 当前 watch（dict 子类），确保所有字段都有合理的默认值。

**关键点**：此时 `__watch_was_edited` 属性还未创建，`_mark_field_as_edited` 检测到属性不存在就直接 return，**这次 update 不会标记 was_edited**。

#### dict.__init__(**kw)——透传 keyword 参数

[watch_base.__init__ #L308](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/__init__.py#L308)：

```python
super(watch_base, self).__init__(*arg, **kw)
```

`kw` 中此时还剩 `default=old_watch`（因为 `__datastore` 和 `datastore_path` 已经在前面被 pop 掉了）。

`dict.__init__(default=old_watch)` 会在 dict 中添加一个 key 为 `'default'`、值为 `old_watch` 的条目——只是作为一个临时的传递通道，后面会被删除。

#### was_edited 标志保留

[watch_base.__init__ #L311-L324](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/__init__.py#L311-L324)：

```python
if self.get('default'):
    default_watch = self.get('default')
    if hasattr(default_watch, 'was_edited') and default_watch.was_edited:
        preserve_edited_flag = True
    del self['default']

self.__watch_was_edited = preserve_edited_flag
```

这一步：
1. 从 dict 中取出 `default` key（也就是旧 watch 对象）
2. 检查旧 watch 的 `was_edited` 标志，如果是 True，新 watch 也继承为 True
3. 删除 `'default'` 这个临时 key
4. **正式创建** `__watch_was_edited` 属性

**注意**：`default` key 在这里被删除了，但 `kw['default']` 还在——kw 参数本身没有被修改。

#### 第二次 update（Watch.model.__init__）——真正的数据迁移

[Watch.model.__init__ #L242-L247](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py#L242-L247)：

```python
if kw.get('default'):
    self.update(kw['default'])    # 关键！从旧 watch 复制所有数据
    del kw['default']

if self.get('default'):
    del self['default']           # 清理可能残留的 default key
```

这是**真正的数据迁移**：把旧 watch 的所有字段值全部 update 到新 watch 中。

**关键点**：
- 此时 `__watch_was_edited` 已经存在（在 watch_base.__init__ 末尾创建）
- `self.update(kw['default'])` 会为每个 key 调用 `_mark_field_as_edited`
- 第一个非只读、非系统字段就会把 `__watch_was_edited` 设为 True（除非 preserve_edited_flag 已经是 True）

### 1.3 子类构造差异

| 类 | 第一次 update | 第二次 update | 用途 |
|---|-------------|-------------|------|
| watch_base | 有（默认值字典） | 无 | 基类，不单独使用 |
| Watch.model | 继承 | **有**（`self.update(kw['default'])`） | 监控项主模型 |
| Tag.model | 继承 | **有**（`self.update(kw['default'])`） | 标签模型 |

Watch 和 Tag 都有第二次 update，逻辑完全一致——都是从 `default` 参数拷贝数据。

区别在于：
- Watch 的第二次 update 在 `super().__init__` 之后
- Tag 的第二次 update 之前还单独处理了 `overrides_watch` 和 `url_match_pattern`（可能是历史遗留）

---

## 2. watch_base.update 标 writable 是否污染 was_edited

### 2.1 _mark_field_as_edited 的判断逻辑

[watch_base._mark_field_as_edited #L326-L354](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/__init__.py#L326-L354)：

```python
def _mark_field_as_edited(self, key):
    # 1. 属性不存在 → 直接返回（初始化阶段保护）
    if not hasattr(self, '_watch_base__watch_was_edited'):
        return

    # 2. 已经是 True → 直接返回（短路）
    if self.__watch_was_edited:
        return

    # 3. __ 前缀的瞬态字段 → 跳过
    if isinstance(key, str) and key.startswith('__'):
        return

    # 4. 不在只读字段集 AND 不是 last_viewed AND 不在系统管理字段集 → 标记为已编辑
    if (key not in get_readonly_watch_fields()
            and key != 'last_viewed'
            and key not in SYSTEM_MANAGED_NON_SPEC_FIELDS):
        self.__watch_was_edited = True
```

**三类字段不会触发 was_edited**：

| 类别 | 例子 | 来源 |
|-----|------|------|
| 只读字段 | `uuid`, `date_created`, `last_checked`, `previous_md5` | OpenAPI spec 中的 `readOnly: true` |
| last_viewed | `last_viewed` | 特殊处理：内部设置但 API 也可写 |
| 系统管理字段 | `last_check_status`, `last_filter_config_hash`, `restock`, `_llm_result`, `llm_evaluation_cache` 等 | `SYSTEM_MANAGED_NON_SPEC_FIELDS` 常量 |

### 2.2 processor_config_* 会触发 was_edited 吗？

**会。**

`processor_config_*` 字段：
- 不在 `get_readonly_watch_fields()` 中（OpenAPI spec 里没有这些字段）
- 不是 `last_viewed`
- 不在 `SYSTEM_MANAGED_NON_SPEC_FIELDS` 中（那些是处理器运行时状态，不是配置）

所以当 `form.data` 中的 `processor_config_*` 被 update 进 Watch 时，第一个这样的字段就会把 `was_edited` 设为 True。

### 2.3 重建过程中的 was_edited 污染

Watch 重建时的第二次 update（从旧 watch 拷贝数据）**一定会导致 was_edited=True**，除非：

1. 旧 watch 的 was_edited 已经是 True（preserve_edited_flag 直接设为 True，跳过后面的）
2. 旧 watch 里的所有可写字段都不存在（不可能，因为默认值就有很多）

**但这是故意设计的**：

从 `worker.py` 中 `was_edited` 的用途可以看出设计意图：
- `was_edited=True` 表示"配置变了，下次检查即使内容没变也需要重新处理"
- 每次保存都重建 Watch，而重建后 was_edited=True → 下次检查会强制重处理
- 这和"用户点了保存，配置应该被视为已修改"的直觉一致

即使实际上表单值一个字都没改，保存后 was_edited 也会是 True。这是为了**正确性而牺牲效率**——宁可多处理一次，也不能漏掉应该处理的情况。

### 2.4 was_edited 的重置时机

在 [worker.py#L308](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/worker.py#L308) 和 [worker.py#L429](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/worker.py#L429)：

```python
watch.reset_watch_edited_flag()
```

worker 成功处理一次检查后重置标志。这意味着：
- 保存 → was_edited = True
- worker 运行检查 → 处理完成 → was_edited = False
- 之后如果内容没变，可以跳过处理

---

## 3. 新旧 Watch 引用窗口期

### 3.1 UI POST 路径上的引用时间线

```
时间点 | 动作 | 旧 Watch 引用 | 新 Watch 引用
-------|------|-------------|-------------
 T0    | 请求进入 edit_page() | datastore.data['watching'][uuid] | -
 T1    | default = deepcopy(...) | 原始 + default(深拷贝) | -
 T2    | form = form_class(data=default) | 原始 + default | -
 T3    | old_watch.update(form.data) | 原始（已被修改） + default | -
 T4    | old_watch.update(extra_update_obj) | 原始（已被修改） + default | -
 T5    | 开始执行 watch_class(...) | 原始（作为 default 参数） | 正在构造中
 T6    | watch_base.__init__ 完成 | 原始（仍在 datastore 中） | 半成品（有默认值）
 T7    | self.update(kw['default']) 数据拷贝 | 原始（只读访问） | 半成品（数据拷贝中）
 T8    | Watch.model.__init__ 完成 | 原始 | 新 Watch 对象构造完成
 T9    | datastore.data['watching'][uuid] = 新watch | 原始（引用数 -1） | 正式入驻 datastore
 T10   | 新 watch.commit() | 原始（等待 GC） | 提交到磁盘
 T11   | 请求结束 | 原始（可被 GC） | 仍在 datastore 中
```

### 3.2 谁持有旧 Watch 的引用

在整个 UI POST 处理过程中，旧 Watch 对象最多时有 **3 个引用**同时存在：

| 引用位置 | 生命周期 | 说明 |
|---------|---------|------|
| `datastore.data['watching'][uuid]` | T0 ~ T9 | datastore 中的正式引用，T9 被替换 |
| `default = deepcopy(...)` | T1 ~ 函数返回 | GET 时用于初始化表单，POST 时也会创建但可能用不上？需要确认 |
| `kw['default']` / `self['default']` | T5 ~ T7 | 构造函数参数传递通道，T7 被删除 |

等等，让我再确认一下 `default = deepcopy(datastore.data['watching'][uuid])` 在 POST 时是否也会创建。

看 [edit.py#L75-L79](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L75-L79)：

```python
# be sure we update with a copy instead of accidently editing the live object by reference
default = None
while not default:
    try:
        default = deepcopy(datastore.data['watching'][uuid])
    except RuntimeError as e:
        continue
```

**无论 GET 还是 POST，这个 deepcopy 都会执行。** 所以在 POST 路径上也有一份 deepcopy 的副本。

但 POST 路径下 `form = form_class(formdata=request.form, data=default, ...)` 时，因为 `formdata` 不为 None，`data=default` 只在 formdata 为空时才使用（WTForms 的设计）。所以 POST 时 default 对象主要用于：
- 提供 `extra_notification_tokens` 等额外参数
- 可能作为表单验证时的参考

实际上 POST 时 `form.data` 的值主要来自 `request.form`，不是 default。

### 3.3 关键窗口期：T5 ~ T9（构造过程中）

在 Watch 构造过程中，新旧对象同时存在：
- 旧对象：作为数据来源被读取
- 新对象：正在被填充

这个窗口期中需要注意的问题：

1. **深拷贝 vs 浅拷贝**：`self.update(kw['default'])` 是 dict.update 的语义——对于 dict、list 这样的可变对象，拷贝的是引用。但好在 `default` 是从 datastore 中的原始 watch 传过来的，而原始 watch 后面会被替换掉，不会再被修改。

2. **数据一致性**：在 T3-T4 阶段，旧 watch 已经被 update 了 form.data 和 extra_update_obj。所以 T5 开始构造时，旧 watch 已经是"最新状态"，数据是一致的。

3. **was_edited 传递**：watch_base.__init__ 中检查旧 watch 的 was_edited 并传递给新 watch，保证标志不会丢失。

### 3.4 旧 Watch 何时可被 GC？

理论上 T9 之后（datastore 引用被替换），如果没有其他引用，旧 Watch 就可以被垃圾回收了。

但有一个潜在的长期引用：**信号（signal）系统**。blinker signal 的订阅者可能持有 watch 引用。不过从代码来看，信号发送时是传 watch_uuid 而不是 watch 对象，所以问题不大。

另外，`default = deepcopy(...)` 那个副本在请求结束后也会被 GC——它只是函数局部变量。

---

## 4. processor_config_* 从 form 到 commit 的完整真实路径

### 4.1 全景流程图

```
表单层 (WTForms)
  │
  │  form.processor_config_min_change_percentage.data = 10
  │  form.processor_config_restock_diff.data = { ... }
  │
  ▼
form.data 第 1 次访问 (L194)
  │
  │  { 'url': '...', 'processor': 'image_ssim_diff',
  │    'processor_config_min_change_percentage': 10, ... }
  │
  ├─→ extract_processor_config_from_form_data(form.data)
  │     ├─ 提取: {'min_change_percentage': 10, ...}
  │     └─ 从"字典 A"中删除 processor_config_* key
  │                          ↑
  │                          └── 只影响第 1 次访问返回的字典
  │
  ├─→ save_processor_config()  →  写入 {processor}.json
  │     （独立的配置文件路径）
  │
  ▼
form.data 第 9 次访问 (L225) ← 全新字典，processor_config_* 又回来了！
  │
  │  { 'url': '...', 'processor': 'image_ssim_diff',
  │    'processor_config_min_change_percentage': 10, ... }
  │
  ▼
旧 Watch.update(form.data)  ← processor_config_* 进入内存
  │
  │  内存中的旧 Watch: { ..., 'processor_config_min_change_percentage': 10, ... }
  │  同时触发 was_edited = True
  │
  ▼
旧 Watch.update(extra_update_obj)
  │
  ▼
新 Watch 构造 (watch_class(default=old_watch))
  │
  ├─ watch_base.__init__ → 第一次 update（默认值，不触发 was_edited）
  ├─ 保留旧 was_edited 标志
  └─ Watch.model.__init__ → 第二次 update（从旧 watch 拷贝所有数据）
        │
        └─ 新 Watch 内存中也有 processor_config_*
        └─ 触发 was_edited = True（如果还不是的话）
  │
  ▼
datastore.data['watching'][uuid] = 新 Watch
  │
  ▼
新 Watch.commit()
  │
  ├─ _get_commit_data() ← 真正的拦截层
  │     └─ { k: v for k, v in snapshot.items()
  │          if not k.startswith('processor_config_')
  │          and not k.startswith('__') }
  │
  └─ _save_to_disk(data_dict) → 写入 watch.json
         （不含 processor_config_*）
```

### 4.2 三层拦截的真实效果

| 拦截层 | 位置 | UI POST 实际效果 | API PUT 实际效果 |
|-------|------|-----------------|-----------------|
| 第 1 层 | `extract_processor_config_from_form_data()` in-place del | ❌ **无效**（form.data 每次新建字典） | ✅ 有效（同一个 json_data 变量） |
| 第 2 层 | 内存 Watch → `_get_commit_data()` 过滤 | ✅ **有效**（磁盘写入前必经过） | ✅ 有效 |
| 第 3 层 | datastore 重启时从磁盘加载 | ✅ 有效（被动生效） | ✅ 有效 |

**在 UI POST 路径上，真正 100% 可靠的拦截层是 `Watch._get_commit_data()`。**

### 4.3 内存中的幽灵数据

虽然磁盘上干净了，但运行时内存中的 Watch 对象**确实包含** `processor_config_*` 字段。

这会导致什么问题吗？

**一般不会**，因为：
1. 处理器自己的配置从 `{processor}.json` 读，不从 watch dict 读
2. `_get_commit_data()` 保证了磁盘上不会有这些字段
3. datastore 重启后，内存中的这些字段会自然消失

**潜在风险**：
- 如果有代码直接从 `watch['processor_config_xxx']` 读取，可能拿到"幽灵数据"
- 但正常情况下应该通过 `processor_instance.get_extra_watch_config()` 读取
- deepcopy 操作会把这些字段也复制过去（在 `__deepcopy__` 中遍历 `self.items()` 全部复制）
- pickle 序列化也会包含这些字段（`__getstate__` 中 `state = dict(self)`）

### 4.4 独立配置路径（旁支）

除了主路径（form → Watch → watch.json），`processor_config_*` 还有一条独立的旁支路径：

```
form.data 第 1 次访问
    ↓
extract_processor_config_from_form_data()
    ↓
processor_config_data = {'min_change_percentage': 10, ...}
    ↓
save_processor_config(datastore, uuid, processor_config_data)
    ↓
difference_detection_processor(datastore, uuid)  # 构造处理器实例
    ↓
update_extra_watch_config(f'{processor_name}.json', config_data)
    ↓
merge=True → 读已有 JSON → update → 写回
    ↓
{uuid}/{processor_name}.json  ← 独立的配置文件
```

这条路径和 Watch 对象的生命周期**完全解耦**：
- 配置文件在 Watch 数据目录下，和 watch.json 并列
- 切换 processor 时，旧的 JSON 不删，新的创建/更新
- 清历史时，processor JSON 特意被保留

---

## 5. 总结

### 5.1 关键结论

| 问题 | 答案 |
|-----|------|
| 为什么每次保存都重建 Watch？ | 保证 processor 对应的 Watch 子类正确，即使是同一类也重建 |
| 为什么有两次 update？ | 第一次（watch_base）填默认值；第二次（Watch/Tag 子类）从旧 watch 拷贝数据 |
| was_edited 在重建时会被污染吗？ | 会。第二次 update 会标记为 True。但这是预期行为——保存即视为"已编辑" |
| processor_config_* 会触发 was_edited 吗？ | 会。它们不是只读、不是系统管理字段，属于"可写"字段 |
| 旧 Watch 引用何时消失？ | datastore 引用替换后（T9）可被 GC，窗口期约几十毫秒 |
| UI POST 上 processor_config_* 的真实拦截层？ | `Watch._get_commit_data()`，第 2 层才是真拦截 |

### 5.2 代码索引

| 机制 | 文件 | 行号 |
|-----|------|------|
| Watch 重建入口 | [edit.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py) | L233-L235 |
| watch_base.__init__ 第一次 update | [__init__.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/__init__.py) | L174-L306 |
| was_edited 标志保留 | [__init__.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/__init__.py) | L311-L324 |
| Watch.model.__init__ 第二次 update | [Watch.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py) | L242-L247 |
| _mark_field_as_edited 逻辑 | [__init__.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/__init__.py) | L326-L354 |
| SYSTEM_MANAGED_NON_SPEC_FIELDS | [schema_utils.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/schema_utils.py) | L21-L32 |
| 真实拦截层 _get_commit_data | [Watch.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py) | L1064-L1093 |
| save_processor_config 旁支 | [processors/__init__.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/__init__.py) | L428-L469 |
