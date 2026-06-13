# changedetection.io 监控项编辑页——多页签表单状态保持机制分析

## 1. 整体架构概览

监控项（Watch）编辑页是一个**单表单、多页签**的经典 Flask + WTForms 应用。核心思路可以概括为：

> **所有页签字段同属一个 `<form>`，页签切换只是 CSS 显隐，并非 DOM 销毁/重建。**

因此"切换页签时已填内容不丢失"并不是某种复杂的状态管理，而是天然地由 HTML 表单机制保证——输入框始终在 DOM 中，只是视觉上被隐藏了。

---

## 2. 页签切换时表单状态不丢失的机制

### 2.1 关键代码位置

| 角色 | 文件 |
|------|------|
| 后端路由 & 表单初始化 | [edit.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py) |
| 表单类定义 | [forms.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/forms.py) |
| 编辑页 HTML 模板 | [edit.html](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/templates/edit.html) |
| 页签切换 JS | [tabs.js](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/static/js/tabs.js) |
| 页签显隐 CSS | [styles.scss](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/static/styles/scss/styles.scss) |
| Watch 数据模型 | [Watch.py](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py) |

### 2.2 单表单结构

在 [edit.html#L71-L72](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/templates/edit.html#L71-L72) 中，整个编辑页只有一个 `<form>`：

```html
<form class="pure-form pure-form-stacked"
      action="{{ url_for('ui.ui_edit.edit_page', uuid=uuid, ...) }}" method="POST">
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
    <!-- 所有页签的内容都在这个 form 内部 -->
    <div class="tab-pane-inner" id="general">...</div>
    <div class="tab-pane-inner" id="request">...</div>
    <div class="tab-pane-inner" id="notifications">...</div>
    <div class="tab-pane-inner" id="filters-and-triggers">...</div>
    <!-- ... -->
</form>
```

所有页签的 `<div class="tab-pane-inner">` 都是同一个 `<form>` 的子元素。用户在任意页签填写的 `<input>`、`<textarea>`、`<select>` 都属于同一表单，浏览器原生维护其值。

### 2.3 CSS `:target` 伪类驱动的显隐

页签的显示/隐藏完全由 CSS `:target` 伪类控制，定义在 [styles.scss#L814-L822](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/static/styles/scss/styles.scss#L814-L822)：

```scss
.tab-pane-inner {
  &:not(:target) {
    display: none;   // 非 target 的页签隐藏
  }
  &:target {
    display: block;  // 当前 target 的页签显示
  }
}
```

当用户点击页签导航（如 `<a href="#general">`），浏览器 URL hash 变为 `#general`，对应 `id="general"` 的 div 成为 `:target`，从而 `display: block`；其余页签不再是 `:target`，变为 `display: none`。

**关键点：`display: none` 只是不渲染，并不销毁 DOM 节点。** 所有 `<input>` 元素仍在 DOM 中，其 `value` 属性完全保留。因此切换页签不会丢失任何已填内容。

### 2.4 JS 层的页签激活逻辑

[tabs.js](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/static/js/tabs.js) 的职责非常轻量：

1. **监听 `hashchange` 事件**：当 URL hash 变化时，移除所有页签导航的 `active` class，然后给当前 hash 对应的导航项添加 `active`。
2. **初始化**：如果没有 hash，自动跳转到第一个页签。
3. **错误定位**：如果表单验证失败，自动跳转到包含错误信息的页签（`focus_error_tab()`）。

JS 层**不做任何表单数据的读取/存储/恢复**，它只管视觉状态（active class）。

### 2.5 小结：状态不丢失的三层保障

| 层次 | 机制 | 效果 |
|------|------|------|
| HTML | 所有字段在同一个 `<form>` 内 | 浏览器原生维护所有 input 的值 |
| CSS | `:target` 伪类仅做 `display: none/block` | DOM 节点不销毁，值自然保留 |
| JS | 仅操控导航 `active` class | 不触碰表单数据 |

---

## 3. 提交时分散字段汇成完整配置的逻辑

### 3.1 WTForms 表单初始化——把 Watch 数据映射为表单字段

在 [edit.py#L118-L122](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L118-L122)：

```python
form = form_class(formdata=request.form if request.method == 'POST' else None,
                  data=default,
                  extra_notification_tokens=default.extra_notification_token_values(),
                  default_system_settings=datastore.data['settings'])
```

- **GET 请求**：`formdata=None`，WTForms 从 `data=default`（即 Watch 对象的 `deepcopy`）填充各字段初始值。
- **POST 请求**：`formdata=request.form`，WTForms 从浏览器提交的表单数据填充各字段，覆盖 `data` 中的值。

`form_class` 通常是 [processor_text_json_diff_form](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/forms.py#L827-L899)，它继承自 `commonSettingsForm`，包含了所有页签的字段定义：URL、tags、time_between_check、include_filters、notification_urls、conditions 等，总计 40+ 个字段。

### 3.2 表单验证

提交后首先进入 [edit.py#L180](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L180)：

```python
if request.method == 'POST' and form.validate():
```

WTForms 的 `validate()` 会：
1. 执行每个字段的内置验证器（如 `validateURL`、`ValidateCSSJSONXPATHInput`、`ValidateJinja2Template` 等）。
2. 执行自定义验证方法（如 `processor_text_json_diff_form.validate()` 中检查 GET 请求不能带 body 等）。
3. 如果验证失败，`form.validate()` 返回 `False`，模板重新渲染，错误信息显示在对应字段旁，JS 会自动跳转到有错误的页签。

### 3.3 提取 `form.data`——一个包含所有字段的字典

WTForms 的 `form.data` 属性会自动聚合所有字段的已处理值，返回一个完整的字典。无论字段在哪个页签，只要它是 `form` 的属性，就会出现在 `form.data` 中。

这意味着：**分散在 8 个页签中的字段，在 `form.data` 中自然汇成了一个完整的字典。** 不需要手动"收集"各页签的数据。

### 3.4 处理器配置的分离存储

`processor_config_*` 前缀的字段有特殊处理。在 [edit.py#L193-L195](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L193-L195)：

```python
processor_config_data = processors.extract_processor_config_from_form_data(form.data)
processors.save_processor_config(datastore, uuid, processor_config_data)
```

[extract_processor_config_from_form_data](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/processors/__init__.py#L472-L498) 的逻辑：
1. 遍历 `form.data` 的所有 key。
2. 找到以 `processor_config_` 开头的 key，提取其值到单独的字典（去掉前缀）。
3. **从 `form.data` 中删除这些 key**（in-place 修改），防止它们被写入主存储。

这些处理器配置会保存到独立的 JSON 文件（如 `restock_diff.json`），而非 `watch.json`。

### 3.5 写回 Watch 对象——两步 update

在 [edit.py#L225-L226](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L225-L226)：

```python
datastore.data['watching'][uuid].update(form.data)
datastore.data['watching'][uuid].update(extra_update_obj)
```

- 第一步：`form.data`（此时已剥离 `processor_config_*` 字段）整体 update 到 Watch 对象。因为 Watch 继承自 `dict`，这等效于把所有表单字段值写入 Watch 的字典。
- 第二步：`extra_update_obj` 包含需要特殊处理的字段，如 `consecutive_filter_failures`（重置为 0）、`time_between_check`（从 FormField 提取）、`proxy`（空字符串转 None）、`tags`（从逗号分隔的标签名转为 UUID 列表）等。

### 3.6 特殊字段的处理

| 字段 | 处理逻辑 | 代码位置 |
|------|---------|----------|
| `ignore_text` | 单独赋值，不走 `form.data` 的 update | [edit.py#L198-L199](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L198-L199) |
| `tags` | 逗号分隔的标签名 → 逐个 `add_tag()` 转为 UUID | [edit.py#L216-L223](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L216-L223) |
| `proxy` | 空字符串转为 `None`（表示使用默认） | [edit.py#L202-L203](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L202-L203) |
| `filter_text_*` | 三项全空时回退为默认值（全部 True） | [edit.py#L207-L212](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L207-L212) |
| `processor_config_*` | 从 `form.data` 剥离，单独存为 JSON 文件 | [edit.py#L193-L195](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L193-L195) |

### 3.7 持久化——commit 到磁盘

在 [edit.py#L234-L238](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/blueprint/ui/edit.py#L234-L238)：

```python
watch_class = processors.get_custom_watch_obj_for_processor(form.data.get('processor'))
datastore.data['watching'][uuid] = watch_class(..., default=datastore.data['watching'][uuid])
datastore.data['watching'][uuid].commit()
```

1. 根据选择的 processor 类型重新构造 Watch 对象（确保类型正确）。
2. 调用 `commit()`，[Watch._get_commit_data()](file:///d:/fz/0601-1/solo-dogfeeding/code/63-changedetection.io/changedetectionio/model/Watch.py#L1064-L1093) 会排除 `processor_config_*` 和 `__` 前缀的瞬态字段，将剩余数据原子写入 `watch.json`。

---

## 4. 完整数据流图

```
┌─────────────────────────────────────────────────────────┐
│                    浏览器（8 个页签）                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ General  │ │ Request  │ │ Filters  │ │ Notific. │   │
│  │ url      │ │ method   │ │ include  │ │ urls     │   │
│  │ tags     │ │ headers  │ │ subtract │ │ title    │   │
│  │ title    │ │ body     │ │ ignore   │ │ body     │   │
│  │ ...      │ │ proxy    │ │ trigger  │ │ format   │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│         │           │           │           │            │
│         └───────────┴───────────┴───────────┘            │
│                     │                                    │
│          同一个 <form method="POST">                      │
│          所有字段在 DOM 中始终存在                          │
│          切换页签仅 CSS display:none/block                │
│                     │                                    │
│          点击 Save 按钮 → 浏览器提交所有字段                │
└─────────────────────┼───────────────────────────────────┘
                      │ POST request.form
                      ▼
┌─────────────────────────────────────────────────────────┐
│              后端 edit_page() (edit.py)                   │
│                                                          │
│  1. WTForms 从 request.form 填充 form 对象               │
│     form = form_class(formdata=request.form, data=default)│
│                                                          │
│  2. form.validate() 验证所有字段                          │
│     ├── 字段级验证器 (URL、CSS、Jinja2、Regex...)         │
│     └── 表单级验证 (GET+body、模板语法...)                 │
│                                                          │
│  3. 提取 processor_config_* → 单独存 JSON 文件           │
│     processor_config_data = extract_processor_config()    │
│     save_processor_config(datastore, uuid, config_data)   │
│                                                          │
│  4. form.data 整体 update 到 Watch 对象                   │
│     datastore.data['watching'][uuid].update(form.data)    │
│                                                          │
│  5. extra_update_obj 补充特殊处理字段                     │
│     datastore.data['watching'][uuid].update(extra)        │
│                                                          │
│  6. Watch.commit() 原子写入 watch.json                   │
│     (排除 processor_config_* 和 __ 前缀字段)              │
└─────────────────────────────────────────────────────────┘
```

---

## 5. 设计要点总结

1. **单表单 + CSS 显隐**是整套状态保持的基石。不需要 JS 状态管理、不需要 localStorage、不需要 SPA 框架，利用浏览器原生表单机制即可。

2. **WTForms 是数据汇合的核心**。`form.data` 自动聚合所有字段的值为一个字典，无论字段在哪个页签。这避免了手动拼接各页签数据的复杂度。

3. **处理器配置的分离存储**体现了关注点分离。`processor_config_*` 字段从 `form.data` 中剥离后独立存为 JSON 文件，而主 Watch 数据存入 `watch.json`，两者生命周期独立。

4. **两步 update 模式**兼顾了通用性和特殊性。`form.data` update 处理大部分字段，`extra_update_obj` update 处理需要转换逻辑的特殊字段（tags 转 UUID、proxy 空值转 None 等）。

5. **验证失败时的用户体验**也依赖单表单结构：WTForms 将错误信息绑定到对应字段，`focus_error_tab()` 自动跳转到有错误的页签，用户无需手动查找。
