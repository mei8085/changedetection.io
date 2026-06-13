# changedetection.io 监控项编辑提交链路——4 处核心机制深度解析

## 目录
1. [processor_config_* 不进 watch.json 的真实拦截层](#1-processor_config_-不进-watchjson-的真实拦截层)
2. [form.data 与 extra_update_obj 二次 update 的覆盖顺序与同名键冲突](#2-formdata-与-extra_update_obj-二次-update-的覆盖顺序与同名键冲突)
3. [GET 从 JSON 回填 FormField 的精确匹配与启发式回退](#3-get-从-json-回填-formfield-的精确匹配与启发式回退)
4. [FormField 子表单与扁平 input 的状态保持差异](#4-formfield-子表单与扁平-input-的状态保持差异)

---

## 1. processor_config_* 不进 watch.json 的真实拦截层

### 1.1 三层防御体系

`processor_config_*` 字段不进入 `watch.json` 并非靠单一拦截点，而是形成了**三层防御**的纵深体系：

| 层级 | 拦截位置 | 作用 |
|------|---------|------|
| 第一层（提交入口） | UI 路由 + API 路由 | 从 form data / JSON data 中剥离 |
| 第二层（内存模型） | `Watch._get_commit_data()` | commit 序列化时再次过滤 |
| 第三层（历史清理） | `Watch.clear_watch()` | 清历史时保留处理器配置文件 |

### 1.2 第一层：提交入口处的剥离

**UI 路径**——[edit.py#L193-L195](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L193-L195)：

```python
processor_config_data = processors.extract_processor_config_from_form_data(form.data)
processors.save_processor_config(datastore, uuid, processor_config_data)
```

**API 路径**——[Watch.py#L198](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/api/Watch.py#L198)：

```python
processor_config_data = processors.extract_processor_config_from_form_data(json_data)
```

两者都调用同一个工具函数 [extract_processor_config_from_form_data](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/__init__.py#L472-L498)：

```python
def extract_processor_config_from_form_data(form_data):
    processor_config_data = {}
    for field_name in list(form_data.keys()):
        if field_name.startswith('processor_config_'):
            config_key = field_name.replace('processor_config_', '')
            processor_config_data[config_key] = form_data[field_name]
            del form_data[field_name]  # 关键：in-place 删除
    return processor_config_data
```

**关键特性**：
- 函数直接 **in-place 修改** 传入的 `form_data` 字典，删除所有 `processor_config_` 前缀的 key
- 返回剥离出来的配置字典（去掉前缀）
- 这是最主要的拦截层，绝大多数情况下在这一步就被剥离了

### 1.3 第二层：Watch 模型 commit 时的防御性过滤

即使第一层因为某种原因漏过去了，[Watch._get_commit_data()](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py#L1064-L1093) 在序列化写入磁盘前会再次过滤：

```python
def _get_commit_data(self):
    snapshot = dict(self)
    watch_dict = {
        k: copy.deepcopy(v) for k, v in snapshot.items()
        if not k.startswith('processor_config_') and not k.startswith('__')
    }
    return watch_dict
```

这是一道**防御性编程**的保险。即使上游代码出 bug 把 `processor_config_*` 写入了 Watch 字典，commit 到磁盘时也会被拦下来。

同时 `__` 前缀的瞬态内存字段也在这一层被过滤，确保磁盘上的 JSON 只存持久化数据。

### 1.4 第三层：清历史时保留配置文件

[Watch.clear_watch()](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py#L312-L350) 清理 watch 数据目录时，特意保留 processor config 文件：

```python
processor_config_files = {f"{name}.json" for name in processor_names}
for item in pathlib.Path(str(self.data_dir)).glob("*.*"):
    if item.name in processor_config_files:
        continue  # 跳过，不删除
    os.unlink(item)
```

这说明处理器配置被视为**配置数据**而非**历史数据**，清历史时保留。

### 1.5 存储路径对照

| 数据类型 | 存储位置 | 生命周期 |
|---------|---------|---------|
| 主配置 | `{uuid}/watch.json` | 随 watch 存在 |
| 处理器配置 | `{uuid}/{processor_name}.json` | 随 watch 存在，独立文件 |
| 历史快照 | `{uuid}/history.txt` + `*.txt.br` 等 | 可被清理 |
| 截图 | `{uuid}/last-screenshot.png` 等 | 可被清理 |

---

## 2. form.data 与 extra_update_obj 二次 update 的覆盖顺序与同名键冲突

### 2.1 执行顺序与覆盖规则

在 [edit.py#L225-L226](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L225-L226)：

```python
datastore.data['watching'][uuid].update(form.data)       # 第一次
datastore.data['watching'][uuid].update(extra_update_obj)  # 第二次（覆盖）
```

由于 Python dict 的 `update()` 是**后写覆盖前写**，所以 `extra_update_obj` 中的字段会覆盖 `form.data` 中的同名字段。

### 2.2 同名键冲突字段全览

通过对比 `form.data` 的字段（来自 `processor_text_json_diff_form` 定义）和 `extra_update_obj` 的字段，找出以下**冲突字段**：

| 冲突字段 | form.data 中值的来源 | extra_update_obj 中值的来源 | 为什么走 extra |
|---------|---------------------|---------------------------|--------------|
| `time_between_check` | `FormField` 的 `.data`（嵌套 dict） | `form.time_between_check.data` | 见 2.3 节详解 |
| `proxy` | 表单原始值（`''` 或具体 proxy 名） | `None`（当 `form.data['proxy'] == ''` 时） | 空字符串需转为 None |
| `filter_text_added` | 表单 Boolean 值 | `True`（当三项全为 False 时） | 全不选 → 回退为默认全选 |
| `filter_text_replaced` | 表单 Boolean 值 | `True`（当三项全为 False 时） | 同上 |
| `filter_text_removed` | 表单 Boolean 值 | `True`（当三项全为 False 时） | 同上 |
| `tags` | `StringTagUUID` 处理后的字符串或 list | 转换后的 UUID 列表 | 字符串 → UUID 列表转换需访问 datastore |
| `paused` | 表单值（如果有的话） | `False`（当 `unpause_on_save` 参数时） | URL 参数驱动的条件重置 |
| `consecutive_filter_failures` | 不在 form 中 | `0` | 重置计数器状态字段 |
| `last_error` | 不在 form 中 | `False` | 重置错误状态字段 |

### 2.3 为什么 `time_between_check` 要走 extra？

这是最容易困惑的一点。`form.data['time_between_check']` 和 `form.time_between_check.data` 按理说应该是同一个东西——都是 FormField 的数据字典。

但仔细看代码顺序会发现端倪：

```python
# 第 190 行：先把 time_between_check 存进 extra
extra_update_obj['time_between_check'] = form.time_between_check.data

# 第 194 行：然后才调用 extract_processor_config_from_form_data
# 这个函数会 in-place 修改 form.data
processor_config_data = processors.extract_processor_config_from_form_data(form.data)
```

实际上 `time_between_check` 不是 `processor_config_*` 字段，不会被 extract 函数修改。那为什么要提前存到 extra 里？

**真正原因**：这是为了保证 `time_between_check` 作为一个 **FormField 嵌套字典**，能够完整地覆盖写入，而不受 `form.data` 中其他字段处理的影响。更重要的是——

因为 `EnhancedFormField`（time_between_check 的类型）的 `.data` 返回的是一个嵌套字典 `{'weeks': ..., 'days': ..., 'hours': ..., 'minutes': ..., 'seconds': ...}`。

虽然 `form.data['time_between_check']` 也会返回同样的结构，但作者选择**显式**地通过 `extra_update_obj` 赋值，相当于给这个字段加了一个"保险"，确保它一定以正确的嵌套字典格式写入，不会因为 WTForms 的某些边界情况导致数据结构异常。

此外还有一个更实际的原因：**`time_between_check_use_default` 开关的联动逻辑**。当用户勾选"使用全局设置"时，表单可能不提交 time_between_check 的子字段，但 extra 中可以显式保证写入正确的结构。

### 2.4 为什么 `tags` 必须走 extra

注释写得很明白——[edit.py#L214](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L214)：

> "Because wtforms doesn't support accessing other data in process_ , but we convert the CSV list of tags back to a list of UUIDs"

- 表单中 `tags` 字段显示的是**逗号分隔的标签名称**（友好的用户界面）
- 存储时需要的是 **UUID 列表**（数据模型的内部表示）
- 这个转换需要访问 `datastore`（调用 `datastore.add_tag()`）
- WTForms 的 `process_formdata()` 方法无法方便地访问外部依赖（如 datastore）
- 所以只能在提交 handler 中手动转换后，通过 `extra_update_obj` 写入

### 2.5 为什么 `proxy` 要走 extra

```python
if datastore.proxy_list is not None and form.data['proxy'] == '':
    extra_update_obj['proxy'] = None
```

原因很简单：表单中空选项的值是 `''`（空字符串），但数据模型中"使用默认代理"用 `None` 表示。需要做一次**空字符串 → None** 的类型转换。

### 2.6 为什么 `filter_text_*` 三兄弟要走 extra

```python
if 'filter_text_added' in form.data and not form.data.get('filter_text_added') \
        and 'filter_text_replaced' in form.data and not form.data.get('filter_text_replaced') \
        and 'filter_text_removed' in form.data and not form.data.get('filter_text_removed'):
    extra_update_obj['filter_text_added'] = True
    extra_update_obj['filter_text_replaced'] = True
    extra_update_obj['filter_text_removed'] = True
```

这是一个**保护性回退**：当用户把三个复选框全部取消勾选时，系统自动把它们全部设回 `True`（默认值）。因为三项全为 False 意味着"不检测任何变化"，这在绝大多数场景下是无意义的配置。

### 2.7 二次 update 模式的设计思想

可以总结为一个清晰的分层：

```
form.data  →  通用字段批量写入（大部分字段）
    ↓
extra_update_obj  →  特殊字段覆盖（需转换、需联动、需重置）
    ↓
最终 Watch 对象
```

这种模式的好处是：
- **主路径简洁**：`form.data` 一把梭，代码清爽
- **特殊情况集中处理**：所有需要特殊逻辑的字段都在 `extra_update_obj` 中处理，边界条件一目了然
- **可扩展性好**：新增特殊字段只需加一行到 extra，不用改主路径

---

## 3. GET 从 JSON 回填 FormField 的精确匹配与启发式回退

### 3.1 回填入口

GET 请求加载编辑页时，需要从独立的 `{processor_name}.json` 文件中读取处理器配置，回填到表单对应的 FormField 中。

代码位置：[edit.py#L131-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L131-L162)

```python
if request.method == 'GET' and processor_name:
    processor_instance = difference_detection_processor(datastore, uuid)
    config_filename = f'{processor_name}.json'
    processor_config = processor_instance.get_extra_watch_config(config_filename)
    # ... 回填逻辑
```

### 3.2 JSON 文件读取

[difference_detection_processor.get_extra_watch_config()](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/base.py#L274-L303)：

```python
def get_extra_watch_config(self, filename):
    filepath = os.path.join(data_dir, filename)
    if not os.path.isfile(filepath):
        return {}
    with open(filepath, 'r', encoding='utf-8') as f:
        return json.load(f)
```

返回的是从 JSON 文件直接解析出的字典，例如 `restock_diff.json` 可能包含：

```json
{
  "restock_diff": {
    "in_stock_processing": "in_stock_only",
    "follow_price_changes": true,
    "price_change_min": null,
    "price_change_max": null,
    "price_change_threshold_percent": 5.0
  }
}
```

注意外层 key 是 `restock_diff`，与 processor 名称一致。

### 3.3 精确匹配（第一优先级）

```python
target_field = getattr(form, f'processor_config_{config_key}', None)
```

直接通过属性名查找。例如 JSON key 是 `restock_diff`，就去找 `form.processor_config_restock_diff` 这个 FormField。

这是**约定优于配置**的设计：
- JSON 中的配置 key → `processor_config_{key}` → 表单字段名
- 只要命名规范对上，就能精确匹配

例如 [restock_diff/forms.py#L34](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/restock_diff/forms.py#L34) 就严格遵守了这个约定：

```python
processor_config_restock_diff = FormField(RestockSettingsForm)
```

### 3.4 启发式回退（第二优先级）

当精确匹配找不到时，启动启发式搜索：

```python
if target_field is None:
    for form_field in form:
        if isinstance(form_field, FormField) and all(k in form_field.form._fields for k in config_value):
            target_field = form_field
            break
```

**匹配规则**：遍历表单中所有 FormField 类型的字段，如果某个 FormField 的**子字段集合完全覆盖 JSON 值的所有 key**，就认为匹配成功。

这是一种**鸭子类型**的匹配方式——不看名字，看"能不能装下这些数据"。

例如 JSON 值有 `in_stock_processing`、`follow_price_changes`、`price_change_min` 等 5 个 key，就找一个**至少**有这 5 个字段的 FormField。

### 3.5 回填方式：逐字段赋值

找到目标 FormField 后，并不是整体赋值（如 `target_field.data = config_value`），而是**逐子字段赋值**：

```python
for sub_key, sub_value in config_value.items():
    sub_field = target_field.form._fields.get(sub_key)
    if sub_field is not None:
        sub_field.data = sub_value
```

为什么这么做？
1. **容错性**：JSON 中可能有多余的字段，或者 FormField 新增了字段，不匹配的就跳过
2. **类型安全**：直接设置 `.data` 绕开了 WTForms 的 process 流程，不会触发验证或类型转换
3. **细粒度控制**：可以只回填存在的字段，不影响其他字段的默认值

### 3.6 两种 processor_config 字段形态

有趣的是，并不是所有 `processor_config_*` 字段都是 FormField 类型：

| 处理器 | 字段类型 | 例子 |
|-------|---------|------|
| restock_diff | FormField（嵌套子表单） | `processor_config_restock_diff = FormField(RestockSettingsForm)` |
| image_ssim_diff | 扁平字段（多个独立字段） | `processor_config_min_change_percentage = IntegerField(...)` |

对于 image_ssim_diff 这种扁平字段，JSON 存储的是：
```json
{
  "min_change_percentage": 10,
  "pixel_difference_threshold_sensitivity": "medium"
}
```

回填时精确匹配会失败（因为没有 `processor_config_image_ssim_diff` 这个 FormField），但逐字段的精确匹配能工作吗？实际上要看代码的处理方式——当前代码只处理 `isinstance(config_value, dict)` 的情况，且只往 FormField 里填。

这意味着**扁平字段的 processor config 在 GET 时可能不会被自动回填**，它们可能依赖 `form.data` 的 data 参数初始化（如果 Watch 对象里有这些字段的话）。这是一个潜在的不一致点。

---

## 4. FormField 子表单与扁平 input 的状态保持差异

### 4.1 共同点：DOM 存在即状态保留

无论是扁平字段还是 FormField 子表单，最终都会渲染为真实的 HTML `<input>` 元素。由于页签切换只是 CSS `display: none/block`，**DOM 节点不销毁**，所以：

- 用户输入的值由浏览器原生维护
- 切换页签不会丢失任何已填内容
- 这一点上两者没有区别

### 4.2 差异一：HTML name 属性的命名空间

扁平字段的 name 就是字段名本身：
```html
<input type="text" name="url" id="url" value="...">
<input type="text" name="title" id="title" value="...">
```

FormField 子表单的字段 name 带前缀，默认用 `-` 分隔：
```html
<!-- time_between_check 是 FormField，包含 weeks/days/hours 等子字段 -->
<select name="time_between_check-weeks" id="time_between_check-weeks">...</select>
<select name="time_between_check-days" id="time_between_check-days">...</select>
<select name="time_between_check-hours" id="time_between_check-hours">...</select>
```

WTForms 的 FormField 通过 `separator` 参数（默认 `-`）拼接前缀，解析时再拆分。

**这意味着**：
- 扁平字段：一个 name 对应一个值
- FormField：多个 name 带相同前缀，共同组成一个嵌套字典值

### 4.3 差异二：form.data 中的数据结构

扁平字段在 `form.data` 中直接是值：
```python
form.data == {
    'url': 'https://example.com',
    'title': 'My Watch',
    'proxy': '',
    # ...
}
```

FormField 在 `form.data` 中是**嵌套字典**：
```python
form.data == {
    'time_between_check': {
        'weeks': '0',
        'days': '0',
        'hours': '24',
        'minutes': '0',
        'seconds': '0'
    },
    'time_schedule_limit': {
        'enabled': False,
        'monday': { 'enabled': True, 'start_time': '00:00', 'duration': {...} },
        # ... 每天都是嵌套的
    },
    # ...
}
```

嵌套深度可以很深。比如 `time_schedule_limit` 是 FormField，里面的 `monday` 也是 FormField，`monday` 里的 `duration` 还是 FormField——三层嵌套。

### 4.4 差异三：初始化回填的路径不同

扁平字段的初始化很简单——`Form` 的构造函数会把 `data` 参数中对应 key 的值赋给字段。

FormField 的初始化则需要**递归**：
1. 外层 Form 找到 `time_between_check` 这个 FormField
2. FormField 从 data 中取出 `time_between_check` 对应的 dict
3. FormField 构造内部的 `TimeBetweenCheckForm`，把这个 dict 作为 data 传进去
4. 内层 Form 再把各个子字段的值填上

正因为这个递归机制，FormField 可以完美对应嵌套的 JSON 数据结构（比如 Watch 的 `time_between_check` 本身就是一个 dict）。

### 4.5 差异四：GET 回填时的处理方式

如第 3 章所述，GET 请求时从 JSON 回填 processor config 数据，对于 FormField 类型是**逐子字段赋值**：

```python
for sub_key, sub_value in config_value.items():
    sub_field = target_field.form._fields.get(sub_key)
    if sub_field is not None:
        sub_field.data = sub_value
```

而扁平字段不需要这个特殊处理——它们直接通过表单构造时的 `data=default` 参数就初始化好了。

### 4.6 差异五：验证逻辑的组织方式

扁平字段的验证器挂在字段上，验证失败产生字段级错误。

FormField 有**两级验证**：
1. 子字段各自的验证器（如 `NumberRange`）
2. FormField 整体的验证器（如 `RequiredTimeInterval`）——检查"至少一个时间间隔 > 0"这种跨字段规则

而且项目中还自定义了 `EnhancedFormField`，支持**条件验证**（依赖另一个字段的值决定是否验证）：

```python
time_between_check = EnhancedFormField(
    TimeBetweenCheckForm,
    conditional_field='time_between_check_use_default',  # 依赖开关
    conditional_test_function=validate_time_between_check_has_values,
)
```

当 `time_between_check_use_default` 为 True 时，即使时间间隔全为 0 也不报错。

### 4.7 差异六：与 Watch 模型的交互方式

扁平字段 update 到 Watch（dict 子类）就是简单的 key-value 写入。

FormField 字段 update 到 Watch 时，写入的是**整个嵌套 dict**。这要求 Watch 模型中对应的 key 也是 dict 结构——恰好 Watch 的 `time_between_check`、`time_schedule_limit` 等字段本身就是 dict。

换句话说：**FormField 的嵌套结构与 Watch 模型的嵌套 dict 是一一对应的**，这不是巧合，是有意设计的。WTForms 的 FormField 在这里扮演了"嵌套数据结构的表单适配器"角色。

### 4.8 状态保持差异总结

| 维度 | 扁平字段 | FormField 子表单 |
|------|---------|----------------|
| HTML name | `fieldname` | `fieldname-subfield`（带前缀） |
| form.data 结构 | 直接值 | 嵌套字典 |
| 初始化方式 | 直接赋值 | 递归构造子 Form |
| GET 回填 | 走 form 构造 data 参数 | 需额外逐字段赋值（processor config） |
| 验证层级 | 仅字段级 | 字段级 + FormField 整体级 |
| 写入 Watch | 单 key 写入 | 整个嵌套 dict 写入 |
| DOM 状态保持 | 浏览器原生维护 | 浏览器原生维护（相同） |
| 页签切换不丢值 | ✅ 是 | ✅ 是 |

**核心结论**：在"页签切换不丢值"这个问题上，两者没有本质区别——都是靠 DOM 不销毁来保持状态。真正的差异体现在数据结构、验证、初始化回填等"数据层"的问题上。

---

## 附录：关键代码索引

| 机制 | 核心文件 | 关键行号 |
|------|---------|---------|
| processor_config 提取 | [processors/__init__.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/__init__.py) | L472-L498 |
| processor_config 保存 | [processors/__init__.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/__init__.py) | L428-L469 |
| commit 层二次过滤 | [model/Watch.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py) | L1064-L1093 |
| 二次 update 逻辑 | [blueprint/ui/edit.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py) | L180-L226 |
| GET 回填 FormField | [blueprint/ui/edit.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py) | L131-L162 |
| FormField 自定义类 | [forms.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/forms.py) | L313-L363 |
| CSS 页签显隐 | [styles.scss](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/static/styles/scss/styles.scss) | L814-L822 |
| JS 页签切换 | [static/js/tabs.js](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/static/js/tabs.js) | L1-L45 |
