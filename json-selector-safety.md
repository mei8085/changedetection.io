# JSON Selector 安全边界分析

## 概述

changedetection.io 支持两种 JSON 选择器：
- **JSONPath** (`json:` 前缀) - 使用 `jsonpath_ng.ext` 库
- **jq** (`jq:` / `jqraw:` 前缀) - 使用 `jq` Python 绑定

本文档精确分析执行路径、安全拦截机制、三类结果处理逻辑。所有结论分为三类标注：
- 🧪 **已有测试直接验证**：结论有对应的测试用例覆盖，注明具体测试锚点
- 🔍 **基于代码推导**：结论来自代码逻辑分析，暂无直接测试验证
- ⚠️ **未被现有测试直接覆盖**：测试空白区域，需要特别注意

---

## 一、执行路径与风险拦截位置

### 1.1 主执行流程图（修正版）

**入口函数：** `html_tools.py:523-564` - `extract_json_as_string()`

```
content (输入)
    │
    ▼
判断是否以 { 或 [ 开头?
    ├─ 是 → 尝试直接解析 JSON
    │       ├─ 解析成功 → 调用 _parse_json()
    │       │         ├─ 表达式合法 → ✅ 继续向下执行
    │       │         └─ 表达式非法 → ❌ 抛出异常（终止流程）
    │       └─ 解析失败 → 记录警告，继续向下尝试其他路径
    └─ 否 → 检查是否为 JSONP 格式
              ├─ 是 JSONP → 提取内部 JSON
              │         ├─ 解析成功 → 调用 _parse_json()
              │         │         ├─ 表达式合法 → ✅ 继续向下执行
              │         │         └─ 表达式非法 → ❌ 抛出异常（终止流程）
              │         └─ 解析失败 → 记录警告，继续向下
              └─ 不是 JSONP → 尝试从 HTML <script>/<body> 提取 JSON blob
                            → 调用 extract_json_blob_from_html()
                                  ├─ 找到可解析 JSON → 调用 _parse_json()
                                  │         ├─ 表达式合法 → ✅ 继续向下执行
                                  │         └─ 表达式非法 → ❌ 抛出异常（终止流程）
                                  └─ 未找到可解析 JSON → ❌ 抛出 JSONNotFound 异常
    │
    ▼
【 重要：仅当表达式合法且成功执行到此处 】
判断匹配结果是否为空?
    ├─ 是 → 返回空字符串 ''
    └─ 否 → 返回匹配结果
```

⚠️ **关键修正**：非法表达式不会走到"结果为空判断"分支，而是在 `_parse_json()` 中直接抛出异常并终止流程。

🧪 **已有测试直接验证**：
- JSONP 格式内容能被正确提取：`test_jsonp_json_filter_extraction`
- 能从 HTML 内嵌 `<script>` 标签中提取 JSON：`test_unittest_inline_html_extract` 第 89-108 行
- 乱码输入时抛出 `JSONNotFound`：`test_unittest_inline_html_extract` 第 113-121 行

---

### 1.2 _parse_json 执行路径

**位置：** `html_tools.py:410-438` - `_parse_json()`

```
json_filter (选择器)
    │
    ├───────────────────────┐
    │                       │
    ▼                       ▼
json: 前缀?             jq:/jqraw: 前缀?
    │                       │
    ▼                       ▼
jsonpath_ng.ext.parse()  ┌─────────────────────────┐
    │                    │ 1. 提取表达式            │
    ▼                    │ 2. validate_jq_expression(expr)  ← 安全拦截点
    │                    │ 3. jq.compile(expr)             ← 实际编译执行
.find(json_data)         │ 4. .input(json_data).all()      ← 执行匹配
    │                    └─────────────────────────────────┘
    ▼
_get_stripped_text_from_json_match(match)
    │
    ▼
返回结果（或空字符串）
```

🧪 **已有测试直接验证**：
- jq 危险表达式会被拦截：`test_blocked_builtins_raise`

🔍 **基于代码推导**：
- jqraw 前缀的选择器也经过 `validate_jq_expression()` 安全检查（与 jq: 相同路径）
- JSONPath 直接编译执行，无安全检查

---

### 1.3 jq 风险拦截的精确位置

**拦截函数：** `html_tools.py:39-54` - `validate_jq_expression()`

```python
def validate_jq_expression(expression: str) -> None:
    # 检查绕过开关 - 最先执行！
    if strtobool(os.getenv('JQ_ALLOW_RISKY_EXPRESSIONS', 'false')):
        return  # 直接返回，跳过所有检查
    
    # 正则表达式黑名单检查
    for pattern, description in _JQ_BLOCKED_PATTERNS:
        if pattern.search(expression):
            logger.critical(f"Security: blocked jq expression containing '{description}'")
            raise ValueError(f"jq expression uses disallowed builtin: {description}")
```

**执行顺序：**
1. 检查环境变量 `JQ_ALLOW_RISKY_EXPRESSIONS`
2. 遍历 12 个正则表达式黑名单模式
3. 如果匹配到危险模式 → 记录 CRITICAL 日志 + 抛出 `ValueError`
4. 检查通过 → 返回，继续执行 `jq.compile()`

🧪 **已有测试直接验证**：
- 危险表达式抛出 `ValueError`：`test_blocked_builtins_raise`，测试文件第 15-44 行
- 安全表达式不抛出异常：`test_safe_expressions_pass`，测试文件第 46-65 行
- `JQ_ALLOW_RISKY_EXPRESSIONS=true` 时绕过检查：`test_allow_risky_env_var_bypasses_check`，测试文件第 67-79 行
- 默认情况下启用检查：`test_allow_risky_env_var_off_by_default`，测试文件第 81-90 行

---

### 1.4 绕过开关的影响范围

| 开关状态 | 影响 |
|---------|------|
| `JQ_ALLOW_RISKY_EXPRESSIONS=false` (默认) | 启用所有 12 个安全检查，危险表达式被拦截 |
| `JQ_ALLOW_RISKY_EXPRESSIONS=true` | **完全禁用所有安全检查**，任何 jq 表达式都能执行 |

🧪 **已有测试直接验证**：
- 绕过开关能完全禁用检查：`test_allow_risky_env_var_bypasses_check`，测试文件第 67-79 行
- 默认状态下检查生效：`test_allow_risky_env_var_off_by_default`，测试文件第 81-90 行

---

## 二、三类结果的精确区分

### 2.1 第一类：合法表达式，无匹配结果

**触发场景：**
- 选择器语法正确，但 JSON 中没有匹配的内容
- 例如：`json:$.does_not_exist` 应用于 `{"id": 5}`

**处理逻辑：** `html_tools.py:452-454` - `_get_stripped_text_from_json_match()`

```python
if not match:
    # Re 265 - Just return an empty string when filter not found
    return ''
```

**最终返回值：** 空字符串 `''`

🔍 **基于代码推导**：
- Re #265 注释说明：允许选择器不匹配任何内容，因为用户可能在等待某个条件出现
- **无直接测试断言**：集成测试间接验证此行为，但没有专门针对"合法无匹配"的测试用例

---

### 2.2 第二类：表达式非法（语法错误或安全拦截）

**触发场景：**
1. **JSONPath 语法错误**：例如 `json:$.[invalid`
   - 抛出：`jsonpath_ng.parser.JsonPathParserError`
   
2. **jq 语法错误**：例如 `jq:.foo | bar | baz(`
   - 抛出：`jq._jq.InvalidExpression`
   
3. **jq 安全拦截**：表达式包含危险内置函数
   - 抛出：`ValueError`

**异常处理：** `html_tools.py:533-558` - `extract_json_as_string()`

```python
try:
    stripped_text_from_html = _parse_json(json.loads(content.lstrip("\ufeff")), json_filter)
except json.JSONDecodeError as e:
    logger.warning(f"Error processing JSON {content[:20]}...{str(e)})")
# 重要：只有 JSONDecodeError 被捕获！
# JsonPathParserError、ValueError、InvalidExpression 都不会被捕获！
```

**关键发现：**
- ✅ `json.JSONDecodeError` 被捕获并记录警告
- ❌ **JSONPath 语法错误**：异常直接向上抛出，不被捕获
- ❌ **jq 语法错误**：异常直接向上抛出，不被捕获  
- ❌ **jq 安全拦截**：`ValueError` 直接向上抛出，不被捕获

**最终行为：** 异常向上抛出，不由本函数处理

⚠️ **未被现有测试直接覆盖**：
- 异常传播路径（只有 JSONDecodeError 被捕获）没有对应的测试用例
- 现有测试仅验证了 `validate_jq_expression()` 会抛出异常（单元测试），但**未验证**在 `extract_json_as_string()` 完整调用链中异常是否会被捕获
- 没有测试覆盖 JSONPath 语法错误的传播路径

🧪 **已有测试直接验证**（仅单元测试层面）：
- `validate_jq_expression()` 对危险表达式抛出 `ValueError`：`test_blocked_builtins_raise`

---

### 2.3 第三类：文档中无可解析的 JSON

**触发场景：**
- 输入内容既不是纯 JSON，也不是 JSONP，也无法从 HTML 的 `<script>` 或 `<body>` 中提取到可解析的 JSON
- 例如：输入为完全乱码 `'COMPLETE GIBBERISH, NO JSON!'`

**抛出位置：** `html_tools.py:490-491` - `extract_json_blob_from_html()`

```python
if not bs_jsons:
    raise JSONNotFound("No parsable JSON found in this document")
```

**抛出异常：** `JSONNotFound`（继承自 `ValueError`）

**关键条件：**
- 仅在通过 `extract_json_blob_from_html()` 路径时才可能抛出
- 纯 JSON 或 JSONP 解析失败时，仅记录警告不抛出此异常

🧪 **已有测试直接验证**：
- 乱码输入时，`json:`、`jq:`、`jqraw:` 三种选择器都抛出 `JSONNotFound`：`test_unittest_inline_html_extract` 第 113-121 行

🔍 **基于代码推导**：
- 纯 JSON 解析失败时，仅记录警告，不抛出 `JSONNotFound`

---

### 2.4 三类结果总结对照表

| 情况 | 触发条件 | 处理方式 | 代码位置 | 证据类型 |
|------|---------|---------|---------|---------|
| **合法无匹配** | 表达式语法正确，但 JSON 中没有匹配内容 | 返回空字符串 `''` | `_get_stripped_text_from_json_match()`:452-454 | 🔍 基于代码推导（Re #265 注释） |
| **表达式非法** | JSONPath 语法错误、jq 语法错误、jq 危险表达式 | 抛出对应异常（**不被捕获，直接向上传播**） | `_parse_json()` 内部 | ⚠️ 异常传播路径未被测试直接覆盖 |
| **无可解析 JSON** | 既不是纯 JSON，也不是 JSONP，也无法从 HTML 提取有效 JSON | 抛出 `JSONNotFound` | `extract_json_blob_from_html()`:490-491 | 🧪 已有测试直接验证 |

---

## 三、jq 危险表达式拦截详情

### 3.1 12 个被阻止的内置函数/变量

**位置：** `html_tools.py:24-37` - `_JQ_BLOCKED_PATTERNS`

| 序号 | 正则模式 | 匹配内容 | 风险类型 | 证据类型 |
|------|---------|---------|---------|---------|
| 1 | `\benv\b` | `env` 函数 | 环境变量泄露 | 🧪 已有测试直接验证 |
| 2 | `\$ENV\b` | `$ENV` 变量 | 环境变量泄露 | 🧪 已有测试直接验证 |
| 3 | `\binclude\b` | `include` 指令 | 文件读取 | 🧪 已有测试直接验证 |
| 4 | `\bimport\b` | `import` 指令 | 文件读取 | 🧪 已有测试直接验证 |
| 5 | `\binputs?\b` | `input` / `inputs` | 读取 stdin 外部数据 | 🧪 已有测试直接验证 |
| 6 | `\bdebug\b` | `debug` 函数 | 数据泄露到 stderr | 🧪 已有测试直接验证 |
| 7 | `\bstderr\b` | `stderr` 函数 | 数据泄露到 stderr | 🧪 已有测试直接验证 |
| 8 | `\bhalt(?:_error)?\b` | `halt` / `halt_error` | 进程终止（DoS） | 🧪 已有测试直接验证 |
| 9 | `\$__loc__\b` | `$__loc__` 变量 | 文件路径信息泄露 | 🧪 已有测试直接验证 |
| 10 | `\bbuiltins\b` | `builtins` 函数 | 枚举可用函数 | 🧪 已有测试直接验证 |
| 11 | `\bmodulemeta\b` | `modulemeta` 函数 | 模块信息泄露 | 🧪 已有测试直接验证 |
| 12 | `\$JQ_BUILD_CONFIGURATION\b` | `$JQ_BUILD_CONFIGURATION` | 构建配置信息泄露 | 🧪 已有测试直接验证 |

---

### 3.2 测试验证的危险表达式示例

**位置：** `tests/unit/test_jq_security.py:15-40`

```python
blocked = [
    'env',                    # 直接调用
    '.foo | env',             # 在管道中
    '$ENV',                   # 变量访问
    '$ENV.SECRET',            # 变量属性访问
    'include "foo"',          # 文件包含
    'import "foo" as f',      # 文件导入
    'input',                  # 读取 stdin
    'inputs',                 # 读取所有输入
    '[.,inputs]',             # 在数组中使用
    'halt',                   # 终止进程
    'halt_error(1)',          # 带错误码终止
    'debug',                  # debug 输出
    '. | debug | .foo',       # 在管道中使用 debug
    'stderr',                 # stderr 输出
    '$__loc__',               # 位置信息
    'builtins',               # 枚举内置函数
    'modulemeta',             # 模块元数据
    '$JQ_BUILD_CONFIGURATION',# 构建配置
]
```

🧪 **已有测试直接验证**：以上所有表达式调用 `validate_jq_expression()` 时必须抛出 `ValueError`，测试文件第 15-44 行

---

### 3.3 测试验证的安全表达式示例

**位置：** `tests/unit/test_jq_security.py:50-59`

```python
safe = [
    '.foo',                                   # 简单字段访问
    '.items[] | .price',                      # 管道操作
    'map(select(.active)) | length',          # 映射+过滤+计数
    '.[] | select(.name | test("foo"))',      # 过滤+正则匹配
    'to_entries | map(.value) | add',         # 转换+映射+求和
    '[.[] | .id] | unique',                   # 数组提取+去重
    '.price | tonumber',                      # 类型转换
    'if .stock > 0 then "in stock" else "out of stock" end',  # 条件表达式
]
```

🧪 **已有测试直接验证**：以上所有表达式调用 `validate_jq_expression()` 时不得抛出异常，测试文件第 46-65 行

---

## 四、JSONPath 与 jq 安全边界对比

### 4.1 JSONPath (`json:`)

**使用库：** `jsonpath_ng.ext`

**安全特性：**
- 🔍 **基于代码推导**：无文件系统访问能力
- 🔍 **基于代码推导**：无环境变量访问能力
- 🔍 **基于代码推导**：无网络访问能力
- 🔍 **基于代码推导**：仅在提供的 JSON 数据上操作
- 🔍 **基于代码推导**：无进程控制能力

**风险等级：** ✅ **低风险**

**安全机制：** 无额外安全检查（基于库特性推导）

---

### 4.2 jq (`jq:` / `jqraw:`)

**使用库：** `jq` Python 绑定

**安全特性：**
- 🧪 **已有测试直接验证**：黑名单式安全检查（12 个正则模式阻止危险内置函数）
- 🔍 **基于代码推导**：检查在编译前执行（`validate_jq_expression()` 在 `jq.compile()` 之前调用）
- 🔍 **基于代码推导**：拦截时记录 CRITICAL 级别日志
- 🧪 **已有测试直接验证**：可通过环境变量 `JQ_ALLOW_RISKY_EXPRESSIONS=true` 完全绕过
- 🔍 **基于代码推导**：基于正则的黑名单存在局限性（可能存在绕过方式）

**已缓解的风险（有测试验证）：**
- 环境变量泄露
- 任意文件读取
- 进程终止（DoS）
- stderr 数据泄露
- 内部信息泄露

**潜在未缓解的风险（基于代码推导）：**
- CPU 资源耗尽（复杂递归表达式导致 DoS）
- 内存资源耗尽
- 正则黑名单绕过

**风险等级：** ⚠️ **中高风险（有保护）**

---

## 五、证据边界总结表（含精确测试锚点）

| 项目 | 结论 | 证据类型 | 精确测试锚点 |
|-----|------|---------|-------------|
| validate_jq_expression 会拦截危险表达式 | 12 种危险 jq 表达式抛出 `ValueError` | 🧪 已有测试直接验证 | `test_blocked_builtins_raise`，`test_jq_security.py` 第 11-44 行 |
| 安全表达式通过检查 | 8 种安全表达式不抛出异常 | 🧪 已有测试直接验证 | `test_safe_expressions_pass`，`test_jq_security.py` 第 46-65 行 |
| 绕过开关 | `JQ_ALLOW_RISKY_EXPRESSIONS=true` 完全禁用所有检查 | 🧪 已有测试直接验证 | `test_allow_risky_env_var_bypasses_check`，`test_jq_security.py` 第 67-79 行 |
| 默认状态 | 未设置环境变量时检查生效 | 🧪 已有测试直接验证 | `test_allow_risky_env_var_off_by_default`，`test_jq_security.py` 第 81-90 行 |
| 无可解析 JSON | 乱码输入抛出 `JSONNotFound` | 🧪 已有测试直接验证 | `test_unittest_inline_html_extract`，`test_jsonpath_jq_selector.py` 第 113-121 行 |
| JSONP 提取 | 能从 JSONP 格式中提取 JSON 并过滤 | 🧪 已有测试直接验证 | `test_jsonp_json_filter_extraction`，`test_jsonpath_jq_selector.py` 第 42-61 行 |
| 合法无匹配 | 返回空字符串 `''` | 🔍 基于代码推导 | 只有代码注释 Re #265，无对应测试用例 |
| 异常传播路径 | 只有 `JSONDecodeError` 被捕获，其他异常直接向上抛出 | ⚠️ 未被现有测试直接覆盖 | 无测试用例验证完整调用链中的异常传播 |
| jqraw 安全检查 | jqraw 前缀也经过 `validate_jq_expression()` | 🔍 基于代码推导 | 代码路径与 jq: 相同，但无专门测试断言 |
| 纯 JSON 解析失败 | 仅记录警告，不抛出 `JSONNotFound` | 🔍 基于代码推导 | 只有代码 try-catch 逻辑，无专门测试用例 |
| JSONPath 安全性 | 无文件/环境/网络/进程访问能力 | 🔍 基于代码推导 | 基于 jsonpath_ng.ext 库特性，无专门测试 |
| 拦截日志记录 | 拦截时记录 CRITICAL 级别日志 | 🔍 基于代码推导 | 代码中有 logger.critical 调用，但无测试验证日志输出 |

---

## 六、安全建议

1. **优先使用 JSONPath**：对于简单的字段提取需求，优先使用 `json:` 选择器，安全性更高

2. **仅在必要时使用 jq**：jq 功能强大但攻击面更大，仅在需要复杂数据处理时使用

3. **不要启用绕过开关**：除非在完全隔离的环境中且完全理解风险，否则不要设置 `JQ_ALLOW_RISKY_EXPRESSIONS=true`

4. **补充异常传播测试**：建议补充测试用例，验证非法表达式在 `extract_json_as_string()` 完整调用链中的行为

5. **考虑执行超时限制**：为 jq 表达式执行添加超时限制，防止 DoS 攻击

6. **监控 jq 表达式审计**：对多用户环境中的 jq 表达式使用进行审计和告警

---

## 七、相关代码位置速查

| 文件名 | 行号 | 说明 |
|-------|------|------|
| `changedetectionio/html_tools.py` | 24-37 | `_JQ_BLOCKED_PATTERNS` 12 个危险模式定义 |
| `changedetectionio/html_tools.py` | 39-54 | `validate_jq_expression()` jq 安全验证函数 |
| `changedetectionio/html_tools.py` | 62-64 | `JSONNotFound` 异常类 |
| `changedetectionio/html_tools.py` | 410-438 | `_parse_json()` 选择器执行核心 |
| `changedetectionio/html_tools.py` | 440-459 | `_get_stripped_text_from_json_match()` 匹配结果处理 |
| `changedetectionio/html_tools.py` | 461-518 | `extract_json_blob_from_html()` HTML 中提取 JSON |
| `changedetectionio/html_tools.py` | 523-564 | `extract_json_as_string()` 主入口函数 |
| `changedetectionio/tests/unit/test_jq_security.py` | 1-94 | jq 安全单元测试（4 个测试用例） |
| `changedetectionio/tests/test_jsonpath_jq_selector.py` | 1-526 | JSONPath/jq 集成测试 |

---

*文档生成时间：2026-05-17*
