# JSON Selector 安全边界分析

## 概述

changedetection.io 支持两种 JSON 选择器：
- **JSONPath** (`json:` 前缀) - 使用 `jsonpath_ng.ext` 库
- **jq** (`jq:` / `jqraw:` 前缀) - 使用 `jq` Python 绑定

本文档详细分析这两种选择器的执行路径、安全拦截点、三类结果的区别，以及安全边界。所有结论与现有测试用例一一对应。

---

## 一、执行路径与风险拦截位置

### 1.1 主执行流程

**入口函数：** `html_tools.py:523-564` - `extract_json_as_string()`

```
content (输入)
    │
    ▼
判断是否以 { 或 [ 开头?
    ├─ 是 → 尝试直接解析 JSON → 调用 _parse_json()
    └─ 否 → 检查是否为 JSONP 格式
              ├─ 是 JSONP → 提取内部 JSON → 调用 _parse_json()
              └─ 不是 JSONP → 尝试从 HTML <script>/<body> 提取 JSON blob
                            → 调用 extract_json_blob_from_html()
                                  │
                                  ▼
                            遍历所有 <script> 和 <body>
                                  │
                                  ▼
                            找到可解析的 JSON 了吗?
                                  ├─ 是 → 调用 _parse_json()
                                  └─ 否 → 抛出 JSONNotFound 异常
    │
    ▼
结果为空吗? → 是 → 返回空字符串 ''
              └─ 否 → 返回匹配结果
```

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
    ▼                    │ 2. validate_jq_expression(expr)  ← 安全拦截点！
.find(json_data)         │ 3. jq.compile(expr)             ← 实际编译执行
    │                    │ 4. .input(json_data).all()      ← 执行匹配
    ▼                    └─────────────────────────────────┘
_get_stripped_text_from_json_match(match)
    │
    ▼
返回结果
```

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
1. ✅ 检查环境变量 `JQ_ALLOW_RISKY_EXPRESSIONS`
2. ✅ 遍历 12 个正则表达式黑名单模式
3. ✅ 如果匹配到危险模式 → 记录日志 + 抛出 `ValueError`
4. ✅ 检查通过 → 返回，继续执行 `jq.compile()`

### 1.4 绕过开关的影响范围

| 开关状态 | 影响 |
|---------|------|
| `JQ_ALLOW_RISKY_EXPRESSIONS=false` (默认) | ✅ 启用所有 12 个安全检查，危险表达式被拦截 |
| `JQ_ALLOW_RISKY_EXPRESSIONS=true` | ❌ **完全禁用所有安全检查**，任何 jq 表达式都能执行 |

**注意：** 绕过开关影响的是 `validate_jq_expression()` 函数本身，不是 `jq.compile()`。开关打开时，函数直接 `return`，不执行任何检查逻辑。

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

**对应测试用例：**
- ✅ Re #265 注释说明：允许选择器不匹配任何内容，因为用户可能在等待某个条件出现

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
# 注意：其他异常 (JsonPathParserError, ValueError, InvalidExpression) 不会被捕获！
```

**关键发现：**
- ⚠️ 只有 `json.JSONDecodeError` 被捕获并记录警告
- ⚠️ 其他异常（包括 JSONPath 语法错误、jq 语法错误、jq 安全拦截）**直接向上抛出**，不会被捕获！
- ⚠️ 只有异常未被捕获且执行到最后，才返回空字符串

**最终行为：** 异常向上抛出，不由本函数处理

**对应测试用例：**
- ✅ `test_blocked_builtins_raise`：12 种危险 jq 表达式必须抛出 `ValueError`
- ✅ `test_safe_expressions_pass`：安全表达式不抛出异常

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

**对应测试用例：**
- ✅ `test_unittest_inline_html_extract`（第 113-121 行）：对乱码输入，`json:`、`jq:`、`jqraw:` 三种选择器都必须抛出 `JSONNotFound`

---

### 2.4 三类结果总结对照表

| 情况 | 触发条件 | 返回/抛出 | 代码位置 | 对应测试 |
|------|---------|-----------|---------|----------|
| **合法无匹配** | 表达式语法正确，但 JSON 中没有匹配内容 | 返回空字符串 `''` | `_get_stripped_text_from_json_match()`:452-454 | Re #265 注释 |
| **表达式非法** | JSONPath 语法错误、jq 语法错误、jq 危险表达式 | 抛出对应异常（不被捕获） | `_parse_json()` 内部 | `test_blocked_builtins_raise` |
| **无可解析 JSON** | 既不是纯 JSON，也不是 JSONP，也无法从 HTML 提取有效 JSON | 抛出 `JSONNotFound` | `extract_json_blob_from_html()`:490-491 | `test_unittest_inline_html_extract` 第 113-121 行 |

---

## 三、jq 危险表达式拦截详情

### 3.1 12 个被阻止的内置函数/变量

**位置：** `html_tools.py:24-37` - `_JQ_BLOCKED_PATTERNS`

| 序号 | 正则模式 | 匹配内容 | 风险类型 | 对应测试用例 |
|------|---------|---------|---------|-------------|
| 1 | `\benv\b` | `env` 函数 | 环境变量泄露 | `test_blocked_builtins_raise` |
| 2 | `\$ENV\b` | `$ENV` 变量 | 环境变量泄露 | `test_blocked_builtins_raise` |
| 3 | `\binclude\b` | `include` 指令 | 文件读取 | `test_blocked_builtins_raise` |
| 4 | `\bimport\b` | `import` 指令 | 文件读取 | `test_blocked_builtins_raise` |
| 5 | `\binputs?\b` | `input` / `inputs` | 读取 stdin 外部数据 | `test_blocked_builtins_raise` |
| 6 | `\bdebug\b` | `debug` 函数 | 数据泄露到 stderr | `test_blocked_builtins_raise` |
| 7 | `\bstderr\b` | `stderr` 函数 | 数据泄露到 stderr | `test_blocked_builtins_raise` |
| 8 | `\bhalt(?:_error)?\b` | `halt` / `halt_error` | 进程终止（DoS） | `test_blocked_builtins_raise` |
| 9 | `\$__loc__\b` | `$__loc__` 变量 | 文件路径信息泄露 | `test_blocked_builtins_raise` |
| 10 | `\bbuiltins\b` | `builtins` 函数 | 枚举可用函数 | `test_blocked_builtins_raise` |
| 11 | `\bmodulemeta\b` | `modulemeta` 函数 | 模块信息泄露 | `test_blocked_builtins_raise` |
| 12 | `\$JQ_BUILD_CONFIGURATION\b` | `$JQ_BUILD_CONFIGURATION` | 构建配置信息泄露 | `test_blocked_builtins_raise` |

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

✅ **测试验证：** 以上所有表达式调用 `validate_jq_expression()` 时必须抛出 `ValueError`

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

✅ **测试验证：** 以上所有表达式调用 `validate_jq_expression()` 时不得抛出异常

---

### 3.4 绕过开关的测试验证

**对应测试用例：**
1. ✅ `test_allow_risky_env_var_bypasses_check`：设置 `JQ_ALLOW_RISKY_EXPRESSIONS=true` 时，即使是最危险的表达式 `env` 和 `$ENV` 也能通过检查

2. ✅ `test_allow_risky_env_var_off_by_default`：未设置环境变量时（默认），危险表达式必须被拦截

---

## 四、JSONPath 与 jq 安全边界对比

### 4.1 JSONPath (`json:`)

**使用库：** `jsonpath_ng.ext`

**安全特性：**
- ✅ 无文件系统访问能力
- ✅ 无环境变量访问能力
- ✅ 无网络访问能力
- ✅ 仅在提供的 JSON 数据上操作
- ✅ 无进程控制能力

**风险等级：** ✅ **低风险**

**安全机制：** 无额外安全检查（因为库本身就是安全的）

---

### 4.2 jq (`jq:` / `jqraw:`)

**使用库：** `jq` Python 绑定

**安全特性：**
- ✅ **黑名单式安全检查**：12 个正则模式阻止危险内置函数
- ✅ **检查在编译前执行**：`validate_jq_expression()` 在 `jq.compile()` 之前调用
- ✅ **安全日志记录**：拦截时记录 CRITICAL 级别日志
- ⚠️ **可完全绕过**：通过环境变量 `JQ_ALLOW_RISKY_EXPRESSIONS=true`
- ⚠️ **基于正则的黑名单局限性**：可能存在绕过方式（如字符串拼接、编码等）

**已缓解的风险：**
- 环境变量泄露
- 任意文件读取
- 进程终止（DoS）
- stderr 数据泄露
- 内部信息泄露

**潜在未缓解的风险：**
- CPU 资源耗尽（复杂递归表达式导致 DoS）
- 内存资源耗尽
- 正则黑名单绕过

**风险等级：** ⚠️ **中高风险（有保护）**

---

## 五、与现有测试用例的完整对应表

| 测试用例 | 验证内容 | 文档章节 |
|---------|---------|---------|
| `test_blocked_builtins_raise` | 12 种危险 jq 表达式必须抛出 `ValueError` | 3.1, 3.2 |
| `test_safe_expressions_pass` | 8 种安全表达式必须通过检查 | 3.3 |
| `test_allow_risky_env_var_bypasses_check` | `JQ_ALLOW_RISKY_EXPRESSIONS=true` 绕过所有检查 | 1.4, 3.4 |
| `test_allow_risky_env_var_off_by_default` | 默认情况下必须启用安全检查 | 1.4, 3.4 |
| `test_jsonp_json_filter_extraction` | JSONP 格式内容能被正确提取和过滤 | 1.1 |
| `test_unittest_inline_html_extract` 第 89-108 行 | 能从 HTML 内嵌 `<script>` 标签中提取 JSON 并正确匹配 | 1.1 |
| `test_unittest_inline_html_extract` 第 113-121 行 | 找不到可解析 JSON 时必须抛出 `JSONNotFound` 异常 | 2.3 |
| Re #265 注释（第 360 行） | 选择器无匹配时返回空字符串（不报错） | 2.1 |
| `test_jsonpath_BOM_utf8` | 支持带 BOM 的 UTF-8 JSON 输入 | 输入处理 |

---

## 六、安全建议

1. **优先使用 JSONPath**：对于简单的字段提取需求，优先使用 `json:` 选择器，安全性更高

2. **仅在必要时使用 jq**：jq 功能强大但攻击面更大，仅在需要复杂数据处理时使用

3. **不要启用绕过开关**：除非在完全隔离的环境中且完全理解风险，否则不要设置 `JQ_ALLOW_RISKY_EXPRESSIONS=true`

4. **考虑执行超时限制**：为 jq 表达式执行添加超时限制，防止 DoS 攻击

5. **监控 jq 表达式审计**：对多用户环境中的 jq 表达式使用进行审计和告警

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
