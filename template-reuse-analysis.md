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

## 四、Tag 编辑场景的变量清单与缺失边界

这是之前分析的重要遗漏点 —— Tag 编辑页向模板传递的变量比 Watch 编辑页少得多。

### 4.1 Watch 编辑页完整 template_args（28 个变量）

位置：[`edit.py:318-354`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L318-L354)

```python
template_args = {
    'available_processors': processors.available_processors(),
    'available_timezones': sorted(available_timezones()),
    'browser_steps_config': browser_step_ui_config,
    'emailprefix': os.getenv('NOTIFICATION_MAIL_BUTTON_PREFIX', False),
    'extra_classes': ' '.join(c),
    'extra_notification_token_placeholder_info': datastore.get_unique_notification_token_placeholders_available(),
    'extra_processor_config': form.extra_tab_content(),
    'extra_title': f" - {gettext('Edit')} - {watch.label}",
    'form': form,
    'has_default_notification_urls': True if len(datastore.data['settings']['application']['notification_urls']) else False,
    'has_extra_headers_file': len(datastore.get_all_headers_in_textfile_for_watch(uuid=uuid)) > 0,
    'has_special_tag_options': _watch_has_tag_options_set(watch=watch),
    'jq_support': jq_support,  # True/False，取决于 import jq 是否成功
    'playwright_enabled': os.getenv('PLAYWRIGHT_DRIVER_URL', False),
    'app_rss_token': app_rss_token,
    'rss_uuid_feed' : {'label': watch.label, 'url': ...},
    'settings_application': datastore.data['settings']['application'],
    'ui_edit_stats_extras': collect_ui_edit_stats_extras(watch),
    'visual_selector_data_ready': datastore.visualselector_data_is_ready(watch_uuid=uuid),
    'timezone_default_config': datastore.data['settings']['application'].get('scheduler_timezone_default'),
    'using_global_webdriver_wait': not default['webdriver_delay'],
    'uuid': uuid,
    'watch': watch,
    'capabilities': capabilities,
    'auto_applied_tags': {tag_uuid: tag ...},
    'llm_configured': bool(_get_llm_config(datastore)),
    'llm_group_overrides': _resolve_llm_group_overrides(watch, datastore),
}
```

### 4.2 Tag 编辑页完整 template_args（仅 5 个变量）

位置：[`tags/__init__.py:182-188`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L182-L188)

```python
template_args = {
    'data': default,                              # Tag 对象
    'form': form,
    'watch': default,                             # Tag 对象赋值给 watch
    'extra_notification_token_placeholder_info': datastore.get_unique_notification_token_placeholders_available(),
    'llm_configured': bool(_get_llm_config(datastore)),
}
```

**render_template 额外补充（4 个变量）**：
```python
output = render_template("edit-tag.html",
                         extra_form_content=included_content,
                         extra_tab_content=form.extra_tab_content() if form.extra_tab_content() else None,
                         matching_watches=matching_watches,
                         settings_application=datastore.data['settings']['application'],
                         **template_args
                         )
```

### 4.3 Tag 编辑页缺失的关键变量清单

| 缺失变量 | Watch 页有 | Tag 页有 | 影响范围 | 未定义时的行为 |
|---------|-----------|---------|---------|---------------|
| `jq_support` | ✅ | ❌ | `include_subtract.html:20`、`:36` | Jinja2 Undefined 在 bool 上下文为 **False**，不报错但 jq 相关提示不显示 |
| `emailprefix` | ✅ | ❌ | `edit-tag.html:12`、`render_common_settings_form` 参数 2 | `{% if emailprefix %}` 为 False，邮件按钮不显示；但宏调用仍传参，宏内 `{% if emailprefix %}` 也为 False |
| `has_default_notification_urls` | ✅ | ❌ | `edit-tag.html:131` | `{% if has_default_notification_urls %}` 为 False，系统级通知 URL 警告不显示 |
| `llm_group_overrides` | ✅ | ❌ | `include_llm_intent.html:26`、`:49`、`:88` | `{% if llm_group_overrides is defined %}` 为 False，走 Tag 模式分支 |
| `has_special_tag_options` | ✅ | ❌ | `edit-tag.html:40`（通过 `has_tag_filters_extra` 间接使用） | Tag 页在模板中直接 `{% set has_tag_filters_extra='' %}`，绕开了对 `has_special_tag_options` 的依赖 |

### 4.4 Tag 对象的 processor 字段真实值

**关键事实修正**：Tag 对象默认继承 `watch_base` 类，其默认值是 `'processor': 'text_json_diff'`（[`model/__init__.py:225`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/model/__init__.py#L225-L225)）。仅在 POST 提交时（[`tags/__init__.py:254`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L254-L254)）才会设置为 `'restock_diff'`。

这意味着：
- **GET 请求（首次加载编辑页）**：Tag 对象的 `processor` 字段是 `'text_json_diff'`
- **POST 保存后重新加载**：Tag 对象的 `processor` 字段是 `'restock_diff'`

但在 `include_llm_intent.html` 中，由于 Tag 模式下 `llm_group_overrides is defined` 为 False，`is_text_json_diff` 始终被设置为 `true`，`watch.get('processor')` 的值实际上**不会影响分支走向**。

---

## 五、三类共享片段在两类编辑页中的真实分支路径

### 5.1 `include_llm_intent.html` — 完整分支路径对比

#### Watch 编辑模式路径

```
llm_group_overrides is defined → ✅ True
├─ is_text_json_diff = not watch.get('processor') or watch.get('processor') == 'text_json_diff'
│  ├─ 如果 processor 是 text_json_diff → ✅ True
│  └─ 如果是 restock_diff → ❌ False（外层 if 会隐藏整个片段）
│
└─ {% if is_text_json_diff %} → ✅ 显示
   ├─ {% if watch is defined and watch %} → ✅ True
   │  ├─ 显示 "An accurate plain-English description..." 描述
   │  └─ 显示 processor='text_json_diff' 的例子
   │
   ├─ {% if llm_group_overrides.llm_intent %} → 取决于组设置
   │  ├─ 显示 "From group 'XXX': YYY" 占位符
   │  └─ intent_placeholder 设置为组覆盖值
   │  └─ {% elif watch is defined and watch %} → ✅ True
   │     └─ intent_placeholder 为默认值
   │
   ├─ {% if watch is defined and watch %} → ✅ True
   │  └─ 渲染 form.llm_intent 字段
   │
   ├─ {% if watch.get('llm_prefilter') %} → 取决于 watch 设置
   │  └─ 显示 "AI pre-filter active: ..." 提示
   │
   └─ llm_change_summary 区域逻辑同上
```

#### Tag 编辑模式路径

```
llm_group_overrides is defined → ❌ False
├─ is_text_json_diff = true （硬编码为 true，忽略 watch.get('processor') 的值）
│
└─ {% if is_text_json_diff %} → ✅ 始终显示
   ├─ {% if watch is defined and watch %} → ✅ True（watch 是 Tag 对象）
   │  ├─ 显示 "Set a change intent for all watches in this tag/group..." 描述
   │  └─ 显示 tag 模式的例子
   │
   ├─ {% if llm_group_overrides.llm_intent %} → ❌ False（llm_group_overrides 未定义）
   │  └─ 跳过组占位符分支
   │  └─ {% elif watch is defined and watch %} → ✅ True
   │     └─ intent_placeholder 为默认值
   │
   ├─ {% if watch is defined and watch %} → ✅ True
   │  └─ 渲染 form.llm_intent 字段
   │
   ├─ {% if watch.get('llm_prefilter') %} → 取决于 Tag 设置
   │  └─ 显示 "AI pre-filter active: ..." 提示
   │
   └─ llm_change_summary 区域逻辑同上
```

### 5.2 `include_subtract.html` — 完整分支路径对比

#### Watch 编辑模式

```
render_field(form.include_filters, ...) → ✅ 正常渲染
  placeholder = has_tag_filters_extra + "#example..."
  has_tag_filters_extra 由 edit.html:40 预 set，值为 '' 或警告字符串

{% if jq_support %} → ✅ True 或 False（取决于 import jq 是否成功）
└─ 显示 jq 例子（如 "jq: [.[] | .version]"）

{% if jq_support %} → 同上
└─ 帮助文案中包含 "or jq selector" 提示

render_field(form.subtractive_selectors, ...) → ✅ 正常渲染
  placeholder = has_tag_filters_extra + "header, footer..."
```

#### Tag 编辑模式

```
render_field(form.include_filters, ...) → ✅ 正常渲染
  placeholder = '' + "#example..." （has_tag_filters_extra 由 edit-tag.html:16 预 set 为 ''）

{% if jq_support %} → ❌ False（jq_support 未定义，Undefined 在 bool 上下文为 False）
└─ 不显示 jq 例子

{% if jq_support %} → ❌ False
└─ 帮助文案中只有 "CSS, JSONPath, XPath"，没有 "jq selector"

render_field(form.subtractive_selectors, ...) → ✅ 正常渲染
```

### 5.3 `text-options.html` — 变量可用性核查

| 依赖项 | Watch 编辑页 | Tag 编辑页 | 验证 |
|-------|-------------|-----------|-----|
| `render_field` 宏 | ✅ import | ✅ import | `edit.html:7` / `edit-tag.html:3` 都 import 了 |
| `render_ternary_field` 宏 | ✅ import | ✅ import | `edit.html:7` / `edit-tag.html:3` 都 import 了 |
| `form.trigger_text` | ✅ 有字段 | ✅ 有字段 | 继承自 `processor_text_json_diff_form` |
| `form.ignore_text` | ✅ 有字段 | ✅ 有字段 | 同上 |
| `form.strip_ignored_lines` | ✅ 有字段 | ✅ 有字段 | 同上 |
| `form.text_should_not_be_present` | ✅ 有字段 | ✅ 有字段 | 同上 |
| `form.extract_lines_containing` | ✅ 有字段 | ✅ 有字段 | 同上 |
| `form.extract_text` | ✅ 有字段 | ✅ 有字段 | 同上 |

**字段继承链**：`group_restock_settings_form` → `restock_settings_form` → `processor_settings_form` → `processor_text_json_diff_form` → `commonSettingsForm`。所有字段在两边都可用。

### 5.4 `render_common_settings_form` 宏 — 参数传递完整性

宏签名（[`_common_fields.html:151`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/templates/_common_fields.html#L151-L151)）：
```jinja2
{% macro render_common_settings_form(form, emailprefix, settings_application, extra_notification_token_placeholder_info) %}
```

| 参数 | Watch 编辑页 | Tag 编辑页 | 宏内行为 |
|-----|-------------|-----------|---------|
| `form` | ✅ 传 | ✅ 传 | 正常渲染字段 |
| `emailprefix` | ✅ 传（`os.getenv('NOTIFICATION_MAIL_BUTTON_PREFIX', False)`） | ❌ **未传**（Undefined） | 宏内 `{% if emailprefix %}` 为 False，邮件通知按钮不显示 |
| `settings_application` | ✅ 传 | ✅ 传 | 正常获取系统默认的 notification_title / notification_body 占位符 |
| `extra_notification_token_placeholder_info` | ✅ 传 | ✅ 传 | 正常显示可用占位符列表 |

**边界风险**：Tag 编辑页虽然没有传递 `emailprefix`，但仍然将它作为第 2 个参数传入了宏调用（[`edit-tag.html:139`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/templates/edit-tag.html#L139-L139)）。此时 Jinja2 会将 Undefined 值传入宏。宏内通过 `{% if emailprefix %}` 安全地判断，不会抛出异常，但邮件按钮功能在 Tag 编辑页实际上被静默禁用了。

---

## 六、列表页 ↔ 详情页的复用边界

### 6.1 共享的部分

| 共享层级 | 具体内容 | 共享方式 |
|---------|---------|---------|
| 布局 | `base.html`（含头部、导航、页脚、样式/脚本） | `{% extends 'base.html' %}` |
| 导航 | `menu.html` | 通过 base.html 间接 include |
| 宏 | `render_field`、`render_simple_field`、`render_nolabel_field` | `{% from '_helpers.html' import ... %}` |
| 全局上下文 | `has_password`、`all_paused`、`llm_configured` 等 7 个变量 | context_processor |
| 全局函数 | `get_darkmode_state()`、`is_checking_now()` 等 20+ 个 | template_global |
| 全局过滤器 | `format_last_checked_time`、`format_timestamp_timeago`、`pagination_slice` 等 | template_filter |

### 6.2 不共享的部分（列表页特有）

- 快速添加 Watch 表单（`#form-quick-watch-add`）
- 批量操作工具栏（暂停/静音/重检/删除）
- Watch 列表表格（分页、排序、搜索过滤）
- 处理器徽章 CSS（内联生成）
- Favicon 懒加载逻辑
- LLM 快速输入框（简化版，不使用 include 片段）

### 6.3 不共享的部分（详情页特有）

- Tab 页切换系统（General / Request / Browser Steps / AI / Filters / Notifications / Stats）
- 所有 `include` 级共享片段（LLM Intent、Subtract、Text Options）
- 通知设置完整表单（`render_common_settings_form`）
- 可视化选择器、Browser Steps 编辑器等复杂交互组件

---

## 七、避免重复渲染的策略与失败边界

### 7.1 策略一：CSS visibility 切换（内容一次渲染，前端切换）

**应用场景**：Tab 页切换、子 Tab 切换

**实现**：
- 顶层 Tab：`tabs.js` 控制 `.tab-pane-inner` 的显示/隐藏
- 子 Tab（`_stab.html`）：[`_stab.html:20`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/templates/_stab.html#L20-L20) 注释明确说明：
  > "Hidden panes use visibility:hidden so form fields inside still submit."

**优势**：
- 所有 Tab 内容在服务器端**只渲染一次**
- 前端切换零延迟
- 隐藏 Tab 中的表单字段仍然会被提交（visibility:hidden 不影响 form 提交）

### 7.2 策略二：前端 JS 条件显示（基于 data 属性）

**应用场景**：编辑页中根据处理器类型、fetcher 类型动态显示/隐藏字段

**实现**：字段带有 `data-visible-for="fetch_backend=html_webdriver"` 等属性，由 `watch-settings.js` 根据当前选择动态切换 CSS display。

**优势**：
- 服务器端只渲染一次所有可能的字段
- 用户切换处理器类型时无需重新请求

### 7.3 策略三：分页切片（列表页渲染量控制）

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

### 7.4 策略四：Favicon 懒加载（前端延迟请求）

**应用场景**：列表页 Favicon 图片

**实现**：[`watch-overview.html:18-57`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/watchlist/templates/watch-overview.html#L18-L57)

使用 Intersection Observer API，真实 URL 存在 `data-src` 属性中，仅当图标进入视口才设置 `src` 发起请求。初始使用 SVG 占位符。

**效果**：
- 首屏减少 N 个 HTTP 请求（N = 不在视口的 watch 数量）
- 服务器端无需为不在首屏的 watch 生成/读取 favicon

### 7.5 策略五：插件内容预渲染（独立 Jinja2 Environment）

**应用场景**：处理器插件的 `extra_form_content`

#### Watch 编辑页实现（[`edit.py:356-363`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/edit.py#L356-L363)）

```python
included_content = None
if form.extra_form_content():
    templates_dir = str(importlib.resources.files("changedetectionio").joinpath('templates'))
    env = Environment(loader=FileSystemLoader(templates_dir))
    template = env.from_string(form.extra_form_content())
    included_content = template.render(**template_args)  # ⚠️ 无 try-except

output = render_template("edit.html",
                         extra_form_content=included_content,
                         ...)
```

#### Tag 编辑页实现（[`tags/__init__.py:190-214`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L190-L214)）

```python
included_content = {}  # ⚠️ 初始值是空字典 {}，不是 None！
if form.extra_form_content():
    from jinja2 import Environment, FileSystemLoader
    import importlib.resources
    templates_dir = str(importlib.resources.files("changedetectionio").joinpath('templates'))
    env = Environment(loader=FileSystemLoader(templates_dir))
    template_str = """<tag-specific template code>"""  # 预先拼接Tag特有代码
    template_str += form.extra_form_content()
    template = env.from_string(template_str)
    included_content = template.render(**template_args)  # ⚠️ 无 try-except
```

主模板中使用（[`edit.html:113-115`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/ui/templates/edit.html#L113-L115)）：
```jinja2
{% if extra_form_content %}
    {{ extra_form_content|safe }}
{% endif %}
```

**优势**：
- 插件模板可以 `{% from '_helpers.html' import ... %}` 复用同一套宏
- 主模板避免了动态 include 的模板查找开销

#### 失败边界分析

**1. 缺少 try-except 的风险**

两个页面的 `template.render(**template_args)` 调用都**没有 try-except 包裹**。如果插件模板中存在：
- Jinja2 语法错误（如 `{{ unterminated }}`）
- 引用了不存在的变量（如 `{{ nonexistent_var }}`）
- 调用了不存在的宏

会直接抛出异常，导致**整个页面 500 错误**。

**2. Tag 编辑页的 included_content 初始值 bug**

Tag 编辑页中 `included_content = {}`（空字典），而 Watch 编辑页是 `None`。

- 如果 `form.extra_form_content()` 返回 falsy 值（`None`、`""`、`[]` 等），`if` 分支不会执行，`included_content` 保持为 `{}`
- 在模板中 `{% if extra_form_content %}` 会判定为 **True**（因为非空字典 `{}` 是 truthy）
- 然后 `{{ extra_form_content|safe }}` 会将字符串 `"{}"` 输出到页面上！

这是一个真实的 bug：当插件没有 `extra_form_content` 时，Tag 编辑页会在页面上多出一个 `"{}"` 字符串。

**3. 插件模板的变量作用域差异**

插件模板通过 `template.render(**template_args)` 渲染，可以访问 template_args 中的所有变量：
- Watch 编辑页：28 个变量可用
- Tag 编辑页：只有 5 个变量可用

如果插件模板引用了 Watch 编辑页特有的变量（如 `jq_support`、`capabilities`），在 Tag 编辑页中就会抛出 `UndefinedError`，导致整个页面崩溃。

**4. 宏 import 的隐式依赖**

插件模板可以 `{% from '_helpers.html' import render_field %}`，但如果插件模板需要使用 `render_ternary_field` 或其他宏而忘记 import，也会抛出 `UndefinedError`。

---

## 八、复用边界全景图

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
          │   - render_simple_  │                     │  │  28 vars      │   │
          │     field           │                     │  └───────┬───────┘   │
          │   - render_nolabel_ │                     │          │           │
          │     field           │                     │          │ 变量差异    │
          │                     │                     │          ▼           │
          │  无 include 片段    │                     │  ┌───────────────┐   │
          └─────────────────────┘                     │  │ Tag 编辑页    │   │
                                                      │  │ edit-tag.html │   │
                                                      │  │  5 + 4 vars   │   │
                                                      │  └───────────────┘   │
                                                      │                     │
                                                      │  共享 include 片段: │
                                                      │  ┌──────────────────┐│
                                                      │  │ include_llm_     ││
                                                      │  │   intent         ││
                                                      │  │ 分支由 llm_group_││
                                                      │  │  overrides 切换   ││
                                                      │  ├──────────────────┤│
                                                      │  │ include_subtract ││
                                                      │  │  jq_support 在 Tag││
                                                      │  │  页为 Undefined  ││
                                                      │  ├──────────────────┤│
                                                      │  │ text-options     ││
                                                      │  │  字段完全兼容    ││
                                                      │  ├──────────────────┤│
                                                      │  │ render_common_   ││
                                                      │  │   settings_form  ││
                                                      │  │  emailprefix 在  ││
                                                      │  │  Tag 页为 Undef  ││
                                                      │  └──────────────────┘│
                                                      └─────────────────────┘
```

---

## 九、设计评价与改进建议

### 9.1 设计优点

| 模式 | 评价 |
|-----|------|
| context_processor 全局注入 | ✅ 消除重复传参，每个视图函数减少约 7 个变量的传递 |
| 宏的显式参数 | ✅ 接口契约清晰，编译期可发现参数缺失 |
| `llm_group_overrides is defined` 模式切换 | ✅ 巧妙利用变量存在性实现双模式适配，单一模板服务两种场景 |
| `has_tag_filters_extra` 预 set | ⚠️ 隐式契约，依赖调用方自觉，缺少编译期检查 |
| Watch/Tag Duck Typing | ⚠️ 最大化代码复用，但隐式接口无保障 |
| `jq_support` Undefined 优雅降级 | ✅ 利用 Jinja2 Undefined 的 bool 行为实现功能渐进式降级 |
| 插件独立 Environment 预渲染 | ✅ 宏复用 + 性能优化，但 ⚠️ 缺少异常处理 |
| CSS visibility Tab 切换 | ✅ 一次渲染，零延迟切换，表单完整提交 |

### 9.2 已发现的 Bug

1. **Tag 编辑页 `included_content = {}` bug**（[`tags/__init__.py:190`](file:///d:/fz/0601-1/solo-dogfeeding/code/9-changedetection.io/changedetectionio/blueprint/tags/__init__.py#L190-L190)）
   - 初始值应为 `None` 而非 `{}`，否则在没有插件内容时页面会输出 `"{}"`

2. **Tag 编辑页 `emailprefix` 静默缺失**
   - 宏调用虽然传了 Undefined 值，但功能被静默禁用，用户无法得知为什么邮件按钮不显示

### 9.3 可改进点

1. **include 片段的自包含性**：`include_subtract.html` 和 `text-options.html` 可以在文件头部自行 `{% from '_helpers.html' import render_field, render_ternary_field %}`，减少对调用方的隐式依赖。目前调用方必须自行 import 所需的宏，否则会报 `UndefinedError`。

2. **跨视图上下文构建的重复代码**：`settings_application`、`emailprefix`、`extra_notification_token_placeholder_info`、`timezone_default_config` 这 4 个变量在 3 个视图中用相同代码构建，可以提取为 `build_form_template_context(datastore)` 辅助函数。

3. **Tag 对象的 Duck Typing 风险**：建议为 Tag 和 Watch 共享字段定义一个显式的 Protocol 或基类，至少在文档中明确列出 include 片段依赖的字段清单。

4. **插件预渲染的异常处理**：应为 `template.render()` 添加 try-except，失败时记录日志并输出一个占位符（如 "插件加载失败"），而不是让整个页面 500。

5. **Tag 编辑页变量缺失的可观测性**：对于 `emailprefix`、`has_default_notification_urls` 这类功能开关型变量的缺失，要么在 Tag 视图中显式传递 False，要么在模板中添加显式的 `is defined` 检查。
