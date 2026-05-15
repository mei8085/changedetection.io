# Diff Tokenizer 选择规则与差异边界分析

## 概述

本文档深入分析 changedetection.io 系统中文本差异比较（Diff）的分词器（Tokenizer）机制，包括：
- 内容类型信号来源与检测流程
- Tokenizer 参数在系统中的完整传递路径
- 两种 Tokenizer（`words` vs `words_and_html`）的切分边界对比
- 不同切分方式如何触发不同的差异标记
- 自动选择 Tokenizer 的设计方案与回退顺序

---

## 一、内容类型信号来源

### 1.1 信号检测链

内容类型检测发生在内容获取与处理流程的早期阶段：

```
HTTP 响应头
    ↓ (Content-Type)
guess_stream_type()  ← processors/magic.py
    ↓ (is_html / is_json / is_plaintext 等标记)
文本提取与预处理
    ↓
Diff 渲染
```

### 1.2 guess_stream_type() 多信号融合机制

`processors/magic.py` 中的 `guess_stream_type` 类采用**多信号融合**策略：

| 信号来源 | 检测方式 | 优先级 | 说明 |
|---------|---------|--------|------|
| **HTTP 头** | `Content-Type` 字段 | 高 | 最可信的来源 |
| **文件魔术** | `puremagic` 库检测 | 中 | 用于二进制文件识别 |
| **内容模式** | 前 200 字符模式匹配 | 高 | 检测 HTML 标签、JSON 结构、RSS 标记 |

**关键检测逻辑：**
```python
# 1. HTML 模式检测（前200字符）
has_html_patterns = any(p in test_content_normalized for p in HTML_PATTERNS)
# HTML_PATTERNS = ['<!doctype html', '<html', '<head', '<body', '<script', '<iframe', '<div']

# 2. JSON 检测（同时排除 JSONP 误报）
if re.match(r'^\w[\w.]*\s*\(', test_content):
    is_plaintext = True  # JSONP 视为纯文本

# 3. RSS/Feed 检测
if '<rss' in test_content or '<feed' in test_content:
    is_rss = True

# 4. PDF 检测
if '%pdf-1' in test_content:
    is_pdf = True
```

### 1.3 检测结果（状态标记）

最终产生的布尔标记：
- `is_html` - HTML 内容
- `is_json` - JSON 内容  
- `is_plaintext` - 纯文本内容
- `is_rss` - RSS/Atom feed
- `is_pdf` - PDF 文档
- `is_xml` - 通用 XML（非 RSS）

**重要：** 这些标记目前**仅用于内容提取预处理**，**尚未传递到 Diff 层**用于自动选择 Tokenizer。

---

## 二、Tokenizer 参数传递完整路径

### 2.1 当前调用链概览

```
┌─────────────────────────────────────────────────────────────────────┐
│  渲染层调用点                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  1. processors/text_json_diff/difference.py: render()             │
│     → diff.render_diff(..., word_diff=True)                        │
│     ⚠️  未传递 tokenizer 参数，使用默认值                          │
│                                                                     │
│  2. notification_service.py: add_rendered_diff_to_notification_vars │
│     → diff.render_diff(...)                                         │
│     ⚠️  未传递 tokenizer 参数，使用默认值                          │
│                                                                     │
│  3. tests/unit/test_notification_diff.py                           │
│     → diff.render_diff(..., tokenizer=?)                           │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  diff/__init__.py: render_diff()                                   │
│  默认值: tokenizer='words_and_html'                                │
├─────────────────────────────────────────────────────────────────────┤
│  → customSequenceMatcher(..., tokenizer=tokenizer)                 │
│     → 当 word_diff=True 且单行变化时                                │
│        → render_inline_word_diff(..., tokenizer=tokenizer)         │
│           → TOKENIZERS.get(tokenizer, tokenize_words_and_html)     │
│              → 实际分词函数                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 详细传递路径

**调用入口 1: Web UI 历史对比**
```python
# processors/text_json_diff/difference.py:183-191
content = diff.render_diff(
    previous_version_file_contents=from_version_file_contents,
    newest_version_file_contents=to_version_file_contents,
    include_replaced=diff_prefs['replaced'],
    include_added=diff_prefs['added'],
    include_removed=diff_prefs['removed'],
    include_equal=diff_prefs['changesOnly'],
    ignore_junk=diff_prefs['ignoreWhitespace'],
    word_diff=diff_prefs['type'] == 'diffWords',
    # ⚠️ 缺失: tokenizer=? 参数
)
```

**调用入口 2: 通知渲染**
```python
# notification_service.py:106
raw = diff_module.render_diff(prev_snapshot or '', current_snapshot or '', word_diff=True)
# ⚠️ tokenizer 使用默认值 'words_and_html'
```

**调用入口 3: 自定义序列匹配器**
```python
# diff/__init__.py:404-405
# 当 word_diff=True 且是单行变化时
inline_diff, has_changes = render_inline_word_diff(
    before_lines[0], after_lines[0], 
    ignore_junk=ignore_junk, 
    tokenizer=tokenizer,  # 传递 tokenizer
    include_change_type_prefix=include_change_type_prefix
)
```

**Tokenzier 注册表查找**
```python
# diff/__init__.py:107
tokenizer_func = TOKENIZERS.get(tokenizer, tokenize_words_and_html)
# 注册表在 diff/tokenizers/__init__.py 中定义
# TOKENIZERS = {
#     'words': tokenize_words,
#     'words_and_html': tokenize_words_and_html,
# }
```

### 2.3 当前硬编码现状

| 调用位置 | 是否传递 tokenizer | 使用的值 |
|---------|-------------------|---------|
| Web UI diff render | ❌ 否 | 默认 `words_and_html` |
| 通知系统 diff | ❌ 否 | 默认 `words_and_html` |
| 单元测试 | ⚠️ 部分是 | 测试特定场景 |
| API 端点 | ❌ 否 | 默认 `words_and_html` |

**结论：** 系统所有主要路径都硬编码使用 `words_and_html`，没有利用内容类型检测结果。

---

## 三、两种 Tokenizer 的切分边界对比（实测数据）

基于真实测试数据，以下是两种 tokenizer 的详细对比。

### 3.1 Tokenizer 实现源码

**`words` tokenizer (`natural_text.py`):**
```python
def tokenize_words(text: str) -> List[str]:
    tokens = []
    current = ''
    for char in text:
        if char.isspace():
            if current:
                tokens.append(current)
                current = ''
            tokens.append(char)  # 空格本身也作为 token 保留
        else:
            current += char
    if current:
        tokens.append(current)
    return tokens
```

**`words_and_html` tokenizer (`words_and_html.py`):**
```python
def tokenize_words_and_html(text: str) -> List[str]:
    tokens = []
    current = ''
    in_tag = False
    for char in text:
        if char == '<':
            if current:
                tokens.append(current)
                current = ''
            current = '<'
            in_tag = True
        elif char == '>' and in_tag:
            current += '>'
            tokens.append(current)
            current = ''
            in_tag = False
        elif char.isspace() and not in_tag:
            if current:
                tokens.append(current)
                current = ''
            tokens.append(char)
        else:
            current += char
    if current:
        tokens.append(current)
    return tokens
```

### 3.2 对比测试 1: 纯文本（无 HTML）

**输入:**
```
Before: Product Price: $99.99 - In Stock
After:  Product Price: $129.99 - In Stock
```

**切分结果（两种 Tokenizer 完全相同）:**
```
Token 序列: ['Product', ' ', 'Price:', ' ', '$99.99', ' ', '-', ' ', 'In', ' ', 'Stock']
Token 数量: 11 个
```

**关键观察:**
- ✅ 两种 tokenizer 对纯文本**输出完全一致**
- ✅ 价格 `$99.99` 是单个 token（无空格分隔）
- ✅ 空格都作为独立 token 保留（用于准确重建文本）

### 3.3 对比测试 2: 带 HTML 标签内容

**输入:**
```html
Before: <span class="price">Price: $99</span><span class="stock">In Stock</span>
After:  <span class="price">Price: $149</span><span class="stock">In Stock</span>
```

#### `words` tokenizer 切分结果（9 个 tokens）:
```
[0] '<span'
[1] ' '
[2] 'class="price">Price:'
[3] ' '
[4] '$99</span><span'       ⚠️ HTML 标签与内容混在一起
[5] ' '
[6] 'class="stock">In'
[7] ' '
[8] 'Stock</span>'           ⚠️ 结束标签与文本混在一起
```

#### `words_and_html` tokenizer 切分结果（10 个 tokens）:
```
[0] '<span class="price">'  ✅ 完整 HTML 标签
[1] 'Price:'
[2] ' '
[3] '$99'                   ✅ 价格单独 token
[4] '</span>'               ✅ 完整结束标签
[5] '<span class="stock">'  ✅ 下一个开始标签
[6] 'In'
[7] ' '
[8] 'Stock'
[9] '</span>'               ✅ 完整结束标签
```

#### Token 交集分析（影响差异标记）:
| Tokenizer | 共同 token 数量 | 说明 |
|-----------|----------------|------|
| `words` | **5 个** | 碎片化，标签被拆分 |
| `words_and_html` | **7 个** | 4 个 HTML 标签都保持完整 |

### 3.4 对比测试 3: 复杂 HTML（带多个属性）

**输入:**
```html
<div class="product" data-id="123"><h1>Old Title</h1><p>Description here</p></div>
```

#### `words` tokenizer 结果（9 个 tokens，高度碎片化）:
```
[0] '<div'
[1] ' '
[2] 'class="product"'
[3] ' '
[4] 'data-id="123"><h1>Old'
[5] ' '
[6] 'Title</h1><p>Description'
[7] ' '
[8] 'here</p></div>'
```

#### `words_and_html` tokenizer 结果（12 个 tokens，结构清晰）:
```
[0] '<div class="product" data-id="123">'  ✅ 完整标签 + 属性
[1] '<h1>'
[2] 'Old'
[3] ' '
[4] 'Title'
[5] '</h1>'
[6] '<p>'
[7] 'Description'
[8] ' '
[9] 'here'
[10] '</p>'
[11] '</div>'
```

### 3.5 对比测试 4: JSON 内容

**输入:**
```json
{"price": 99, "stock": true, "name": "Product A"}
```

**结果:** 两种 tokenizer **输出完全相同**（13 个 tokens），因为没有 `<` 和 `>` 字符触发 HTML 标签识别。

### 3.6 边界情况: 格式不规范的 HTML

**输入:** `< div >Hello< /div >`

| Tokenizer | 结果 | 问题 |
|-----------|------|------|
| `words` | `['<', ' ', 'div', ' ', '>Hello<', ' ', '/div', ' ', '>']` | 极度碎片化 |
| `words_and_html` | `['< div >', 'Hello', '< /div >']` | ⚠️ 把空格也包含在 "标签" 中，但至少保持了单元完整性 |

### 3.7 切分边界总结表

| 场景 | `words` tokenizer | `words_and_html` tokenizer | 最优选择 |
|------|------------------|---------------------------|---------|
| 纯英文文章 | 按空格切分，正确 | 与 words 完全相同 | 任意 |
| 带 HTML 的网页 | ❌ 标签被空格拆分 | ✅ 标签保持完整 | `words_and_html` |
| JSON 数据 | 按空格切分 | 与 words 完全相同 | 任意 |
| 价格变化 ($99→$149) | ✅ 单个 token | ✅ 单个 token | 任意 |
| 代码片段 | 按空格切分 | 与 words 相同（无 <>） | 需专用 tokenizer |
| 格式不规范 HTML | 极度碎片化 | 保持单元（但包含空格） | `words_and_html` |

---

## 四、切分边界如何触发不同的差异标记

### 4.1 Diff 标记类型

系统使用三种占位符（Placemarker）标记差异：

| 标记类型 | 触发条件 | 占位符 |
|---------|---------|--------|
| **REMOVED** | 纯删除（对应行在新版本不存在） | `@removed_PLACEMARKER_OPEN...CLOSED` |
| **ADDED** | 纯新增（对应行在旧版本不存在） | `@added_PLACEMARKER_OPEN...CLOSED` |
| **CHANGED** | 整行替换（无共同 token 时） | `@changed_PLACEMARKER_OPEN...CLOSED` |
| **CHANGED_INTO** | 替换后的新内容 | `@changed_into_PLACEMARKER_OPEN...CLOSED` |

### 4.2 整行替换判定逻辑

```python
# render_inline_word_diff 中的判定
whole_line_replaced = not any(op == 0 and text.strip() for op, text in diffs)
#                      ↑ 没有任何相等（op==0）的非空白 token → 判定为整行替换

if whole_line_replaced:
    # 使用 CHANGED / CHANGED_INTO 标记
    result = f'{CHANGED_PLACEMARKER_OPEN}{removed_full}{CHANGED_PLACEMARKER_CLOSED}'
    result += f'{CHANGED_INTO_PLACEMARKER_OPEN}{added_full}{CHANGED_INTO_PLACEMARKER_CLOSED}'
else:
    # 混合模式，分别使用 REMOVED / ADDED
    for op, text in diffs:
        if op == -1:  # 删除
            result += f'{REMOVED_PLACEMARKER_OPEN}{text}{REMOVED_PLACEMARKER_CLOSED}'
        elif op == 1:  # 新增
            result += f'{ADDED_PLACEMARKER_OPEN}{text}{ADDED_PLACEMARKER_CLOSED}'
```

### 4.3 Tokenizer 选择对标记的实际影响

**场景：HTML 内容中仅价格变化**

```html
版本 A: <span class="price">$99</span>
版本 B: <span class="price">$149</span>
```

#### 情况 1: 使用 `words` tokenizer

```
tokens A: ['<span', ' ', 'class="price">$99</span>']
tokens B: ['<span', ' ', 'class="price">$149</span>']
                 ↑         ↑
              这两个相等    ↑
                         这个不同

共同 token: 2 个 ('<span', ' ')
→ ❗ 有共同 token → 不会触发整行替换
→ 使用 REMOVED/ADDED 标记分别标记删除和新增
→ 但被删除和新增的 token 包含了整个 class="price">...</span>
→ 高亮区域过大，视觉噪声多
```

#### 情况 2: 使用 `words_and_html` tokenizer

```
tokens A: ['<span class="price">', '$99', '</span>']
tokens B: ['<span class="price">', '$149', '</span>']
                 ↑                    ↑
             这两个完全相等         只有价格不同

共同 token: 2 个完整标签 + 1 个结束标签 = 3 个
→ ✅ 只有 `$99` ↔ `$149` 被标记为变化
→ HTML 标签被正确识别为相等，不高亮
→ 高亮区域精确，视觉干净
```

### 4.4 标记影响总结表

| 场景 | `words` tokenizer 效果 | `words_and_html` 效果 |
|------|----------------------|----------------------|
| **纯文本变化** | 精确高亮变化词 | 与 words 相同，精确 |
| **HTML + 价格变化** | ❌ 整段标签 + 价格被高亮 | ✅ 只有价格被高亮 |
| **整句变化（无共同词）** | 触发 CHANGED 标记 | 触发 CHANGED 标记（相同） |
| **多词同行变化** | 每个变化词独立标记 | 每个变化词独立标记（相同） |
| **通知模板提取** | `extract_changed_from/to` 可能包含多余 HTML 标签 | ✅ 提取更干净，只有变化内容 |

---

## 五、自动选择 Tokenizer 设计方案

### 5.1 自动选择回退顺序

基于内容类型检测结果，建议以下优先级顺序：

```
                        ┌─────────────────┐
                        │  检测内容类型    │
                        └────────┬────────┘
                                 ↓
          ┌─────────────────────────────────────┐
          │  1. is_html == True ?               │
          │     → 使用 words_and_html           │
          └───────────────────┬─────────────────┘
                              ↓ 否
          ┌─────────────────────────────────────┐
          │  2. is_json == True ?               │
          │     → 使用 words（JSON无HTML标签）   │
          └───────────────────┬─────────────────┘
                              ↓ 否
          ┌─────────────────────────────────────┐
          │  3. is_plaintext == True ?          │
          │     → 使用 words                     │
          └───────────────────┬─────────────────┘
                              ↓ 否/不确定
          ┌─────────────────────────────────────┐
          │  4. 安全回退                        │
          │     → 使用 words_and_html           │
          └─────────────────────────────────────┘
```

### 5.2 实现方案：render_diff 层自动检测

**修改位置:** `changedetectionio/diff/__init__.py`

```python
import re

def detect_content_type_for_tokenizer(text: str) -> str:
    """
    轻量级内容类型检测，用于选择合适的 tokenizer
    只检测前 500 字符以保证性能
    """
    if not text:
        return 'text'
    
    sample = text[:500].lower()
    
    # 检测 HTML 标签模式（查找完整的标签）
    # 注意：必须是 <tag...> 格式，不能有空格紧跟 <
    html_pattern = re.compile(r'<[a-zA-Z][^>]*>')
    if html_pattern.search(sample):
        return 'html'
    
    # 纯文本/其他
    return 'text'

def auto_select_tokenizer(text: str) -> str:
    """
    根据内容自动选择最优 tokenizer
    """
    content_type = detect_content_type_for_tokenizer(text)
    mapping = {
        'html': 'words_and_html',
        'text': 'words',
    }
    return mapping.get(content_type, 'words_and_html')
```

**修改 render_diff 签名:**
```python
def render_diff(
    previous_version_file_contents: str,
    newest_version_file_contents: str,
    include_equal: bool = False,
    include_removed: bool = True,
    include_added: bool = True,
    include_replaced: bool = True,
    include_change_type_prefix: bool = True,
    patch_format: bool = False,
    word_diff: bool = True,
    context_lines: int = 0,
    case_insensitive: bool = False,
    ignore_junk: bool = False,
    tokenizer: str = 'auto'  # ← 新增默认值 'auto'
) -> str:
    """
    Args:
        tokenizer: Tokenizer 名称，支持 'auto'（自动检测）, 'words', 'words_and_html'
    """
    # 自动检测逻辑
    if tokenizer == 'auto':
        # 使用两个版本的内容联合检测
        combined_sample = (previous_version_file_contents or '')[:250] + (newest_version_file_contents or '')[:250]
        tokenizer = auto_select_tokenizer(combined_sample)
    
    # 原有逻辑继续...
```

### 5.3 上层调用修改

**processor 层传递内容类型信息:**
```python
# processors/text_json_diff/difference.py
# 在 render 函数中，我们已经有 stream_content_type 对象！

content = diff.render_diff(
    previous_version_file_contents=from_version_file_contents,
    newest_version_file_contents=to_version_file_contents,
    # ... 其他参数
    word_diff=diff_prefs['type'] == 'diffWords',
    # ↓ 新增：传递已有的检测结果
    tokenizer='words_and_html' if stream_content_type.is_html else 'words'
)
```

**优势：** 使用已有的检测结果，避免重复检测开销。

### 5.4 兼容性考虑

| 变更点 | 影响 | 处理方案 |
|-------|------|---------|
| tokenizer 默认值改为 `'auto'` | 现有代码如果依赖旧默认值 | 保持向后兼容：`'auto'` 在无 HTML 时回退到 `words`，有 HTML 时用 `words_and_html` |
| 纯文本内容现在用 `words` | 行为变化 | 两种 tokenizer 对纯文本输出相同，无影响 |
| API 响应格式 | 可能变化 | 变化更精确是改进 |

---

## 六、性能与效果权衡分析

### 6.1 性能开销

| 操作 | 耗时估计 | 说明 |
|------|---------|------|
| `tokenize_words` | O(n) | 最快 |
| `tokenize_words_and_html` | O(n) | 稍慢（多一个状态变量检查） |
| `detect_content_type_for_tokenizer` | O(500) | 只检查前 500 字符，可忽略 |
| 每次 diff 的总开销 | < 1ms | 与页面抓取相比可忽略 |

### 6.2 效果对比矩阵

| 维度 | 硬编码 `words_and_html` | 自动选择 | 纯 `words` |
|------|-----------------------|---------|-----------|
| HTML 高亮精度 | ✅ 优秀 | ✅ 优秀 | ❌ 差 |
| 纯文本高亮精度 | ✅ 优秀 | ✅ 优秀 | ✅ 优秀 |
| JSON 高亮精度 | ✅ 良好 (与 words 相同) | ✅ 良好 | ✅ 良好 |
| 通知变更提取 | ⚠️ 可能包含多余标签 | ✅ 优化 | ❌ 更差 |
| 代码复杂度 | 低 | 中等 | 低 |
| 向后兼容性 | 100% | >99% | 低（破坏HTML场景） |

---

## 七、关键设计决策记录

### 7.1 已确定的设计边界

1. **Token 是原子比较单元**
   - diff-match-patch 不会在 token 内部比较字符
   - token 边界直接决定高亮精度

2. **空白字符必须保留为独立 token**
   - 用于准确重建原始文本格式
   - 否则无法正确还原空格位置

3. **不应用语义清理（diff_cleanupSemantic）**
   - 避免破坏 token 边界完整性
   - 例如 `$90.00` → `$9.00` 保持为整体变化，不拆分成字符级

4. **整行替换判定是用户体验优化**
   - 无共同 token 时使用 CHANGED 标记而非 REMOVED+ADDED
   - 视觉上更清晰表达"旧值 → 新值"关系

### 7.2 待验证的边界问题

1. **JSON 专用 tokenizer**
   - 当前两种 tokenizer 对 JSON 的切分是相同的
   - 但 JSON 的键值对可以有更智能的切分
   - 例如：`"price": 99` 切分为 `['"price":', ' ', '99']` 是正确的

2. **正则表达式提取后的内容**
   - 用户自定义提取规则可能产生混合内容
   - 需要更灵活的策略

3. **RSS/XML 内容**
   - 标签格式与 HTML 类似但语义不同
   - 是否需要专用处理？

---

## 八、实施建议

### 8.1 短期方案（低风险）

1. 在 `render_diff` 中添加 `tokenizer='auto'` 选项
2. 实现轻量级的 HTML 检测（前 500 字符）
3. 保持 `words_and_html` 为默认回退值

### 8.2 中期方案

1. 利用 processor 层已有的 `stream_content_type` 检测结果
2. 在所有调用点传递 tokenizer 参数（不再依赖默认值）
3. 添加单元测试覆盖 tokenizer 自动选择逻辑

### 8.3 长期优化方向

1. 考虑 JSON 专用 tokenizer（按结构切分但不破坏格式）
2. 考虑 URL 专用 tokenizer（按路径段切分）
3. 为高级用户添加 tokenizer 选择 UI（如在 Watch 设置中）

---

## 九、测试用例覆盖

系统现有测试已覆盖 (`test_notification_diff.py`):

- ✅ 基础 diff 输出
- ✅ 词级 diff（word_diff=True/False）
- ✅ 价格变化原子性 (`$90.00` → `$9.00`)
- ✅ 多词同行变化
- ✅ 整行替换标记触发
- ✅ 上下文行数控制
- ✅ 空白字符忽略
- ✅ 标记前缀开关
- ✅ 变更提取函数 (`extract_changed_from/to`)

**需新增测试：**
- ⏳ tokenizer 自动选择逻辑
- ⏳ 不同内容类型的 diff 输出对比
- ⏳ 边界情况（混合内容、空内容、极短内容）

---

## 总结

| 关键发现 | 说明 |
|---------|------|
| **当前现状** | 所有调用硬编码使用 `words_and_html` |
| **内容检测已存在** | `guess_stream_type` 在 processor 层运行，但结果未传递到 diff 层 |
| **纯文本等价性** | 对无 HTML 的内容，两种 tokenizer 输出完全相同 |
| **HTML 场景差异大** | `words` 会拆分 HTML 标签，导致高亮区域过大 |
| **自动选择低风险** | 纯文本场景无变化，只改进 HTML 场景 |
| **回退安全** | 不确定时回退到 `words_and_html` 总是安全的 |

**推荐行动：** 实施短期方案，在 `render_diff` 层添加轻量级自动检测，风险极低但能显著提升 HTML 内容的 diff 高亮精度。
