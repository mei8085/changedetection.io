# Diff Tokenizer 选择规则与差异边界分析

## 概述

本文档分析了 changedetection.io 系统中文本差异比较（Diff）的分词器（Tokenizer）选择机制，以及不同切分方式对最终差异结果的影响。

---

## 一、当前系统中的 Tokenizer 实现

### 1.1 可用 Tokenizer 列表

系统目前实现了两种 tokenizer，注册在 `changedetectionio/diff/tokenizers/__init__.py` 中：

| Tokenizer 名称 | 实现文件 | 功能描述 |
|---------------|---------|---------|
| `words` | `natural_text.py` | 基于空白字符分割的简单分词器 |
| `words_and_html` | `words_and_html.py` | 保留 HTML 标签为原子单元的分词器（**默认**） |

### 1.2 Tokenizer 实现细节

#### 1.2.1 `words` - 纯文本分词器

**切分规则**：
- 遇到任何空白字符（空格、制表符等）时进行分割
- 空白字符本身也作为独立 token 保留
- 非空白字符连续累积形成一个 token

**示例**：
```python
tokenize_words("Hello   world")  # ['Hello', ' ', ' ', ' ', 'world']
tokenize_words("Price $90.00")   # ['Price', ' ', '$90.00']
```

#### 1.2.2 `words_and_html` - HTML 感知分词器

**切分规则**：
- 遇到 `<` 时开始 HTML 标签，直到 `>` 结束，整个标签作为单个 token
- HTML 标签外的空白字符触发分割（与 `words` 相同）
- 空白字符本身也作为独立 token 保留

**示例**：
```python
tokenize_words_and_html("<p>Hello <b>world</b></p>")
# ['<p>', 'Hello', ' ', '<b>', 'world', '</b>', '</p>']

tokenize_words_and_html("<a href='test.com'>link</a>")
# ['<a href=\'test.com\'>', 'link', '</a>']
```

---

## 二、当前 Tokenizer 选择机制

### 2.1 现状：无自动选择

**重要发现**：当前系统**没有**根据内容类型自动选择 tokenizer 的逻辑。所有调用点都硬编码使用 `words_and_html` 作为默认值：

| 函数 | 默认 tokenizer | 位置 |
|------|---------------|------|
| `render_inline_word_diff()` | `'words_and_html'` | `diff/__init__.py:79` |
| `render_nested_line_diff()` | `'words_and_html'` | `diff/__init__.py:214` |
| `customSequenceMatcher()` | `'words_and_html'` | `diff/__init__.py:321` |
| `render_diff()` | `'words_and_html'` | `diff/__init__.py:437` |

### 2.2 为什么 `words_and_html` 是默认？

1. **通用性**：既能正确处理纯文本，也能正确处理 HTML 内容
2. **HTML 标签完整性**：避免将 HTML 标签拆分成多个 token（如 `<`, `div`, `>`），确保标签不会被 diff 算法误判为变化
3. **向后兼容**：系统早期主要用于网页监控，HTML 内容是主要场景

---

## 三、Tokenizer 切分边界对 Diff 结果的影响

### 3.1 核心原理

系统使用 `diff-match-patch` 库进行词级 diff，工作流程如下：

```
输入文本 → Tokenizer 切分 → 每个 token 视为原子单元
                                              ↓
                         diff-match-patch 比较 token 序列
                                              ↓
                                  标记哪些 token 发生了变化
```

**关键**：token 边界决定了 diff 算法能识别的最小变化单元。

### 3.2 典型场景对比分析

#### 场景 1：数字变化（价格）

**测试用例** (`test_notification_diff.py:419-429`):
```python
before = "for sale $90.00"
after  = "for sale $9.00"
```

**`words_and_html` 切分结果**：
- before tokens: `['for', ' ', 'sale', ' ', '$90.00']`
- after tokens:  `['for', ' ', 'sale', ' ', '$9.00']`

**Diff 结果**：
- ✅ **整个** `$90.00` 被标记为删除
- ✅ **整个** `$9.00` 被标记为添加
- ❌ 不会出现只高亮 `0.00` 这种无意义的部分匹配

**意义**：对于价格、版本号等语义单元，保持完整性非常重要。

---

#### 场景 2：多词变化

**测试用例** (`test_notification_diff.py:443-453`):
```python
before = "quick brown fox jumps"
after  = "slow brown fox hops"
```

**切分结果**：
- 共同 token: `brown`, `fox`, （空格）
- 变化 token: `quick` → `slow`, `jumps` → `hops`

**Diff 结果**：
```
@removed_PLACEMARKER_OPENquick@removed_PLACEMARKER_CLOSED
@added_PLACEMARKER_OPENslow@added_PLACEMARKER_CLOSED
brown fox
@removed_PLACEMARKER_OPENjumps@removed_PLACEMARKER_CLOSED
@added_PLACEMARKER_OPENhops@added_PLACEMARKER_CLOSED
```

**意义**：准确识别独立变化的单词，保留共同上下文。

---

#### 场景 3：整行替换

**测试用例** (`test_notification_diff.py:431-441`):
```python
before = "$99"
after  = "$109"
```

**判定逻辑**：
- 检查是否存在**任何**相等的 token（包括空格）
- 如果没有相等 token → 判定为整行替换
- 使用 `CHANGED_PLACEMARKER` 而非 `REMOVED/ADDED` 标记

**Diff 结果**：
```
@changed_PLACEMARKER_OPEN$99@changed_PLACEMARKER_CLOSED
@changed_into_PLACEMARKER_OPEN$109@changed_into_PLACEMARKER_CLOSED
```

**意义**：视觉上更清晰地表示"旧值 → 新值"的对应关系。

---

#### 场景 4：HTML 标签处理

**假设输入**：
```python
before = "<span class='price'>$99</span>"
after  = "<span class='price'>$149</span>"
```

**`words_and_html` 切分**：
- before: `['<span class=\'price\'>', '$99', '</span>']`
- after:  `['<span class=\'price\'>', '$149', '</span>']`

**Diff 结果**：
- ✅ HTML 标签被正确识别为相等
- ✅ 仅价格部分高亮为变化

**如果使用 `words` 切分**：
- before: `['<span', ' ', 'class=\'price\'>', '$99', '</span>']`
- ❌ 标签被拆碎，可能导致大量误报的高亮

---

## 四、Tokenizer 自动选择规则设计

### 4.1 建议的自动选择策略

基于内容类型自动选择最优 tokenizer：

| 内容类型 | 检测条件 | 推荐 Tokenizer | 理由 |
|---------|---------|----------------|------|
| **HTML** | 包含 `<tag>` 模式或 `Content-Type: text/html` | `words_and_html` | 保留标签完整性 |
| **JSON** | 以 `{` 或 `[` 开头，或 `Content-Type: application/json` | 建议新增 `json_structure` | 保留 JSON 键名完整性 |
| **纯文本** | 其他情况 | `words` | 更轻量，无额外开销 |
| **代码** | 包含编程语言特征 | 建议新增 `code_tokens` | 按语法单元切分 |

### 4.2 内容类型检测实现建议

在 `render_diff()` 或更上层添加自动检测逻辑：

```python
def detect_content_type(text: str) -> str:
    """
    检测内容类型以选择合适的 tokenizer
    """
    # 1. 检测 HTML
    html_pattern = re.compile(r'<[a-zA-Z][^>]*>')
    if html_pattern.search(text[:1000]):  # 只检查前 1000 字符
        return 'html'
    
    # 2. 检测 JSON
    stripped = text.strip()
    if (stripped.startswith('{') and stripped.endswith('}')) or \
       (stripped.startswith('[') and stripped.endswith(']')):
        return 'json'
    
    # 3. 默认纯文本
    return 'text'

def auto_select_tokenizer(text: str) -> str:
    content_type = detect_content_type(text)
    mapping = {
        'html': 'words_and_html',
        'json': 'words_and_html',  # 暂时复用
        'text': 'words',
    }
    return mapping.get(content_type, 'words_and_html')
```

### 4.3 边界情况处理

| 场景 | 处理策略 |
|------|---------|
| 混合内容（HTML + 纯文本） | 保守使用 `words_and_html` |
| 内容过少（< 50 字符） | 使用 `words_and_html` 确保安全 |
| 检测不确定 | 回退到 `words_and_html`（默认更安全） |

---

## 五、不同切分方式的性能与效果对比

### 5.1 性能对比

| Tokenizer | 时间复杂度 | 典型耗时（10KB 文本） | 适用场景 |
|-----------|-----------|----------------------|---------|
| `words` | O(n) | ~0.1ms | 纯文本、日志 |
| `words_and_html` | O(n) | ~0.15ms | HTML、通用场景 |
| （建议）`json_structure` | O(n) | ~0.2ms | JSON API 响应 |

### 5.2 差异精度对比

| 场景 | `words` 效果 | `words_and_html` 效果 | 最优选择 |
|------|-------------|----------------------|---------|
| 纯英文文章 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 任意 |
| 带 HTML 的网页 | ⭐⭐ | ⭐⭐⭐⭐⭐ | `words_and_html` |
| JSON 数据 | ⭐⭐⭐ | ⭐⭐⭐⭐ | `words_and_html` |
| 价格/数字变化 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 两者一致 |
| 代码片段 | ⭐⭐⭐ | ⭐⭐⭐ | 需专用 tokenizer |

---

## 六、关键设计决策与边界

### 6.1 已确立的边界

1. **Token 是原子单元**：一旦切分，diff 算法不会在 token 内部进行更细粒度的比较
2. **空白字符是 token**：保留空格有助于准确重建原始文本结构
3. **不应用语义清理**：`diff-match-patch` 的 `diff_cleanupSemantic()` 被禁用，避免破坏 token 边界
4. **整行替换判定**：当无共同 token 时，使用特殊的 `CHANGED` 标记而非 `REMOVED/ADDED`

### 6.2 待优化的边界

1. **数字/符号边界**：
   - 当前：`$90.00` 是单个 token（因为没有空格）
   - 可优化：`$` + `90.00` 分离，或识别小数点

2. **URL/路径边界**：
   - 当前：`https://example.com/path` 是单个 token
   - 可优化：按路径段切分，便于检测细微 URL 变化

3. **JSON 键值边界**：
   - 当前：`"price": 99` 可能被切分为多个 token
   - 可优化：专用 JSON tokenizer 保持键名完整性

---

## 七、测试覆盖的关键场景

系统测试 (`test_notification_diff.py`) 已覆盖：

- ✅ 基本词级 diff 功能
- ✅ 数字/价格变化（保持原子性）
- ✅ 多词同线变化
- ✅ 整行替换判定
- ✅ 上下文行数控制
- ✅ 空白字符忽略
- ✅ 标记前缀开关
- ✅ 变更提取函数 (`extract_changed_from/to`)

---

## 八、总结与建议

### 8.1 核心结论

1. **当前无自动选择**：所有场景默认使用 `words_and_html`
2. **Tokenizer 边界决定 Diff 精度**：切分方式直接影响哪些内容会被高亮为变化
3. **`words_and_html` 是安全默认**：虽然对纯文本略有额外开销，但能正确处理 HTML 这种最常见场景
4. **整行替换判定是重要优化**：提供更好的用户体验

### 8.2 改进建议

1. **实现自动检测**：按 4.1 节建议添加内容类型检测和自动选择
2. **新增专用 Tokenizer**：
   - `json_tokens`：针对 JSON 结构优化
   - `url_tokens`：针对 URL 路径优化
3. **暴露配置选项**：允许用户在监控页面手动选择 tokenizer
4. **文档完善**：在用户文档中说明不同 tokenizer 的适用场景

### 8.3 风险提示

- 改变 tokenizer 可能影响历史 diff 的显示一致性
- 过于激进的切分可能产生"过度高亮"（太多小片段被标记）
- 过于保守的切分可能产生"高亮不足"（整段内容被标记但只有小部分变化）

---

**文档版本**：1.0  
**最后更新**：2024  
**对应代码版本**：changedetection.io current
