# Changedetection.io 模板复用机制源码分析

## 一、核心问题澄清

在开始分析之前，需要先澄清一个常见误解：**列表页（watch-overview.html）与详情/编辑页（edit.html）之间直接共享的模板片段非常有限**，大量的 include 级复用发生在 **Watch 编辑页 ↔ Tag 编辑页** 这两个"编辑型页面"之间。

三页面的复用关系如下：

```
                        ┌──────────────────────┐
                        │   全局共享层 (所有页面)│
                        │  base.html / menu.html│
                        │  _helpers.html (宏)   │
                        │  context_processor    │
                        └──────────┬───────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼─────────┐ ┌──────▼────────┐ ┌─────────▼──────────┐
    │  列表页            │ │ Watch 编辑页   │ │  Tag 编辑页        │
    │  watch-overview    │ │  edit.html     │ │  edit-tag.html      │
    │                    │ │                │ │                     │
    │  仅使用:           │ │  使用:         │ │  使用:              │
    │  - render_field    │ │  - include_llm │ │  - include_llm      │
    │  - render_simple_  │ │    _intent     │ │    _intent          │
    │    field           │ │  - include_    │ │  - include_         │
    │  - render_nolabel_ │ │    subtract    │ │    subtract         │
    │    field           │ │  - text-       │ │  - text-options     │
    │                    │ │    options     │ │                     │
    │                    │ │  - render_     │ │  - render_          │
    │                    │ │    common_     │ │    common_settings_ │
    │                    │ │    settings_   │ │    form             │
    │                    │ │    form        │ │                     │
    └────────────────────┘ └──────┬────────┘ └─────────┬──────────┘
                                 │                       │
                                 └───────────┬───────────┘
                                             │
                                    这两层才是 include
                                    片段复用的核心区
```

---

## 二、三层模板复用体系

| 层级 | 实现方式 | 契约形式 | 典型代表 |
|-----|---------|---------|---------|
| L1 布局层 | `{% extends %}` | 约定 block 名 | `base.html` |
| L2 片段层 | `{% include %}` | 隐式上下文 + 预 set 变量 | `include_llm_intent.html`、`include_subtract.html`、`text-options.html` |
| L3 组件层 | `{% macro %}` | 显式参数列表 | `_helpers.html`、`_common_fields.html`、`_stab.html` |

### 2.1 L1 布局层：base.html 的全域共享

所有 16 个页面模板均通过 `{% extends 'base.html' %}` 继承基础布局。

**变量来源 — 零成本全局注入**：

base.html 中引用的大量变量**不需要每个视图单独传递**，而是通过三种机制自动注入：

**1. `@app.context_processor`（请求级别）**

位置：[`changedetectionio/__init__.py:628-638`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/__init__.py#L628-L638)

```python
@app.context_processor
def inject_template_globals():
    return dict(
        right_sticky="v"+__version__,
        new_version_available=app.config['NEW_VERSION_AVAILABLE'],
        has_password=datastore.data['settings']['application']['password'] != False,
        socket_io_enabled=datastore.data['settings']['application'].get('ui', {}).get('socket_io_enabled', True),
        all_paused=datastore.data['settings']['application'].get('all_paused', False),
        all_muted=datastore.data['settings']['application'].get('all_muted', False),
        llm_configured=bool(_get_llm_config(datastore)),
    )
```

**2. `jinja_env.globals.update()`（应用级别）**

位置：[`changedetectionio/flask_app.py:518-523`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/flask_app.py#L518-L523)

```python
app.jinja_env.globals.update({
    '_': gettext,
    'get_locale': get_locale,
    'get_flag_for_locale': get_flag_for_locale,
    'available_languages': available_languages,
})
```

**3. `@app.template_global` / `@app.template_filter`（函数/过滤器）**

注册了 20+ 个全局函数和过滤器，如：
- `get_darkmode_state()`、`get_css_version()`、`get_socketio_path()` — [`flask_app.py:191-218`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/flask_app.py#L191-L218)
- `is_checking_now()`、`get_watch_queue_position()` — [`flask_app.py:233-260`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/flask_app.py#L233-L260)
- `format_number_locale`、`format_last_checked_time`、`pagination_slice` — [`flask_app.py:221-414`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/flask_app.py#L221-L414)

**共享片段 `menu.html`** 被 include 了两次（桌面端 + 移动端抽屉），完全依赖全局上下文，无需视图传参。

---

## 三、L2 片段层：变量传递的四种模式

这是本次分析的重点 —— Watch 编辑页与 Tag 编辑页共享的三个 include 片段，采用了四种不同的变量传递模式。

### 3.1 模式一：视图显式传参 + include 隐式继承

**原理**：视图通过 `render_template(key=value)` 传入变量，include 片段直接使用父模板上下文中的同名变量。

**代表**：`form` 变量、`watch` 变量、`settings_application` 变量。

| 变量 | Watch 编辑页来源 | Tag 编辑页来源 | 用途 |
|-----|----------------|---------------|-----|
| `form` | `template_args['form']` — [`edit.py:327`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L327-L327) | `template_args['form']` — [`tags/__init__.py:184`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L184-L184) | WTForms 表单对象，提供所有字段 |
| `watch` | `template_args['watch']` = Watch 对象 — [`edit.py:344`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L344-L344) | `template_args['watch']` = Tag 对象 — [`tags/__init__.py:185`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L185-L185) | 当前编辑对象（Duck Typing） |
| `settings_application` | `datastore.data['settings']['application']` — [`edit.py:338`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L338-L338) | 同左 — [`tags/__init__.py:227`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L227-L227) | 系统设置引用 |
| `llm_configured` | `bool(_get_llm_config(datastore))` — [`edit.py:352`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L352-L352) | 同左 — [`tags/__init__.py:187`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L187-L187) | LLM 是否配置 |

> **注意**：Tag 编辑页中 `watch` 变量实际上是 **Tag 对象**。这是一种 Duck Typing 复用策略 —— 只要对象有 include 片段需要的字段（如 `.get('processor')`、`.get('llm_prefilter')`），就可以"冒充" Watch 对象被复用。

### 3.2 模式二：include 前 `{% set %}` 预设置变量

**原理**：调用方模板在 include 之前，用 `{% set var = value %}` 在当前上下文设置变量，include 片段直接读取。

这是最容易被忽略的隐式契约模式。

**代表**：`has_tag_filters_extra` 变量。

**Watch 编辑页**（[`edit.html:40`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/templates/edit.html#L40-L40)）：
```jinja2
{% set has_tag_filters_extra="WARNING: Watch has tag/groups set with special filters\n" if has_special_tag_options else '' %}
```

**Tag 编辑页**（[`edit-tag.html:16`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/templates/edit-tag.html#L16-L16)）：
```jinja2
{% set has_tag_filters_extra='' %}
```

**使用方** `include_subtract.html`（[`include_subtract.html:2-7`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/templates/edit/include_subtract.html#L2-L7)）：
```jinja2
{% set field = render_field(form.include_filters,
    rows=5,
    placeholder=has_tag_filters_extra+"#example\nxpath://body/div/span[contains(@class, 'example-class')]",
    class="m-d")
%}
```

**隐式契约风险**：如果调用方忘记 `{% set %}` 这个变量，Jinja2 会抛出 `UndefinedError`，而非静默失败。这是一种"约定优于配置"的模式，但缺乏编译期检查。

### 3.3 模式三：存在性探测 — `variable is defined`

**原理**：include 片段通过 `{% if var is defined %}` 判断变量是否存在，从而自适应不同的调用上下文。

**代表**：`llm_group_overrides` 变量 —— 这是区分 Watch 编辑模式和 Tag 编辑模式的**核心开关**。

**Watch 编辑页**：传递了 `llm_group_overrides` — [`edit.py:353`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L353-L353)
```python
'llm_group_overrides': _resolve_llm_group_overrides(watch, datastore),
```

**Tag 编辑页**：**不传递** `llm_group_overrides`。

**使用方** `include_llm_intent.html`（[`include_llm_intent.html:24-30`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/templates/edit/include_llm_intent.html#L24-L30)）：
```jinja2
{# Processor check only applies in watch-edit context (llm_group_overrides present). #}
{# In tag/group edit context the AI section is always visible.                        #}
{% if llm_group_overrides is defined %}
    {% set is_text_json_diff = not watch.get('processor') or watch.get('processor') == 'text_json_diff' %}
{% else %}
    {% set is_text_json_diff = true %}
{% endif %}
```

**双条件分级判断**是这个片段最精巧的设计：

| 判断条件 | Watch 模式 | Tag 模式 | 控制内容 |
|---------|-----------|---------|---------|
| `llm_group_overrides is defined` | ✅ True | ❌ False | 是否做 processor 类型检查、是否显示"来自组"的占位符 |
| `watch is defined and watch` | ✅ True | ✅ True（注意：Tag 模式下 watch 也被定义了！） | 是否显示例子、是否显示 llm_prefilter 提示、描述文案选择 |

> **关键修正**：之前的分析误认为 `watch is defined` 是区分模式的依据，但实际上 Tag 模式下 `watch` 变量**也被设置为 Tag 对象**。真正的模式开关是 `llm_group_overrides is defined`。

### 3.4 模式四：宏的显式参数列表

**原理**：通过 `{% macro name(arg1, arg2) %}` 定义明确的参数接口，调用时必须传参。

这是最安全、最可维护的复用模式。

**代表**：`render_common_settings_form` 宏。

定义（[`_common_fields.html:151`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/templates/_common_fields.html#L151-L151)）：
```jinja2
{% macro render_common_settings_form(form, emailprefix, settings_application, extra_notification_token_placeholder_info) %}
```

三个调用方都严格按照参数列表传参：
- Watch 编辑页：[`edit.html:304`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/templates/edit.html#L304-L304)
- Tag 编辑页：[`edit-tag.html:139`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/templates/edit-tag.html#L139-L139)
- 系统设置页：[`settings.html:108`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/settings/templates/settings.html#L108-L108)

---

## 四、列表页 ↔ 详情页的复用边界

### 4.1 共享的部分

| 共享层级 | 具体内容 | 共享方式 |
|---------|---------|---------|
| 布局 | `base.html`（含头部、导航、页脚、样式/脚本） | `{% extends 'base.html' %}` |
| 导航 | `menu.html` | 通过 base.html 间接 include |
| 宏 | `render_field`、`render_simple_field`、`render_nolabel_field` | `{% from '_helpers.html' import ... %}` |
| 全局上下文 | `has_password`、`all_paused`、`llm_configured` 等 7 个变量 | context_processor |
| 全局函数 | `get_darkmode_state()`、`is_checking_now()` 等 20+ 个 | template_global |
| 全局过滤器 | `format_last_checked_time`、`format_timestamp_timeago`、`pagination_slice` 等 | template_filter |

### 4.2 不共享的部分（列表页特有）

- 快速添加 Watch 表单（`#form-quick-watch-add`）
- 批量操作工具栏（暂停/静音/重检/删除）
- Watch 列表表格（分页、排序、搜索过滤）
- 处理器徽章 CSS（内联生成）
- Favicon 懒加载逻辑
- LLM 快速输入框（简化版，不使用 include 片段）

### 4.3 不共享的部分（详情页特有）

- Tab 页切换系统（General / Request / Browser Steps / AI / Filters / Notifications / Stats）
- 所有 `include` 级共享片段（LLM Intent、Subtract、Text Options）
- 通知设置完整表单（`render_common_settings_form`）
- 可视化选择器、Browser Steps 编辑器等复杂交互组件

---

## 五、Tag 编辑页 ↔ Watch 编辑页的复用边界

这是模板复用最密集的区域，两个页面共享了三个完整的 include 片段和一个宏。

### 5.1 共享全景

| 共享片段 | 复用方式 | 适配机制 |
|---------|---------|---------|
| `include_llm_intent.html` | include | `llm_group_overrides is defined` 切换模式 + `watch` Duck Typing |
| `include_subtract.html` | include | `has_tag_filters_extra` 预 set 变量 + `jq_support` 存在性探测 |
| `text-options.html` | include | 纯表单字段渲染，依赖 `form` 对象的字段兼容性 |
| `render_common_settings_form` | macro import | 显式参数列表 |

### 5.2 Duck Typing 适配：Tag 对象如何"冒充" Watch 对象

Tag 编辑页中（[`tags/__init__.py:182-188`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L182-L188)）：

```python
template_args = {
    'data': default,       # Tag 对象
    'form': form,
    'watch': default,      # 关键：把 Tag 对象赋值给 watch 变量
    ...
}
```

这样做的前提是 Tag 对象和 Watch 对象在 include 片段访问的字段上保持了**接口一致性**：

| 字段访问 | Watch 对象 | Tag 对象 | 兼容性 |
|---------|-----------|---------|-------|
| `watch.get('processor')` | ✅ 有 | ✅ 有（tag.processor = 'restock_diff'） | ✅ |
| `watch.get('llm_prefilter')` | ✅ 有 | ⚠️ 可能有 / 可能没有（取决于 form） | `get()` 安全 |
| `watch.get('llm_intent')` | ✅ 有 | ✅ 有 | ✅ |
| `watch.label` | ✅ 有 | ✅ 有 | ✅ |

> **边界风险**：这种隐式接口契约没有编译期保障。如果未来 Watch 新增一个字段而 Tag 没有对应添加，include 片段中对应的 `watch.get('new_field')` 会静默返回 None，可能导致难以排查的显示 bug。

### 5.3 各自独立的部分

| 功能 | Watch 编辑页 | Tag 编辑页 |
|-----|-------------|-----------|
| URL 编辑 | ✅ 完整 URL + 变量支持 | ❌ 改为 `url_match_pattern` 通配符 |
| 处理器选择 | ✅ 可切换 text_json_diff / restock_diff 等 | ❌ 固定 restock_diff |
| Browser Steps | ✅ 完整编辑器 | ❌ 无 |
| 可视化选择器 | ✅ 有 | ❌ 无 |
| 请求设置 | ✅ Headers / Cookies / 代理 | ❌ 无 |
| 统计信息 | ✅ 详细检查统计 | ❌ 无 |
| 标签颜色 | ❌ 无 | ✅ 颜色选择器 |
| 匹配的 Watch 列表 | ❌ 无 | ✅ 显示当前匹配此 tag 的 watches |

---

## 六、避免重复渲染的五种策略

### 6.1 策略一：CSS visibility 切换（内容一次渲染，前端切换）

**应用场景**：Tab 页切换、子 Tab 切换

**实现**：
- 顶层 Tab：`tabs.js` 控制 `.tab-pane-inner` 的显示/隐藏
- 子 Tab（`_stab.html`）：[`_stab.html:20`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/templates/_stab.html#L20-L20) 注释明确说明：
  > "Hidden panes use visibility:hidden so form fields inside still submit."

**优势**：
- 所有 Tab 内容在服务器端**只渲染一次**
- 前端切换零延迟
- 隐藏 Tab 中的表单字段仍然会被提交（visibility:hidden 不影响 form 提交）

### 6.2 策略二：前端 JS 条件显示（基于 data 属性）

**应用场景**：编辑页中根据处理器类型、fetcher 类型动态显示/隐藏字段

**实现**：字段带有 `data-visible-for="fetch_backend=html_webdriver"` 等属性，由 `watch-settings.js` 根据当前选择动态切换 CSS display。

**优势**：
- 服务器端只渲染一次所有可能的字段
- 用户切换处理器类型时无需重新请求

### 6.3 策略三：分页切片（列表页渲染量控制）

**应用场景**：Watch 列表分页

**实现**：[`flask_app.py:291-297`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/flask_app.py#L291-L297)

```python
@app.template_filter('pagination_slice')
def _jinja2_filter_pagination_slice(arr, skip):
    per_page = datastore.data['settings']['application'].get('pager_size', 50)
    if per_page:
        return arr[skip:skip + per_page]
    return arr
```

模板中使用（[`watch-overview.html:239`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/watchlist/templates/watch-overview.html#L239-L239)）：
```jinja2
{%- for watch in (watches|sort(...))|pagination_slice(skip=pagination.skip) -%}
```

**效果**：默认每页 50 条，避免一次性渲染数千行 DOM。

### 6.4 策略四：Favicon 懒加载（前端延迟请求）

**应用场景**：列表页 Favicon 图片

**实现**：[`watch-overview.html:18-57`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/watchlist/templates/watch-overview.html#L18-L57)

使用 Intersection Observer API，真实 URL 存在 `data-src` 属性中，仅当图标进入视口才设置 `src` 发起请求。初始使用 SVG 占位符。

**效果**：
- 首屏减少 N 个 HTTP 请求（N = 不在视口的 watch 数量）
- 服务器端无需为不在首屏的 watch 生成/读取 favicon

### 6.5 策略五：插件内容预渲染（独立 Jinja2 Environment）

**应用场景**：处理器插件的 `extra_form_content`

**实现**：[`edit.py:356-363`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L356-L363)

```python
templates_dir = str(importlib.resources.files("changedetectionio").joinpath('templates'))
env = Environment(loader=FileSystemLoader(templates_dir))
template = env.from_string(form.extra_form_content())
included_content = template.render(**template_args)  # 预渲染为字符串

output = render_template("edit.html",
                         extra_form_content=included_content,  # 字符串传入主模板
                         ...)
```

主模板中只做字符串拼接（[`edit.html:113-115`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/templates/edit.html#L113-L115)）：
```jinja2
{% if extra_form_content %}
    {{ extra_form_content|safe }}
{% endif %}
```

**优势**：
- 插件模板可以 `{% from '_helpers.html' import ... %}` 复用同一套宏
- 插件渲染错误不会污染主模板（异常隔离）
- 主模板避免了动态 include 的模板查找开销

Tag 编辑页采用完全相同的模式 — [`tags/__init__.py:191-214`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L191-L214)

---

## 七、复用边界全景图

```
                              ┌──────────────────────────────┐
                              │    全局注入层 (所有页面)      │
                              │  context_processor (7 vars)   │
                              │  template_global (20+ funcs)  │
                              │  template_filter (10+ filters)│
                              └──────────────┬───────────────┘
                                             │
                              ┌──────────────▼───────────────┐
                              │     布局层 base.html          │
                              │  menu.html (含 2 次 include)  │
                              │  所有页面 extends 继承        │
                              └──────┬───────────┬───────────┘
                                     │           │
                     ┌───────────────┘           └───────────────┐
                     │                                           │
          ┌──────────▼──────────┐                     ┌──────────▼──────────┐
          │  列表页             │                     │  编辑型页面家族      │
          │  watch-overview     │                     │                     │
          │                     │                     │  ┌───────────────┐   │
          │  宏:                │                     │  │ Watch 编辑页  │   │
          │   - render_field    │                     │  │  edit.html    │   │
          │   - render_simple_  │                     │  └───────┬───────┘   │
          │     field           │                     │          │           │
          │   - render_nolabel_ │                     │  ┌───────▼───────┐   │
          │     field           │                     │  │ Tag 编辑页    │   │
          │                     │                     │  │ edit-tag.html │   │
          │  无 include 片段    │                     │  └───────────────┘   │
          └─────────────────────┘                     │                     │
                                                      │  三者共享:          │
                                                      │   - include_llm_    │
                                                      │     intent          │
                                                      │   - include_        │
                                                      │     subtract        │
                                                      │   - text-options    │
                                                      │   - render_common_  │
                                                      │     settings_form   │
                                                      └─────────────────────┘
```

---

## 八、设计评价与改进建议

### 8.1 设计优点

| 模式 | 评价 |
|-----|------|
| context_processor 全局注入 | ✅ 消除重复传参，每个视图函数减少约 7 个变量的传递 |
| 宏的显式参数 | ✅ 接口契约清晰，编译期可发现参数缺失 |
| `llm_group_overrides is defined` 模式切换 | ✅ 巧妙利用变量存在性实现双模式适配，单一模板服务两种场景 |
| `has_tag_filters_extra` 预 set | ⚠️ 隐式契约，依赖调用方自觉，缺少编译期检查 |
| Watch/Tag Duck Typing | ⚠️ 最大化代码复用，但隐式接口无保障 |
| 插件独立 Environment 预渲染 | ✅ 错误隔离 + 宏复用 + 性能优化 |
| CSS visibility Tab 切换 | ✅ 一次渲染，零延迟切换，表单完整提交 |

### 8.2 可改进点

1. **include 片段的自包含性**：`include_subtract.html` 和 `text-options.html` 可以在文件头部自行 `{% from '_helpers.html' import render_field %}`，减少对调用方的隐式依赖。目前调用方必须自行 import 所需的宏，否则会报 `UndefinedError`。

2. **跨视图上下文构建的重复代码**：`settings_application`、`emailprefix`、`extra_notification_token_placeholder_info`、`timezone_default_config` 这 4 个变量在 3 个视图中用相同代码构建，可以提取为 `build_form_template_context(datastore)` 辅助函数。

3. **Tag 对象的 Duck Typing 风险**：建议为 Tag 和 Watch 共享字段定义一个显式的 Protocol 或基类，至少在文档中明确列出 include 片段依赖的字段清单。

4. **列表页 Tag 颜色 CSS 的服务端生成**：目前每个标签的颜色 CSS 在列表页内联生成（[`watch-overview.html:70-103`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/watchlist/templates/watch-overview.html#L70-L103)），每次请求都重新生成。如果 tag 数量较多，可以考虑缓存或移到 CSS 文件中。
