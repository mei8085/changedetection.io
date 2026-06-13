# changedetection.io 监控项编辑——form.data 语义与 processor_config_* 真实拦截层

> **TL;DR**：WTForms 3.2 的 `form.data` **每次访问都返回全新构造的字典**。这意味着 `extract_processor_config_from_form_data(form.data)` 对 UI POST 路径上第 1 次访问返回的字典做的 in-place del，对第 9 次访问（L225 `watch.update(form.data)`）的结果**完全没有影响**。真正的拦截层是 `Watch._get_commit_data()` 在写入磁盘前的防御性过滤。

---

## 1. form.data property 的真实语义

### 1.1 WTForms 3.2 源码

WTForms 3.2.x `BaseForm` 中 `data` property 的实现（[wtforms.readthedocs.io 源码](https://wtforms.readthedocs.io/en/3.2.x/_modules/wtforms/form/)）：

```python
@property
def data(self):
    return {name: f.data for name, f in self._fields.items()}
```

这是一个**完全没有缓存的 property**，每次访问都会：
1. 遍历 `self._fields`（OrderedDict，包含所有已定义字段）
2. 对每个字段读取 `f.data`
3. 构造一个全新的 Python `dict` 字面量并返回

**没有任何 `@cached_property`、`lru_cache` 或实例级缓存。**

### 1.2 语义验证

WTForms 1.0 文档就已明确标注了这个行为（[wtforms.simplecodes.com](http://wtforms.simplecodes.com/docs/1.0.1/forms.html)）：

> "Note that this is **generated each time you access the property**, so care should be taken when using it, as it can potentially be very expensive if you repeatedly access it."

3.2 版本保持了完全相同的实现，没有引入缓存。

### 1.3 对代码的直接影响

如果代码是：

```python
d1 = form.data          # 第 1 次访问 → 新字典 A
del d1['some_key']      # 从字典 A 中删除
d2 = form.data          # 第 2 次访问 → 全新字典 B，仍然包含 some_key！
```

`d1` 的修改完全不影响 `d2`，因为它们是两个完全独立的字典对象。

---

## 2. UI POST 路径上 form.data 的访问全景

在 [edit.py#L180-L238](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L180-L238) 的 UI POST 处理代码中，`form.data` 被访问了 **10 次**：

| # | 行号 | 代码 | 说明 |
|---|------|------|------|
| 1 | L194 | `extract_processor_config_from_form_data(form.data)` | 传入新字典 A，被 in-place 删除 processor_config_* |
| 2 | L202 | `form.data['proxy']` | 新字典 B，读取 proxy 值 |
| 3 | L207 | `form.data.get('filter_text_added')` | 新字典 C |
| 4 | L208 | `form.data.get('filter_text_replaced')` | 新字典 D |
| 5 | L209 | `form.data.get('filter_text_removed')` | 新字典 E |
| 6 | L216 | `form.data.get('tags')` | 新字典 F |
| 7 | L218 | `form.data.get('tags')` | 新字典 G |
| 8 | L221 | `form.data.get('tags')` | 新字典 H |
| 9 | L225 | `watch.update(form.data)` | **新字典 I，用于写入 Watch** |
| 10 | L234 | `form.data.get('processor')` | 新字典 J |

**关键结论**：L194 删除的是"字典 A"，而 L225 写入 Watch 的是完全独立的"字典 I"。两次访问之间相隔了 10+ 行代码，**in-place del 的影响在 L195 之后就已经完全消失了**。

### 2.1 模拟验证

假设表单有字段 `url`、`title`、`processor_config_min_change_percentage`：

```python
# 第 1 次访问
>>> d1 = form.data
>>> d1
{'url': 'https://example.com', 'title': 'My Watch',
 'processor_config_min_change_percentage': 10}

# extract 函数 in-place 修改 d1
>>> processor_config_data = extract_processor_config_from_form_data(d1)
>>> d1
{'url': 'https://example.com', 'title': 'My Watch'}
>>> processor_config_data
{'min_change_percentage': 10}

# 第 9 次访问（L225）—— 全新字典
>>> d9 = form.data
>>> d9
{'url': 'https://example.com', 'title': 'My Watch',
 'processor_config_min_change_percentage': 10}   # ← 字段又回来了！
```

`processor_config_min_change_percentage` 在 `d9` 中**仍然存在**，因为 WTForms 重新从各个 Field 实例的 `.data` 属性读取构造了新字典。

### 2.2 为什么 L195-L224 之间其他访问也"感觉没问题"

因为在 L202、L207-L209、L216-L221 这些地方访问的是 `proxy`、`filter_text_*`、`tags` 这些**非** `processor_config_*` 字段，所以即使是全新构造的字典，这些字段的值也总是存在的。代码没有在这些行再去访问任何 `processor_config_*` 字段，因此 in-place del 的失效被完全掩盖了。

---

## 3. extract_processor_config_from_form_data 的 in-place del 在 UI POST 路径上是否真有效？

**结论：对 UI POST 的 Watch.update() 写入完全无效；但对 processor_config_data 的提取是有效的。**

### 3.1 有效的部分

```python
processor_config_data = processors.extract_processor_config_from_form_data(form.data)
processors.save_processor_config(datastore, uuid, processor_config_data)
```

`processor_config_data` 的提取和保存**完全正常工作**——它是从第 1 次访问的字典 A 中正确提取的。

### 3.2 无效的部分

注释写的是：

```python
# IMPORTANT: These must NOT be saved to url-watches.json,
# only to the processor-specific JSON file
processor_config_data = processors.extract_processor_config_from_form_data(form.data)
```

注释的意图是"从 form.data 中剔除这些字段，防止它们被写入 Watch"，但实际上这个意图**没有实现**——因为后续 L225 的 `watch.update(form.data)` 使用的是全新构造的字典 I，里面仍然包含了所有 `processor_config_*` 字段。

**所以 L194 的 in-place del 在 UI POST 路径上只起到了"提取数据供保存"的作用，没有起到"拦截数据进入 Watch"的作用。**

### 3.3 API PUT 路径的对比

API PUT 路径 [api/Watch.py#L195-L198](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/api/Watch.py#L195-L198)：

```python
json_data = strip_internal_api_fields(dict(request.json))
processor_config_data = processors.extract_processor_config_from_form_data(json_data)
# ...
watch.update(json_data)    # 用的是同一个被 in-place 修改过的 json_data
```

这里用的是**同一个** `json_data` 字典变量，不是每次重新构造的 property。所以 in-place del 在 API PUT 路径上**完全有效**——`watch.update(json_data)` 拿到的就是已经剥离了 `processor_config_*` 的字典。

两条链路使用同一个函数，但实际效果天差地别。

---

## 4. update 写入的字典是否已剥离 processor_config_*？

**结论：在 UI POST 路径上，L225 `watch.update(form.data)` 写入的字典**并未被剥离**，`processor_config_*` 字段完整地进入了内存中的 Watch 对象。**

完整的数据流：

```
form.validate() 通过
    ↓
form.data 第 1 次访问 → 字典 A（含 processor_config_*）
    ↓
extract_processor_config_from_form_data(字典 A)
    ├── in-place del 字典 A 中的 processor_config_*（只影响字典 A）
    └── 返回 processor_config_data → save_processor_config() → 写入 {processor}.json
    ↓
form.data 第 9 次访问（L225）→ 字典 I（重新从 fields 构造，含 processor_config_*）
    ↓
watch.update(字典 I)
    ↓
内存中的 Watch 对象现在包含 processor_config_* 字段！
    ↓
watch.commit()
    ↓
Watch._get_commit_data() 再次从 Watch 内存对象复制 snapshot
    └── k.startswith('processor_config_') → 过滤掉，不写入磁盘
    ↓
watch.json 磁盘文件不含 processor_config_*  ✓
```

所以：
- **内存中 Watch 对象**：包含 `processor_config_*`（从 L225 写入后一直存在）
- **磁盘上 watch.json**：不含 `processor_config_*`（被 `_get_commit_data()` 拦截）
- **磁盘上 {processor}.json**：正确保存了配置

### 4.1 内存 vs 磁盘的不一致

在 commit 之后、下一次 datastore 重新加载之前：

```python
# 运行时内存中的 Watch 对象
>>> watch = datastore.data['watching'][uuid]
>>> 'processor_config_min_change_percentage' in watch
True  # ← 存在！

# 但磁盘上的 watch.json
>>> json.load(open(f'{uuid}/watch.json'))
>>> 'processor_config_min_change_percentage' in _
False  # ← 不存在
```

这种"内存有、磁盘无"的不一致，在 datastore 下次 reload（如进程重启）之后会自然消失——因为从磁盘加载时，`watch.json` 里本来就没有这些字段。

但在运行时，如果有代码直接从内存中的 Watch 对象读取（如 `watch.get('processor_config_xxx')`），可能会读到过时的值。

---

## 5. UI POST 路径下 processor_config_* 真正的拦截层

经过上面的分析，可以画出完整的拦截层次图：

```
┌─────────────────────────────────────────────────────────────────┐
│                     UI POST 提交处理流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  L180 form.validate()                                           │
│    └─ processor_config_* 字段参与 WTForms 验证                    │
│       （如 NumberRange、Optional 等）                              │
│                                                                 │
│  L194 extract_processor_config_from_form_data(form.data)        │
│    ├─ ✓ 正确提取 processor_config_data                           │
│    │     └─ 供 L195 save_processor_config() 写入 JSON            │
│    └─ ✗ 意图是剥离 form.data，实际只剥离了"字典 A"                │
│         不影响后续 form.data 访问返回的新字典                       │
│                                                                 │
│  L225 watch.update(form.data)  ← 全新字典 I，含 processor_config_* │
│    └─ ✗ 拦截失效：processor_config_* 进入了内存 Watch 对象          │
│                                                                 │
│  L238 watch.commit()                                            │
│    └─ Watch._get_commit_data()                                  │
│         └─ {k: v for k, v in snapshot.items()                   │
│             if not k.startswith('processor_config_')            │
│             and not k.startswith('__')}                         │
│              └─ ✓ 真正的拦截层：写入磁盘前过滤掉                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.1 真实拦截层次

| 层次 | 位置 | 实际效果 | 备注 |
|------|------|---------|------|
| 第 1 层（名义） | `extract_processor_config_from_form_data()` in-place del | **UI POST 无效**，API PUT 有效 | 注释认为它拦截了，但 UI 路径下 form.data 每次新建 |
| 第 2 层（真实） | `Watch._get_commit_data()` 序列化过滤 | **100% 有效** | 写入磁盘前的最后一道防线，是真正起作用的拦截层 |
| 第 3 层（隐式） | datastore 重启时从磁盘加载 | 自然生效 | 即使内存里有，重启后从磁盘读出来就没有了 |

### 5.2 为什么 `_get_commit_data()` 是"真正的拦截层"

因为它是**唯一对两条链路都生效、且在所有写入路径上都一定会经过**的检查点：

- UI POST 路径：经过（L238）
- API PUT 路径：经过（[api/Watch.py#L231](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/api/Watch.py#L231) `watch.commit()`）
- datastore.sync() 自动持久化：经过（`commit()` → `_get_commit_data()`）
- Watch 对象单独 commit：经过

无论前面的代码有没有正确剥离，`_get_commit_data()` 都保证了磁盘上的 `watch.json` 绝对不会包含 `processor_config_*` 字段。

### 5.3 代码注释中的自证

`_get_commit_data()` 的注释写得非常明确（[Watch.py#L1064-L1069](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py#L1064-L1069)）：

```python
def _get_commit_data(self):
    """
    Prepare watch data for commit.

    Excludes processor_config_* keys (stored in separate files).
    Excludes __-prefixed keys (transient in-memory state — must not persist to disk).
    """
```

"Excludes processor_config_* keys"——说明开发者**明确知道**这一层需要负责过滤这些字段，也侧面印证了上游可能存在"漏过滤"的情况。

---

## 6. 总结与改进建议

### 6.1 事实结论

1. **`form.data` 每次访问返回全新字典**：WTForms 3.2 的 `data` property 是无缓存的字典推导式。
2. **UI POST 路径的 in-place del 完全无效**：`extract_processor_config_from_form_data(form.data)` 只修改了第 1 次访问返回的字典，L225 的 `watch.update(form.data)` 用的是第 9 次访问的全新字典，仍然包含 `processor_config_*`。
3. **内存中 Watch 对象会包含 processor_config_***：从 L225 写入后到下一次 datastore reload 之前，这些字段存在于运行时内存中。
4. **真正的拦截层是 `Watch._get_commit_data()`**：写入磁盘前的防御性过滤是唯一可靠的保障。
5. **API PUT 路径的 in-place del 是有效的**：因为它用的是同一个 `json_data` 字典变量，不是 property。

### 6.2 潜在问题

| 问题 | 影响 | 严重程度 |
|------|------|---------|
| 运行时内存中 Watch 对象含 processor_config_* | 如果有代码直接从 Watch 读取这些字段，可能拿到"幽灵数据"（用户没提交但内存里有上次的值） | 中 |
| L194 的注释与实际行为不符 | 误导后续维护者，以为上游已拦截，可能在修改 `_get_commit_data()` 时引入回归 | 中 |
| 多次 form.data 访问造成性能浪费 | 每次都遍历所有字段并构造新字典，10 次访问等于遍历了 10 遍 | 低（字段数只有 40+，开销可忽略） |

### 6.3 修复建议（无需实际修改，仅作分析）

最简单且不改变其他行为的修复——把 L194 提取出来的"已干净"的 form.data 保存到一个变量，后续统一使用：

```python
# L194 修改为
form_data = processors.extract_processor_config_from_form_data(form.data)
processors.save_processor_config(datastore, uuid, processor_config_data)

# 后续所有 form.data 访问改为使用 form_data（一个已剥离的字典）
# L202: form_data['proxy']
# L207-L209: form_data.get('filter_text_added')
# L216-L221: form_data.get('tags')
# L225: watch.update(form_data)
# L234: form_data.get('processor')
```

这样既消除了内存不一致，又减少了 9 次不必要的 form.data 构造开销，同时让 L194 的注释名副其实。

但需要注意：`form_data` 里不再有 `processor_config_*` 字段，而 L234 `form.data.get('processor')` 读取的是普通字段 `processor`（不是 `processor_config_*`），不受影响。
