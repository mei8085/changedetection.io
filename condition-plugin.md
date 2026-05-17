# 条件插件执行链路与界面表单绑定协作逻辑报告

## 1. 插件发现与注册机制

### 1.1 插件系统架构
条件插件系统基于 [Pluggy](https://pluggy.readthedocs.io/) 框架实现，采用**钩子（Hook）驱动**的插件架构。核心文件位于 `changedetectionio/conditions/` 目录。

### 1.2 命名空间与接口定义
- **命名空间**: `changedetectionio_conditions`（`pluggy_interface.py:8`）
- **钩子规范类**: `ConditionsSpec`（`pluggy_interface.py:14-40`）

插件必须实现以下钩子方法（全部为可选）：

| 钩子方法 | 作用 |
|---------|------|
| `register_operators()` | 注册自定义 JSON Logic 运算符 |
| `register_operator_choices()` | 注册运算符在 UI 中的显示选项 |
| `register_field_choices()` | 注册可供条件判断使用的字段选项 |
| `add_data()` | 在运行时向条件评估上下文注入数据 |
| `ui_edit_stats_extras()` | 在编辑页面的统计标签页注入 HTML 内容 |

### 1.3 插件发现与加载流程
插件加载在 `pluggy_interface.py` 中实现，分为三个阶段，按顺序执行：

1. **内置插件注册** (`pluggy_interface.py:49`)：
   ```python
   plugin_manager.register(default_plugin, "default_plugin")
   ```
   - 显式注册，调用 `register()` 方法

2. **目录扫描加载** (`pluggy_interface.py:52-71`)：
   - 扫描 `changedetectionio/conditions/plugins/` 目录
   - 使用 `os.listdir()` 遍历文件，**其返回顺序由文件系统决定，不保证排序**（非字母序、非时间序）
   - 加载所有 `.py` 文件（排除 `__init__.py`），按遍历顺序动态导入并调用 `register()`
   - ⚠️ **关键注意**：`os.listdir()` 顺序在不同操作系统、不同文件系统、甚至同一系统文件增删后都可能变化

3. **外部包发现** (`pluggy_interface.py:74`)：
   ```python
   plugin_manager.load_setuptools_entrypoints(PLUGIN_NAMESPACE)
   ```
   - 支持通过 setuptools entry points 安装的外部插件
   - 遍历顺序由 setuptools 的发现机制决定

### 1.4 现有插件列表
- **default_plugin** (`default_plugin.py`): 提供基础运算符（`starts_with`, `ends_with`, `contains_regex` 等）和字段（`extracted_number`, `page_filtered_text`）
- **levenshtein_plugin** (`plugins/levenshtein_plugin.py`): 提供文本相似度度量（`levenshtein_ratio`, `levenshtein_distance`）
- **wordcount_plugin** (`plugins/wordcount_plugin.py`): 提供词数统计（`word_count`）

## 2. 表单字段与插件参数绑定

### 2.1 表单类结构
```
processor_text_json_diff_form (forms.py:816)
└── conditions_match_logic (RadioField) - 匹配逻辑（ALL/ANY）
└── conditions (FieldList of FormField)
    └── ConditionFormRow (form.py:6)
        ├── field (SelectField) - 字段选择
        ├── operator (SelectField) - 运算符选择
        └── value (StringField) - 比较值
```

### 2.2 动态选项填充
插件注册的字段和运算符在模块加载时自动合并到表单选项中：

```python
# __init__.py:140-152
for plugin in plugin_manager.get_plugins():
    # 注册自定义运算符
    new_ops = plugin.register_operators()
    if isinstance(new_ops, dict):
        CUSTOM_OPERATIONS.update(new_ops)
    
    # 注册运算符显示选项
    new_operator_choices = plugin.register_operator_choices()
    if isinstance(new_operator_choices, list):
        operator_choices.extend(new_operator_choices)
    
    # 注册字段选项
    new_field_choices = plugin.register_field_choices()
    if isinstance(new_field_choices, list):
        field_choices.extend(new_field_choices)
```

⚠️ **关键事实**：`plugin_manager.get_plugins()` 返回的是 **`set` 类型**（Pluggy 官方 API 定义），Python 的 `set` **不保证任何遍历顺序**。因此：
- `operator_choices` 和 `field_choices` 中选项的顺序是**不确定**的
- 每次程序启动时，UI 下拉框中的选项顺序都可能不同
- 这与注册顺序无关，完全取决于 set 的内部哈希遍历顺序

### 2.3 表单验证逻辑
`ConditionFormRow.validate()` (`form.py:25-45`) 实现了智能验证：
- 如果任意字段（operator/field/value）被设置，则三者都必须设置
- 允许空行（全部为空），但不允许部分填写
- 空行在执行时会被 `filter_complete_rules()` 过滤掉

### 2.4 UI 渲染
- 使用宏 `render_conditions_fieldlist_of_formfields_as_table` (`_helpers.html:147`) 渲染条件表格
- 每个条件行包含：字段选择器、运算符选择器、值输入框、操作按钮
- JavaScript (`conditions.js`) 提供动态添加/删除行和单条规则验证功能

## 3. 运行时执行顺序与流程

### 3.1 核心执行入口
`execute_ruleset_against_all_plugins()` (`__init__.py:82-138`) 是条件评估的核心函数。

### 3.2 执行流程详解

```
1. 初始化 EXECUTE_DATA 字典（调用方局部变量）
2. 获取 watch 对象和 conditions 配置
3. 确定逻辑运算符（ALL → and, ANY → or）
4. 过滤掉不完整的规则行
5. 插件数据注入阶段（遍历顺序不确定）：
   ├─ 遍历 plugin_manager.get_plugins() 返回的 set
   ├─ 为每个插件创建线程池执行器
   ├─ 调用 plugin.add_data(current_watch_uuid, application_datastruct, ephemeral_data)，超时 10 秒
   ├─ 若返回字典，调用方通过 EXECUTE_DATA.update() 合并数据
   └─ 插件异常被捕获并记录，但不中断流程
6. 将规则转换为 JSON Logic 格式
7. 使用 jsonLogic 评估规则（传入 EXECUTE_DATA 作为数据上下文）
8. 返回评估结果和执行时数据
```

### 3.3 add_data 接口的输入输出契约
根据 `__init__.py:108-113` 的调用代码，`add_data` 的输入参数严格限定为：

| 参数名 | 类型 | 说明 | 是否必须 |
|-------|------|------|---------|
| `current_watch_uuid` | `str` | 当前监控的 UUID | 是 |
| `application_datastruct` | `dict` | 完整的应用数据结构，包含所有 watch 配置和历史数据 | 是 |
| `ephemeral_data` | `dict` | 临时数据，通常包含 `text` 字段（过滤后的页面文本） | 是 |

**插件能看到的上下文仅限上述三个参数**。特别注意：
- ❌ 插件**不能**访问 `EXECUTE_DATA`（这是调用方的局部变量）
- ❌ 插件**不能**直接访问其他插件返回的数据
- ❌ 插件之间是完全隔离的，无法感知彼此的存在

插件的返回值：
- 应返回一个 `dict`，键为字段名，值为字段值
- 返回 `None` 或非字典类型会被忽略
- 所有返回数据由调用方统一合并到 `EXECUTE_DATA`

### 3.4 规则转换逻辑
`convert_to_jsonlogic()` (`__init__.py:41-79`) 将结构化规则转换为 JSON Logic 格式：

- **标准二元运算符** (`>`, `<`, `==` 等): `{operator: [{"var": field}, value]}`
- **in 运算符**: `{"in": [value, {"var": field}]}`（注意参数顺序反转）
- **一元运算符** (`!`, `!!`, `-`): `{operator: [{"var": field}]}`
- **多参数运算符** (`min`, `max`, `cat`): `{operator: value}`

### 3.5 插件遍历语义与确定性分析

#### 3.5.1 Pluggy get_plugins() 的底层实现
根据 Pluggy 官方 API 文档和源码：
```python
def get_plugins(self):
    """Return a set of all registered plugin objects."""
    return set(self._plugin2hookcallers)
```

**关键事实**：
- 返回类型：**`set`**，不是 `list`
- Python `set` 的特性：不保证元素顺序，遍历顺序取决于元素的哈希值和内部存储结构
- `_plugin2hookcallers` 是 `dict` 类型，虽然 Python 3.7+ 的 dict 保持插入顺序，但转换为 set 后顺序丢失
- **"按注册顺序执行" 是错误表述**，实际遍历顺序是不确定的

#### 3.5.2 数据合并机制（调用方负责）
每个插件的 `add_data()` 返回的字典由调用方通过 `EXECUTE_DATA.update(new_data)` 合并：
- 如果不同插件返回**同名字段**，后遍历到的插件数据会覆盖先遍历到的
- 由于遍历顺序不确定，同名字段的覆盖结果也是**不确定**的
- 插件本身无法感知或控制这种覆盖行为

#### 3.5.3 当前插件的数据依赖分析
检查现有三个插件的 `add_data()` 实现，它们之间**没有数据依赖**，输入全部来自函数参数：

| 插件 | 输入来源 | 输出字段 | 是否依赖其他插件 |
|-----|---------|---------|----------------|
| `default_plugin` | `ephemeral_data['text']` | `extracted_number`, `page_filtered_text` | 无（仅依赖输入参数） |
| `levenshtein_plugin` | `ephemeral_data['text']`, `application_datastruct` 中的 watch history | `levenshtein_ratio`, `levenshtein_similarity`, `levenshtein_distance` | 无（仅依赖输入参数） |
| `wordcount_plugin` | `ephemeral_data['text']` | `word_count` | 无（仅依赖输入参数） |

由于所有插件的输入都来自函数参数（`ephemeral_data`, `application_datastruct`）而非其他插件的输出，且输出字段互不冲突，**当前插件遍历顺序的不确定性不会影响条件判定结果**。

#### 3.5.4 对条件判定稳定性的影响

| 影响方面 | 当前状态 | 说明 |
|---------|---------|------|
| 条件判定结果 | ✅ 稳定 | 字段名无冲突，插件无数据依赖，set 遍历顺序不影响最终结果 |
| UI 下拉框选项顺序 | ❌ **不稳定** | 每次启动都可能变化，取决于 set 遍历顺序 |
| 同名字段覆盖 | ⚠️ 理论风险 | 若不同插件返回同名字段，覆盖结果不确定，但当前无此情况 |
| 插件执行日志顺序 | ❌ 不稳定 | 每次运行日志中插件出现的顺序可能不同 |

> **架构事实**：由于插件间无法直接通信，"插件依赖其他插件注入的数据"这种模式在当前架构下**不可能实现**。任何数据依赖必须通过 `application_datastruct` 或 `ephemeral_data` 传递。

#### 3.5.5 遍历顺序的可见影响
虽然条件判定结果稳定，但遍历顺序在以下场景可见且不稳定：
1. **UI 表单选项顺序**：字段和运算符下拉框的选项顺序每次启动可能不同
2. **`EXECUTE_DATA` 调试输出**：`verify-condition-single-rule` 接口返回的数据字段顺序可能变化
3. **日志输出**：插件执行日志的顺序每次运行可能不同

## 4. 与抓取流程及通知渲染的衔接

### 4.1 在抓取流水线中的位置
条件评估发生在 `text_json_diff/processor.py` 的 `run_changedetection()` 方法中，处于**内容处理完成后、变更检测前**的位置：

```
内容获取 → 预处理（RSS/PDF/JSON） → 过滤（CSS/XPath/JSON） → 文本提取
→ 文本转换（修剪、去重、排序） → 忽略文本处理 → 提取过滤
→ 【条件评估】 → 变更检测（MD5 比较） → 通知触发
```

具体在 `RuleEngine.evaluate_conditions()` (`processor.py:248-263`) 中调用：

```python
# processor.py:612-614
if rule_engine.evaluate_conditions(watch, self.datastore, stripped_text):
    blocked = True
```

调用时传入的 `ephemeral_data` 为 `{'text': stripped_text}`，其中 `stripped_text` 是经过所有过滤和转换后的最终文本。

### 4.2 与其他阻塞规则的关系
条件评估是三大阻塞规则之一，执行顺序为：

1. **trigger_text** (`evaluate_trigger_text()`): 内容必须包含触发文本，否则阻塞
2. **text_should_not_be_present** (`evaluate_text_should_not_be_present()`): 内容不能包含禁止文本，否则阻塞
3. **conditions** (`evaluate_conditions()`): 自定义条件不满足则阻塞

**任意规则返回 `True`（表示阻塞）都会导致 `changed_detected = False`**，即使内容实际发生了变化。

### 4.3 对通知渲染的影响
条件评估失败不会直接影响通知渲染，而是通过阻止变更检测来间接影响：
- 如果条件不满足 → `changed_detected = False` → 不触发通知
- 如果条件满足 → 正常进行变更检测 → 检测到变化则触发通知
- 通知渲染使用的是标准通知模板，不直接访问条件插件数据

### 4.4 预览验证接口
`conditions/blueprint.py` 提供了 `verify-condition-single-rule` 端点，允许用户在编辑页面实时验证单条规则：

- 接收表单数据和规则 JSON
- 使用 `prepare_filter_prevew` 应用当前表单的过滤设置
- 创建临时 watch 对象执行条件评估
- 返回评估结果和 `EXECUTE_DATA` 供调试

## 5. 插件失败对整条流水线的影响

### 5.1 异常处理策略
插件执行采用**容错设计**，单个插件失败不会导致整个条件评估失败：

```python
# __init__.py:125-129
except Exception as e:
    import logging
    logging.error(f"Error executing plugin {plugin.__class__.__name__}: {str(e)}")
    continue
```

### 5.2 超时处理
每个插件的 `add_data()` 调用有 **10 秒超时限制**（`__init__.py:117-124`）：
- 超时会抛出 `concurrent.futures.TimeoutError`
- 被捕获后记录错误，继续执行下一个插件
- 该插件的数据不会被注入到 `EXECUTE_DATA` 中

### 5.3 失败场景分析

| 失败场景 | 对条件评估的影响 | 对流水线的影响 |
|---------|-----------------|---------------|
| 插件 `add_data()` 抛出异常 | 该插件数据不注入，其他插件继续执行 | 条件评估继续，可能因缺少数据导致规则误判 |
| 插件执行超时 | 同上 | 同上 |
| 插件返回非字典数据 | 数据被忽略 | 同上 |
| `convert_to_jsonlogic()` 遇到不完整规则 | 抛出 `EmptyConditionRuleRowNotUsable` | 未被捕获，会导致整个条件评估失败 |
| `jsonLogic` 执行异常 | 未被捕获，会抛出异常 | 导致整个抓取任务失败 |

### 5.4 关键风险点
1. **静默失败**: 插件异常只记录日志不报错，可能导致规则因缺少数据而意外通过或不通过
2. **缺少回退机制**: 如果插件提供的关键字段缺失，依赖该字段的规则会使用 `undefined` 进行比较，结果可能不符合预期
3. **遍历顺序不确定性**: `get_plugins()` 返回 set，遍历顺序完全不确定（详见 3.5 节）
4. **同名字段覆盖风险**: 插件返回同名字段时，后遍历到的会覆盖先遍历到的，由于遍历顺序不确定，覆盖结果不可预测
5. **UI 选项顺序不稳定**: 每次启动程序，条件编辑页面的字段和运算符下拉框顺序都可能变化，影响用户体验

### 5.5 边界情况处理
- 无配置条件: `execute_ruleset_against_all_plugins()` 直接返回 `result=True`
- 所有规则行不完整: `filter_complete_rules()` 返回空列表，跳过评估，返回 `result=True`
- 插件返回 `None`: 不会更新 `EXECUTE_DATA`

## 6. 架构总结

### 6.1 设计优点
- **高度可扩展**: 基于 Pluggy 的插件架构，支持内部和外部插件
- **容错性好**: 单个插件失败不影响整体流程
- **实时验证**: 提供预览接口，用户可在配置时验证规则
- **灵活的规则表达**: 基于 JSON Logic，支持复杂逻辑组合
- **插件隔离**: 插件间通过明确的参数接口通信，避免隐式耦合

### 6.2 可改进点
- **遍历顺序确定性**: 调用 `get_plugins()` 后应转换为排序列表，如 `sorted(plugin_manager.get_plugins(), key=lambda p: p.__name__)`，确保跨环境一致性
- **插件失败反馈**: 插件失败时应在 UI 上给出提示，而不是静默失败
- **数据契约**: 应该定义插件返回数据的 schema，确保字段存在性和类型
- **字段冲突检测**: 检测插件返回的同名字段冲突，给出警告或采用显式的命名空间机制
- **单元测试**: 为插件执行添加更全面的异常处理和测试

### 6.3 核心文件索引

| 文件 | 职责 |
|-----|------|
| `conditions/__init__.py` | 核心执行逻辑、规则转换、插件数据聚合 |
| `conditions/pluggy_interface.py` | 插件管理器、钩子定义、插件加载 |
| `conditions/form.py` | 条件表单定义、验证逻辑 |
| `conditions/blueprint.py` | HTTP API 端点（规则验证） |
| `conditions/default_plugin.py` | 内置基础插件 |
| `processors/text_json_diff/processor.py` | 抓取流水线集成点 |
| `forms.py:877-878` | 条件表单字段定义 |
| `templates/_helpers.html:147` | 条件 UI 渲染宏 |
