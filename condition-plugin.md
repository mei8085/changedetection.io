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
   - 显式注册，顺序固定为第一个

2. **目录扫描加载** (`pluggy_interface.py:52-71`)：
   - 扫描 `changedetectionio/conditions/plugins/` 目录
   - 使用 `os.listdir()` 遍历文件，**其返回顺序由文件系统决定，不保证排序**（非字母序、非时间序）
   - 加载所有 `.py` 文件（排除 `__init__.py`），按遍历顺序动态导入并注册
   - ⚠️ **关键注意**：`os.listdir()` 顺序在不同操作系统、不同文件系统、甚至同一系统文件增删后都可能变化

3. **外部包发现** (`pluggy_interface.py:74`)：
   ```python
   plugin_manager.load_setuptools_entrypoints(PLUGIN_NAMESPACE)
   ```
   - 支持通过 setuptools entry points 安装的外部插件
   - 顺序由 setuptools 的发现机制决定

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

由于 `operator_choices` 和 `field_choices` 使用 `extend()` 按插件注册顺序追加，**UI 下拉框中的选项顺序会随插件加载顺序变化**。

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
1. 初始化 EXECUTE_DATA 字典
2. 获取 watch 对象和 conditions 配置
3. 确定逻辑运算符（ALL → and, ANY → or）
4. 过滤掉不完整的规则行
5. 插件数据注入阶段（按注册顺序）：
   ├─ 为每个插件创建线程池执行器
   ├─ 调用 plugin.add_data()，超时 10 秒
   ├─ 合并返回的数据到 EXECUTE_DATA
   └─ 插件异常被捕获并记录，但不中断流程
6. 将规则转换为 JSON Logic 格式
7. 使用 jsonLogic 评估规则
8. 返回评估结果和执行时数据
```

### 3.3 规则转换逻辑
`convert_to_jsonlogic()` (`__init__.py:41-79`) 将结构化规则转换为 JSON Logic 格式：

- **标准二元运算符** (`>`, `<`, `==` 等): `{operator: [{"var": field}, value]}`
- **in 运算符**: `{"in": [value, {"var": field}]}`（注意参数顺序反转）
- **一元运算符** (`!`, `!!`, `-`): `{operator: [{"var": field}]}`
- **多参数运算符** (`min`, `max`, `cat`): `{operator: value}`

### 3.4 插件执行顺序与确定性分析

#### 3.4.1 实际执行顺序
插件执行顺序由注册顺序决定，而注册顺序受以下因素影响：

| 阶段 | 顺序确定性 | 说明 |
|-----|-----------|------|
| 1. `default_plugin` | ✅ 完全确定 | 显式调用 `register()`，始终为第一个 |
| 2. 目录扫描插件 | ❌ **不确定** | 由 `os.listdir()` 返回顺序决定，取决于文件系统 |
| 3. 外部包插件 | ⚠️ 半确定 | 由 setuptools entry points 发现顺序决定 |

> **重要修正**：目录插件**不是按文件名排序**加载的。`os.listdir()` 的返回顺序由底层文件系统的目录项存储顺序决定，跨平台、跨环境、甚至同一环境文件增删后都可能发生变化。

#### 3.4.2 数据合并机制
每个插件的 `add_data()` 返回的字典通过 `EXECUTE_DATA.update(new_data)` 合并到全局上下文：
- 如果不同插件返回**同名字段**，后执行的插件会覆盖先执行的
- 后续插件可以通过 `EXECUTE_DATA` 访问前面插件注入的数据（但当前插件实现中未使用此能力）

#### 3.4.3 当前插件的数据依赖分析
检查现有三个插件的 `add_data()` 实现，它们之间**没有数据依赖**：

| 插件 | 输入来源 | 输出字段 | 是否依赖其他插件 |
|-----|---------|---------|----------------|
| `default_plugin` | `ephemeral_data['text']` | `extracted_number`, `page_filtered_text` | 无（仅依赖输入参数） |
| `levenshtein_plugin` | `ephemeral_data['text']`, watch history | `levenshtein_ratio`, `levenshtein_similarity`, `levenshtein_distance` | 无（仅依赖输入参数） |
| `wordcount_plugin` | `ephemeral_data['text']` | `word_count` | 无（仅依赖输入参数） |

由于所有插件的输入都来自函数参数（`ephemeral_data`, `application_datastruct`）而非 `EXECUTE_DATA`，且输出字段互不冲突，**当前执行顺序的不确定性不会影响条件判定结果**。

#### 3.4.4 对条件判定稳定性的影响

| 影响方面 | 当前状态 | 潜在风险 |
|---------|---------|---------|
| 条件判定结果 | ✅ 稳定 | 字段名无冲突，无数据依赖 |
| UI 下拉框选项顺序 | ❌ 可能变化 | `operator_choices` 和 `field_choices` 按注册顺序 `extend`，顺序变化会影响用户体验 |
| 未来插件兼容性 | ⚠️ 需注意 | 如果新增插件依赖其他插件注入的数据，或出现同名字段覆盖，顺序不确定性将导致不稳定 |

#### 3.4.5 执行顺序的可见影响
虽然条件判定结果稳定，但执行顺序在以下场景可见：
1. **UI 表单选项顺序**：字段和运算符下拉框的选项顺序随插件注册顺序变化
2. **`EXECUTE_DATA` 调试输出**：`verify-condition-single-rule` 接口返回的数据字段顺序可能变化
3. **日志输出**：插件执行日志的顺序可能变化

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
- 返回评估结果和执行时数据供调试

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
3. **执行顺序不确定性**: 目录插件加载顺序依赖 `os.listdir()`，存在固有的不确定性（详见 3.4 节）
4. **同名字段覆盖风险**: 插件返回同名字段时，后执行的会覆盖先执行的，而执行顺序不确定可能导致覆盖结果不可预测
5. **潜在连锁反应**: 如果未来新增插件依赖前面插件注入的数据，前面插件失败会导致连锁反应

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

### 6.2 可改进点
- **执行顺序确定性**: 目录插件加载应使用 `sorted(os.listdir())` 确保跨环境一致性，或支持插件显式声明优先级
- **插件失败反馈**: 插件失败时应在 UI 上给出提示，而不是静默失败
- **数据契约**: 应该定义插件返回数据的 schema，确保字段存在性和类型
- **依赖管理**: 支持插件间显式声明依赖关系，确保执行顺序满足依赖要求
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
