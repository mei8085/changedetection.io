# changedetection.io 监控项编辑提交链路——4 处细节深度解析（Followup）

本文是 [form-mechanism.md](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/form-mechanism.md) 的补充，针对 4 处具体细节展开源码级分析。

---

## 1. processor_config_* 在 UI POST 与 API PUT 两条链路上拦截层的真实差异

虽然两条链路都调用了同一个 `extract_processor_config_from_form_data()` 函数，但在它**前后**的处理逻辑有显著不同，形成了不同的拦截层级。

### 1.1 UI POST 链路（edit.py）

完整流程见 [edit.py#L180-L238](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L180-L238)：

```
步骤 1: form.validate()
   ↓  WTForms 对所有字段（含 processor_config_*）做验证
步骤 2: extra_update_obj['time_between_check'] = form.time_between_check.data
   ↓  先把 time_between_check 存进 extra（在 extract 之前就取出来）
步骤 3: processor_config_data = extract_processor_config_from_form_data(form.data)
   ↓  ← 核心拦截点：in-place 删除 form.data 中的 processor_config_*
步骤 4: processors.save_processor_config(..., processor_config_data)
   ↓  写入 {processor_name}.json
步骤 5: form.data → watch.update(form.data)  ← 已干净，不含 processor_config_*
步骤 6: watch.update(extra_update_obj)
步骤 7: watch_class = get_custom_watch_obj_for_processor(form.data.get('processor'))
   ↓  可能切换 Watch 子类
步骤 8: watch.commit()
   ↓  ← 防御性二次过滤：_get_commit_data() 再次剔除 processor_config_*
```

**UI 链路特点**：
- 拦截时机在 **form.validate() 之后**，`processor_config_*` 字段参与了完整的 WTForms 验证（如 `NumberRange(min=1, max=100)`）。
- `extra_update_obj` 中的 `time_between_check` 在拦截**之前**就从 `form.time_between_check.data` 取出，不受 `extract_processor_config_from_form_data` 影响（当然它本身也不是 `processor_config_*` 字段）。
- 有 `Watch.commit()` 的防御性二次过滤兜底。

### 1.2 API PUT 链路（api/Watch.py）

完整流程见 [api/Watch.py#L145-L236](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/api/Watch.py#L145-L236)：

```
步骤 1: 手动验证 proxy、time_between_check、notification_urls、url 等
   ↓  注意：这些是手动验证，不是 WTForms！
步骤 2: json_data = strip_internal_api_fields(dict(request.json))
   ↓  先剔除所有 __ 前缀的瞬态字段
步骤 3: processor_config_data = extract_processor_config_from_form_data(json_data)
   ↓  ← 核心拦截点：in-place 删除 json_data 中的 processor_config_*
步骤 4: readonly_fields = get_readonly_watch_fields()
        property_fields = WatchModel.get_property_names()
        fields_to_ignore = readonly_fields | property_fields
   ↓  剔除只读字段和 @property 计算字段
步骤 5: unknown_fields = set(json_data.keys()) - valid_fields
        if unknown_fields: return 400
   ↓  ← 白名单校验：只接受 schema 中定义的字段
步骤 6: watch.update(json_data)
   ↓  已干净：processor_config_* + 只读 + 计算 + 未知字段 都被剔除
步骤 7: watch.commit()
   ↓  ← 防御性二次过滤
步骤 8: processors.save_processor_config(..., processor_config_data)
   ↓  写入 {processor_name}.json
```

**API 链路特点**：
- 拦截时机在**最前面**（仅次于手动验证和 `strip_internal_api_fields`），`processor_config_*` 不参与后续的白名单校验。
- **没有 WTForms 验证**，完全靠 OpenAPI schema 的 `validate_openapi_request('updateWatch')` 装饰器做请求校验。也就是说 `processor_config_*` 的子字段格式由 OpenAPI schema 约束，不由 WTForms 验证。
- `save_processor_config` 在 `watch.commit()` **之后**调用，顺序与 UI 相反。但因为写入的是独立 JSON 文件，这个顺序不影响正确性。
- 多了一道**白名单校验**（步骤 5），而 UI 链路没有——UI 链路依靠 WTForms 只处理已定义的字段来隐式实现白名单。

### 1.3 拦截层差异对比表

| 维度 | UI POST | API PUT |
|------|---------|---------|
| 拦截前置条件 | `form.validate()` 已通过（含 processor_config_* 字段级验证） | 仅手动验证了 proxy/url/notification 等少数字段 |
| 拦截函数 | `extract_processor_config_from_form_data(form.data)` | `extract_processor_config_from_form_data(json_data)` |
| 拦截后额外过滤 | 无（信任 WTForms） | `strip_internal_api_fields` + 只读字段剔除 + `@property` 剔除 + 白名单校验 |
| processor_config 验证方式 | WTForms 字段验证器（NumberRange、Optional 等） | OpenAPI schema（`validate_openapi_request` 装饰器） |
| 保存顺序 | 先保存 JSON → 后 watch.commit() | 先 watch.commit() → 后保存 JSON |
| 防御性二次过滤 | `watch.commit()` → `_get_commit_data()` | `watch.commit()` → `_get_commit_data()` |

### 1.4 为什么 API PUT 要额外多一层白名单校验？

UI 链路中 WTForms 的 `form.data` 只会返回表单上已定义的字段，客户端即使 POST 了额外的字段也不会被 WTForms 处理——天然就是白名单。但 API 链路用的是原始的 `request.json`（Python dict），如果不做白名单校验，客户端可以写入任意字段到 Watch 对象，存在安全隐患。

---

## 2. time_between_check 走 extra 的真实理由

在 [edit.py#L190](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L190)：

```python
extra_update_obj['time_between_check'] = form.time_between_check.data
```

上一份报告分析了几种可能的原因，但深入代码后发现**真正的原因只有一个**——而且它和 `time_between_check_use_default` 开关紧密相关。

### 2.1 关键：time_between_check_use_default 的联动逻辑

表单定义在 [forms.py#L832-L842](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/forms.py#L832-L842)：

```python
time_between_check = EnhancedFormField(
    TimeBetweenCheckForm,
    label=_l('Time Between Check'),
    conditional_field='time_between_check_use_default',
    conditional_message=REQUIRE_ATLEAST_ONE_TIME_PART_WHEN_NOT_GLOBAL_DEFAULT,
    conditional_test_function=validate_time_between_check_has_values
)

time_between_check_use_default = BooleanField(
    _l('Use global settings for time between check and scheduler.'),
    default=False
)
```

`EnhancedFormField.validate()` 的逻辑（[forms.py#L337-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/forms.py#L337-L363)）：
- 当 `time_between_check_use_default == True` 时，**跳过** `time_between_check` 的"至少一个时间间隔 > 0"的验证
- 当 `time_between_check_use_default == False` 时，必须满足至少一个时间间隔 > 0

### 2.2 问题场景

用户勾选了"使用全局设置"（`time_between_check_use_default=True`），然后在子表单里把 weeks/days/hours/minutes/seconds 都填成了 0。

由于条件验证生效，`form.validate()` 通过。此时：

```python
form.time_between_check.data == {
    'weeks': '0', 'days': '0', 'hours': '0',
    'minutes': '0', 'seconds': '0'
}
```

**这些全 0 的值会出现在 `form.data['time_between_check']` 里。**

### 2.3 真正的理由——和 time_between_check_use_default 一起保存才能表示"使用全局设置"

看 API 的验证函数 [api/Watch.py#L21-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/api/Watch.py#L21-L47)：

```python
def validate_time_between_check_required(json_data):
    use_default = json_data.get('time_between_check_use_default', True)
    if use_default:
        return None  # 使用全局设置，不需要验证
    # ... 否则必须至少一个时间间隔 > 0
```

系统用 `time_between_check_use_default == True` + `time_between_check` 全 0 来表示"使用全局设置"。而 `time_between_check_use_default` 字段也在 `form.data` 中，它是 BooleanField，值为 `True`/`False`。

**那为什么 `time_between_check` 还要走 extra？难道 `form.data['time_between_check']` 和 `form.time_between_check.data` 不一样吗？**

答案是：**在正常情况下它们确实完全相同**。`form.data['time_between_check']` 就是 `form.time_between_check.data`。

走 extra 的真实理由是——**代码防御性 + 明确语义**：

1. **提取时机的保险**：`extra_update_obj['time_between_check']` 在 L190 就提取了，而 `form.data` 在 L194 被 `extract_processor_config_from_form_data` in-place 修改。虽然 `time_between_check` 不是 `processor_config_*` 不会被删，但作者的代码习惯是"重要的嵌套字段提前存好"。

2. **避免 WTForms 边界问题**：FormField 的 `.data` 涉及子表单递归构造，某些场景下（如 `process_formdata` 异常、子字段类型转换失败）`form.data` 中 FormField 的值可能和直接访问 `form.fieldname.data` 有细微差异（例如一个返回原始 dict，一个返回经过 process 的值）。显式通过 extra 写入可以保证写入的是 `.data` 属性的"官方"输出。

3. **与 ignore_text 的处理风格一致**：`ignore_text`（L198-L199）也是单独赋值不走 `form.data` update。这似乎是作者处理"需要特殊关注字段"的统一模式。

4. **为了明确覆盖顺序**：即使 `form.data['time_between_check']` 有值，extra 中的值会在第二次 update 时覆盖它，确保最终写入的值正确。这和 `proxy`、`filter_text_*` 的处理思路一致——**只要一个字段需要特殊关注，就放进 extra 里显式控制写入**。

### 2.4 结论

`time_between_check` 走 extra **不是**因为 `form.data['time_between_check']` 的值有问题，而是因为代码风格上的防御性编程：把需要"特别保证"的字段显式放入 `extra_update_obj`，在第二次 update 时覆盖写入。这是一种"不依赖隐式路径"的工程实践。

---

## 3. 扁平 processor_config 在 GET 是否被回填

### 3.1 两种 processor_config 形态

先回顾：

- **FormField 型**（如 restock_diff）：
  ```python
  processor_config_restock_diff = FormField(RestockSettingsForm)
  ```
  JSON 存储：
  ```json
  {"restock_diff": {"in_stock_processing": "...", "follow_price_changes": true}}
  ```

- **扁平型**（如 image_ssim_diff）——[image_ssim_diff/forms.py#L46-L62](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/image_ssim_diff/forms.py#L46-L62)：
  ```python
  processor_config_min_change_percentage = IntegerField(...)
  processor_config_pixel_difference_threshold_sensitivity = SelectField(...)
  ```
  JSON 存储（扁平结构）：
  ```json
  {"min_change_percentage": 10, "pixel_difference_threshold_sensitivity": "medium"}
  ```

### 3.2 GET 回填代码分析

关键代码在 [edit.py#L131-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L131-L162)：

```python
processor_config = processor_instance.get_extra_watch_config(config_filename)
# processor_config 可能是:
#   FormField 型: {"restock_diff": {"in_stock_processing": ..., ...}}
#   扁平型:     {"min_change_percentage": 10, "pixel_difference_threshold_sensitivity": "..."}

for config_key, config_value in processor_config.items():
    if not isinstance(config_value, dict):
        continue        # ← 关键判断！非 dict 直接跳过

    # 精确匹配: processor_config_{config_key}
    target_field = getattr(form, f'processor_config_{config_key}', None)

    # 启发式回退: 找 FormField 子字段覆盖所有 key
    if target_field is None:
        for form_field in form:
            if isinstance(form_field, FormField) and all(k in form_field.form._fields for k in config_value):
                target_field = form_field
                break

    # 逐子字段赋值
    if target_field is not None:
        for sub_key, sub_value in config_value.items():
            sub_field = target_field.form._fields.get(sub_key)
            if sub_field is not None:
                sub_field.data = sub_value
```

### 3.3 扁平型字段为什么不会被回填？

因为 `if not isinstance(config_value, dict): continue` 这行代码。

对于扁平型 JSON：
```json
{"min_change_percentage": 10, "pixel_difference_threshold_sensitivity": "medium"}
```

遍历时：
- `config_key = "min_change_percentage"`, `config_value = 10` → `10` 不是 dict → **跳过**
- `config_key = "pixel_difference_threshold_sensitivity"`, `config_value = "medium"` → 字符串不是 dict → **跳过**

**所有扁平字段全部被 `continue` 跳过，不会被回填到表单。**

### 3.4 那 GET 时扁平 processor_config 的值从哪里来？

GET 请求构造表单时用的是（[edit.py#L118-L122](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L118-L122)）：

```python
form = form_class(formdata=None,    # GET 请求 formdata=None
                  data=default,     # default 是 Watch 对象的 deepcopy
                  ...)
```

WTForms 构造表单时会从 `data=default` 参数中读取对应字段的值。问题是：**扁平型 `processor_config_*` 字段的值存在独立的 `image_ssim_diff.json` 文件里，不在 Watch 对象（watch.json）中。**

所以结论是：

> **GET 加载编辑页时，扁平型 `processor_config_*` 字段的值不会被回填，显示的是表单默认值。**

这是一个**真实的 Bug**。用户如果用 image_ssim_diff 处理器，保存 `min_change_percentage=10` 后重新打开编辑页，字段会显示为占位符（默认值），而不是已保存的 10。

对比 FormField 型的 restock_diff：GET 时 JSON 中的 `{"restock_diff": {...}}` 因为 `restock_diff` key 的值是 dict，能通过 `isinstance(config_value, dict)` 判断，进入精确匹配 → `form.processor_config_restock_diff` → 逐子字段赋值，正常回填。

### 3.5 扁平字段在 POST/PUT 时的行为

提交时 `extract_processor_config_from_form_data` 能正确提取扁平字段（它不关心值的类型，只看 key 前缀），保存也正常。问题只在 GET 回填环节。

---

## 4. 切换 processor 后旧同名 JSON 的残留与清理路径

### 4.1 切换 processor 的完整流程

假设用户在编辑页把 processor 从 `restock_diff` 切换到 `text_json_diff` 并保存。

**UI 链路**（[edit.py#L233-L238](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L233-L238)）：

```python
# form.data['processor'] 现在是 'text_json_diff'
watch_class = processors.get_custom_watch_obj_for_processor(form.data.get('processor'))

# 重新构造 Watch 对象（可能是不同的子类）
datastore.data['watching'][uuid] = watch_class(
    datastore_path=datastore.datastore_path,
    __datastore=datastore.data,
    default=datastore.data['watching'][uuid]
)

# 写入 watch.json（不含 processor_config_*）
datastore.data['watching'][uuid].commit()
```

**processor 保存**（[processors/__init__.py#L428-L469](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/__init__.py#L428-L469)）：

```python
def save_processor_config(datastore, watch_uuid, config_data):
    processor_name = watch.get('processor', 'text_json_diff')  # ← 用的是新 processor 名
    config_filename = f'{processor_name}.json'                 # ← 写 text_json_diff.json
    processor_instance.update_extra_watch_config(config_filename, config_data)
```

保存时用的是 Watch 对象中**新的** `processor` 字段值，所以写入的是 `text_json_diff.json`，而不是旧的 `restock_diff.json`。

### 4.2 旧 JSON 文件的残留状态

切换后，watch 的数据目录里可能同时存在：

```
{uuid}/
├── watch.json           ← 更新后，processor 字段已变为 'text_json_diff'
├── restock_diff.json    ← 旧 processor 的配置文件，被遗留在磁盘上
├── text_json_diff.json  ← 新 processor 的配置文件（可能为空 {}）
└── ... 其他历史文件
```

**旧文件 `restock_diff.json` 不会被自动删除。**

### 4.3 有没有清理路径？

代码中**不存在**任何在切换 processor 时主动删除旧 `{processor}.json` 文件的逻辑。让我们逐个排查可能的清理路径：

| 可能的清理位置 | 是否清理？ | 说明 |
|--------------|-----------|------|
| UI POST 切换 processor | ❌ 否 | 只写入新 processor 的 JSON，不删旧的 |
| API PUT 切换 processor | ❌ 否 | 同上，只写新的 |
| `Watch.clear_watch()` | ✅ 部分 | 但它只在"清历史"时调用，且 processor config 文件会被 `continue` 跳过（见 [Watch.py#L341-L345](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py#L341-L345)） |
| `Watch.delete()` | ✅ 是 | 删除整个数据目录，自然包含所有 processor JSON |
| 数据迁移 `update_30` | ❌ 否 | 只负责把 watch.json 里的 `restock_settings` 迁到 `restock_diff.json`，不清理 |
| 定时任务/GC | ❌ 否 | 没有任何后台清理逻辑 |

### 4.4 残留文件会产生什么影响？

**运行时无影响**。GET 加载编辑页时：
```python
processor_name = datastore.data['watching'][uuid].get('processor', '')
config_filename = f'{processor_name}.json'  # 只读取当前 processor 对应的 JSON
```

只读取当前 processor 的配置文件，旧文件被完全忽略。

**潜在问题**：
1. **磁盘占用**：切换过的 processor 越多，遗留的 JSON 文件越多（虽然单个文件很小）。
2. **隐私/安全**：旧配置中可能包含敏感信息（如 API Key），在"清历史"时不会被删除。
3. **切回旧 processor 时**：用户可能惊讶地发现很久之前的配置"自动回来了"——因为旧 JSON 文件一直留在磁盘上。

### 4.5 切换后再切回来

如果用户从 `restock_diff` → `text_json_diff` → 再切回 `restock_diff`：

- GET 编辑页时，`config_filename = 'restock_diff.json'`
- `get_extra_watch_config('restock_diff.json')` 会读取到磁盘上残留的旧配置
- 旧配置被回填到表单中

**这是一个"幽灵数据复活"现象**：用户以为已经切换走 processor 了，旧配置应该被清除，但实际上它们一直留在磁盘上，切回来时自动复活。

### 4.6 为什么没有清理？

最可能的原因是**设计意图上的保守选择**：
- 自动删除旧配置可能导致用户数据丢失（万一用户只是临时切换想回来呢？）
- processor config 文件本身很小（几 KB），不会造成存储压力
- 如果要清理，需要明确记录"前一个 processor 是什么"，然后在切换时删除对应的 JSON，增加了复杂度

但缺少对用户的提示或手动清理入口确实是一个 UX 缺陷。

---

## 附录：发现的两个 Bug

### Bug 1：扁平型 processor_config 在 GET 时不回填

- **位置**：[edit.py#L145](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L145)
- **条件**：`if not isinstance(config_value, dict): continue`
- **影响**：`image_ssim_diff` 等使用扁平字段的 processor，GET 编辑页时已保存的配置不会显示
- **修复思路**：对扁平字段（`config_value` 不是 dict 的情况），直接找 `getattr(form, f'processor_config_{config_key}')` 并设置 `.data`

### Bug 2：切换 processor 后旧 JSON 无清理

- **位置**：无对应代码（缺失逻辑）
- **影响**：旧 processor 的 JSON 文件永久残留，切回时自动复活
- **修复思路**：
  - 在 UI POST / API PUT 中，保存新 processor 配置前，记录旧 processor 名并删除对应的 JSON
  - 或者在 Watch 对象上新增 `cleanup_unused_processor_configs()` 方法，在 commit 时调用
