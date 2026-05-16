# JSON Selector 安全边界分析

## 概述

changedetection.io 支持两种 JSON 选择器：
- **JSONPath** (`json:` 前缀) - 使用 `jsonpath_ng.ext` 库
- **jq** (`jq:` / `jqraw:` 前缀) - 使用 `jq` Python 绑定

本文档详细分析这两种选择器的执行路径、安全拦截点、空结果与非法表达式的区别，以及安全边界。

---

## 一、执行路径

### 1.1 主入口函数

**位置：`html_tools.py:523-564` - `extract_json_as_string()`

```python
def extract_json_as_string(content, json_filter, ensure_is_ldjson_info_type=None):
    # 1. 检测内容类型检测
    # 2. 尝试直接解析 JSON
    # 3. 如果失败，尝试提取 JSONP 内部内容
    # 4. 如果仍失败，尝试从 HTML <script> 或 <body> 中提取 JSON blob
    # 5. 调用 _parse_json() 执行实际的选择器
```

**执行流程：**
1. **内容类型检测**：检查内容是否以 `{` 或 `[` 开头
2. **直接 JSON 解析
3. **JSONP 包装器检测与提取
4. **HTML 嵌入 JSON 提取（从 `<script>` 或 `<body>`）
5. **调用 `_parse_json()` 执行选择器

### 1.2 JSONPath 执行路径

**位置：** `html_tools.py:410-416` - `_parse_json()`

```python
if json_filter.startswith("json:"):
    from jsonpath_ng.ext import parse
    jsonpath_expression = parse(json_filter.replace('json:', ''))
    match = jsonpath_expression.find(json_data)
    return _get_stripped_text_from_json_match(match)
```

**关键点：
- 使用 `jsonpath_ng.ext` 扩展版本（而非 `jsonpath_ng`
- 无表达式验证 - 直接编译执行
- 解析失败会抛出 `jsonpath_ng.parser.JsonPathParserError`

### 1.3 jq 执行路径

**位置：** `html_tools.py:418-438` - `_parse_json()`

```python
if json_filter.startswith("jq:") or json_filter.startswith("jqraw:"):
    import jq
    expr = json_filter.removeprefix("jq:") 或 "jqraw:"
    validate_jq_expression(expr)  # 安全验证点！
    jq_expression = jq.compile(expr)
    match = jq_expression.input(json_data).all()
    return _get_stripped_text_from_json_match(match) 或原始字符串
```

**关键点：**
- **必须通过 `validate_jq_expression()` 进行安全验证
- 验证在 `jq.compile()` 之前执行
- `jq:` 返回 JSON 格式字符串
- `jqraw:` 返回原始字符串格式

---

## 二、危险表达式拦截机制

### 2.1 jq 安全验证

**位置：** `html_tools.py:39-54` - `validate_jq_expression()`

```python
def validate_jq_expression(expression: str) -> None:
    if strtobool(os.getenv('JQ_ALLOW_RISKY_EXPRESSIONS', 'false')):
        return  # 绕过安全检查
    
    for pattern, description in _JQ_BLOCKED_PATTERNS:
        if pattern.search(expression):
            raise ValueError(msg)
```

### 2.2 被阻止的 jq 内置函数

**位置：** `html_tools.py:24-37` - `_JQ_BLOCKED_PATTERNS`

| 模式 | 描述 | 风险 |
|------|------|------|
| `\benv\b` | `env` 函数 | 读取所有环境变量（密码、API 密钥等） |
| `\$ENV\b` | `$ENV` 变量 | 读取所有环境变量 |
| `\binclude\b` | `include` 指令 | 从磁盘读取任意文件 |
| `\bimport\b` | `import` 指令 | 从磁盘读取任意文件 |
| `\binputs?\b` | `input` / `inputs` | 读取超出提供的 JSON 数据之外的内容 |
| `\bdebug\b` | `debug` 函数 | 将数据泄露到 stderr |
| `\bstderr\b` | `stderr` 函数 | 将数据泄露到 stderr |
| `\bhalt(?:_error)?\b` | `halt` / `halt_error` | 终止进程（DoS 攻击） |
| `\$__loc__\b` | `$__loc__` 变量 | 泄露文件路径信息 |
| `\bbuiltins\b` | `builtins` 函数 | 枚举可用函数 |
| `\bmodulemeta\b` | `modulemeta` 函数 | 泄露模块信息 |
| `\$JQ_BUILD_CONFIGURATION\b` | `$JQ_BUILD_CONFIGURATION` | 泄露构建配置信息 |

### 2.3 绕过机制

可以通过设置环境变量绕过安全检查：

```bash
JQ_ALLOW_RISKY_EXPRESSIONS=true
```

**注意：** 这会禁用所有安全检查，允许执行任意危险表达式。

### 2.4 JSONPath 安全边界

JSONPath 使用 `jsonpath_ng.ext` 库，该库：
- ✅ **无文件系统访问
- ✅ **无环境变量访问
- ✅ **无网络访问
- ✅ **仅在提供的 JSON 数据上操作

**JSONPath 是相对安全，无内置危险表达式验证机制。

---

## 三、空结果 vs 非法表达式

### 3.1 空结果（匹配失败）

**返回值：** 空字符串 `''`

**触发场景：**
1. JSONPath 表达式合法但未匹配到任何数据
2. jq 表达式合法但未匹配到任何数据

**代码位置：**
- `html_tools.py:452-454` - `_get_stripped_text_from_json_match()`
- `html_tools.py:560-562` - `extract_json_as_string()`

```python
if not match:
    return ''  # Re 265 - Just return an empty string when filter not found
```

### 3.2 非法表达式

**行为：**
- **JSONPath 非法表达式：** 抛出 `jsonpath_ng.parser.JsonPathParserError`
- **jq 非法表达式：** 抛出 `ValueError`（安全验证失败) 或 `jq._jq.InvalidExpression`（语法错误)

**异常处理：**
- 在 `extract_json_as_string()` 中，`json.JSONDecodeError` 被捕获并记录警告
- 最终也返回空字符串 `''`

**代码位置：
```python
try:
    stripped_text_from_html = _parse_json(json.loads(content.lstrip("\ufeff")), json_filter)
except json.JSONDecodeError as e:
    logger.warning(f"Error processing JSON {content[:20]}...{str(e)})
```

### 3.3 JSONNotFound 异常

**位置：** `html_tools.py:62-64` - `JSONNotFound` 类

**触发场景：** 文档中找不到任何可解析的 JSON blob

```python
if not bs_jsons:
    raise JSONNotFound("No parsable JSON found in this document")
```

**注意：** 此异常仅在从 HTML 中提取 JSON blob 失败时抛出。

### 3.4 总结

| 情况 | 返回值/行为 |
|------|-------------|
| 合法表达式，匹配成功 | 匹配结果（JSON 格式或原始字符串） |
| 合法表达式，无匹配 | 空字符串 `''` |
| 非法表达式（语法错误） | 空字符串 `''`（异常被捕获） |
| 找不到可解析的 JSON | 抛出 `JSONNotFound` 异常 |

---

## 四、安全边界总结

### 4.1 JSONPath (`json:`)

| 风险等级：✅ 低风险

| 功能限制：
- ✅ 仅在提供的 JSON 数据上操作
- ✅ 无文件系统访问
- ✅ 无环境变量访问
- ✅ 无网络访问
- ✅ 无进程控制

### 4.2 jq (`jq:` / `jqraw:`

| 风险等级：⚠️ 中高风险（有保护）

**已缓解措施：
- ✅ 黑名单方式阻止危险内置函数
- ✅ 基于正则表达式的模式匹配
- ⚠️ 可通过环境变量绕过
- ⚠️ 正则匹配可能存在绕过漏洞

**已阻止的风险：
- 环境变量泄露
- 文件读取
- 进程终止
- stderr 数据泄露

**潜在风险：**
- 复杂的 jq 表达式可能导致 CPU 占用过高（DoS）
- 正则表达式黑名单可能存在绕过（如字符串拼接、编码等）

### 4.3 安全建议

1. **优先使用 JSONPath**：对于简单需求优先使用 JSONPath
2. **jq 仅在必要时使用 jq**：仅在需要复杂数据处理时使用 jq
3. **不要启用绕过安全检查**：除非绝对必要，否则不要设置 `JQ_ALLOW_RISKY_EXPRESSIONS=true`
4. **监控 jq 表达式**：对用户提供的 jq 表达式进行审计
5. **设置执行超时**：考虑为 jq 执行设置超时限制

---

## 五、测试覆盖

**位置：** `tests/unit/test_jq_security.py`

测试用例：
1. ✅ 被阻止的内置函数必须抛出异常
2. ✅ 安全的表达式必须通过
3. ✅ `JQ_ALLOW_RISKY_EXPRESSIONS=true 必须绕过检查
4. ✅ 默认情况下必须启用阻止机制

**位置：** `tests/test_jsonpath_jq_selector.py`

测试用例：
1. ✅ JSONP 内容处理
2. ✅ 内嵌 HTML 中的 JSON 提取
3. ✅ 空结果返回空字符串
4. ✅ 找不到 JSON 时抛出 `JSONNotFound`

---

## 六、相关代码位置

| 文件名 | 行号 | 说明 |
|---------|------|------|
| `changedetectionio/html_tools.py` | 18-37 | `_JQ_BLOCKED_PATTERNS` 危险模式定义 |
| `changedetectionio/html_tools.py` | 39-54 | `validate_jq_expression()` 安全验证函数 |
| `changedetectionio/html_tools.py` | 62-64 | `JSONNotFound` 异常类 |
| `changedetectionio/html_tools.py` | 410-459 | `_parse_json()` 选择器执行核心 |
| `changedetectionio/html_tools.py` | 461-518 | `extract_json_blob_from_html()` HTML 中提取 JSON |
| `changedetectionio/html_tools.py` | 523-564 | `extract_json_as_string()` 主入口函数 |
| `changedetectionio/tests/unit/test_jq_security.py` | 1-94 | jq 安全单元测试 |
| `changedetectionio/tests/test_jsonpath_jq_selector.py` | 1-526 | JSONPath/jq 集成测试 |

---

*文档生成时间：2026-05-17*
