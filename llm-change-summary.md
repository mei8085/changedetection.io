# LLM 变更摘要 - 上下文体积控制机制分析

## 1. 概述

本文档分析 changedetection.io 项目中 LLM 变更摘要功能的上下文体积控制机制，包括 Prompt 组装流程、Token Budget 裁剪顺序以及响应解析失败的处理方式。

## 2. 上下文体积控制机制

### 2.1 多层次字符限制机制

系统实现了多层级的字符限制控制：

| 限制级别 | 配置位置 | 默认值 | 说明 |
|---------|---------|--------|------|
| 全局输入字符限制 | `_get_max_input_chars()` | 100,000 | 通过环境变量 `LLM_MAX_INPUT_CHARS` 或 UI 设置可配置 |
| 快照上下文字符限制 | `prompt_builder.SNAPSHOT_CONTEXT_CHARS` | 3,000 | 评估调用时的当前页面状态摘录限制 |
| BM25 裁剪默认限制 | `bm25_trim.MAX_CONTEXT_CHARS` | 15,000 | BM25 相关性裁剪的默认字符数 |
| 预览内容限制 | `build_preview_prompt()` | 6,000 | 实时预览提取的页面内容限制 |
| 设置调用摘录限制 | `build_setup_prompt()` | 4,000 | 预过滤器设置调用的页面摘录限制 |

**关键代码位置**:
- `evaluator.py:34-45` - `_get_max_input_chars()`
- `prompt_builder.py:12` - `SNAPSHOT_CONTEXT_CHARS`
- `bm25_trim.py:12` - `MAX_CONTEXT_CHARS`

### 2.2 输入大小检查机制

在调用 LLM 之前会进行输入大小检查，超过限制时抛出 `LLMInputTooLargeError` 异常：

```python
def _check_input_size(text: str, max_chars: int) -> None:
    """Raise LLMInputTooLargeError if text exceeds max_chars."""
    if len(text) > max_chars:
        raise LLMInputTooLargeError(
            f"Change too large for AI summary ({len(text):,} chars, limit {max_chars:,})"
        )
```

**调用位置**:
- `summarise_change()` - 变更摘要生成前检查 (evaluator.py:521)
- `preview_extract()` - 实时预览提取前检查 (evaluator.py:598)
- `evaluate_change()` - 变更评估前检查 (evaluator.py:651)

## 3. Prompt 组装流程

### 3.1 变更摘要 Prompt 组装

**文件**: `prompt_builder.py:134-155` - `build_change_summary_prompt()`

Prompt 组装顺序：

```
1. URL (可选)
   ↓
2. Page title (可选)
   ↓
3. 用户自定义 Instructions (custom_prompt)
   ↓
4. What changed (diff) - 经过 _annotate_moved_lines() 预处理
```

**Diff 预处理**: `_annotate_moved_lines()` 函数会将同时出现在 `+` 和 `-` 两侧的行标记为 `~` 前缀，表示移动/重排序/琐碎变更，避免 LLM 误判。

### 3.2 系统 Prompt 分离设计

系统 Prompt 和用户 Prompt 分离：

- **系统 Prompt**: `build_change_summary_system_prompt()` - 通用 diff 读取规则和准确性要求
- **用户 Prompt**: `build_change_summary_prompt()` - 包含具体内容、URL、标题、用户自定义指令

这种设计允许用户完全自定义输出格式，而不会被硬编码的格式规则覆盖。

### 3.3 评估调用 Prompt 组装

**文件**: `prompt_builder.py:43-65` - `build_eval_prompt()`

```
1. URL (可选)
   ↓
2. Page title (可选)
   ↓
3. Intent
   ↓
4. Current page state (relevant excerpt) - 经过 BM25 裁剪 (可选)
   ↓
5. What changed (diff)
```

### 3.4 Prompt 级联解析

**文件**: `evaluator.py:156-173` - `resolve_llm_field()`

用户自定义 Prompt 的级联解析顺序：

```
1. Watch 级别的配置 → 2. Tag 级别的配置 → 3. 全局默认配置 → 4. 硬编码默认值
```

## 4. Token Budget 裁剪顺序

### 4.1 BM25 相关性裁剪

**文件**: `bm25_trim.py:15-52` - `trim_to_relevant()`

**裁剪流程**:

```
输入文本 + 查询意图
    ↓
1. 检查文本长度是否 <= max_chars → 是: 直接返回
    ↓ 否
2. 按行分割文本
    ↓
3. 尝试导入 rank_bm25 库
    ↓ 导入失败
    ├─→ 回退: 简单头部截断 (text[:max_chars])
    ↓ 导入成功
4. 对每行进行 tokenization (小写 + 空格分割)
    ↓
5. 构建 BM25Okapi 索引
    ↓
6. 计算每行相对于查询的 BM25 分数
    ↓
7. 按分数降序排序
    ↓
8. 按分数从高到低选择行，直到达到字符预算
    ↓
9. 按原始文档顺序重新排序选中的行
    ↓
输出: 相关性最高的行，保持原始顺序
```

**调用位置**:
- `build_eval_prompt()` - 评估调用的页面状态摘录 (prompt_builder.py:59)
- `build_setup_prompt()` - 设置调用的页面内容摘录 (prompt_builder.py:189)

### 4.2 输出 Token 动态调整

**文件**: `evaluator.py:116-118` - `_summary_max_tokens()`

```python
def _summary_max_tokens(diff: str, max_cap: int = LLM_DEFAULT_MAX_SUMMARY_TOKENS) -> int:
    """Scale completion tokens to diff size: floor 400, ~1 token per 4 chars, ceiling max_cap."""
    return max(400, min(len(diff) // 4, max_cap))
```

**调整规则**:
- 最小: 400 tokens (保底)
- 比例: 每 4 个字符分配约 1 个 token
- 最大: `LLM_DEFAULT_MAX_SUMMARY_TOKENS` (默认 3000)

### 4.3 本地产 Token 乘数

**文件**: `evaluator.py:121-149` - `apply_local_token_multiplier()`

针对自托管 OpenAI 兼容端点（如 vLLM、LM Studio、llama.cpp）的特殊处理：

- **激活条件**: `llm_cfg['provider_kind'] == 'openai_compatible'`
- **默认乘数**: 5x
- **配置范围**: 1-20x (UI 强制范围，代码中也有防御性限制)
- **目的**: 为推理模型（如 Qwen3、DeepSeek-R1、Gemma 3）提供足够的思考空间，避免因 `finish_reason='length'` 导致响应被截断

### 4.4 全局月度 Token 预算

**文件**: `evaluator.py:232-335`

**预算检查流程**:

```
调用 LLM 前
    ↓
1. 检查 `is_global_token_budget_exceeded()`
    ↓ 超出
2. 记录 WARNING 日志
    ↓
3. 变更摘要: 返回空字符串，不生成摘要
   变更评估: 开放失败 (important=True)，不抑制通知
    ↓ 未超出
4. 正常执行 LLM 调用
    ↓
5. 调用成功后: `accumulate_global_tokens()` 累加使用量
    - total_tokens
    - input_tokens
    - output_tokens
    - 估算的 USD 成本 (通过 litellm pricing)
```

**预算配置优先级**:
1. 环境变量 `LLM_TOKEN_BUDGET_MONTH` (最高优先级)
2. UI 设置中的 token budget
3. 0 = 无限制 (默认)

### 4.5 Per-Watch Token 限制

**文件**: `evaluator.py:342-370` - `_check_token_budget()`

```
检查维度:
├─ 每次检查限制 (max_tokens_per_check)
└─ 累计使用限制 (max_tokens_cumulative)

超限处理:
├─ 变更摘要: 抛出异常，由上层处理
└─ 变更评估: 开放失败 (important=True)，不抑制通知
```

## 5. 响应解析失败处理

### 5.1 JSON 提取与清理

**文件**: `response_parser.py:19-27` - `_extract_json()`

```
原始响应文本
    ↓
1. 去除首尾空白
    ↓
2. 移除 Markdown 代码块围栏 (```json 或 ```)
    ↓
3. 正则匹配第一个 { ... } 块 (DOTALL 模式)
    ↓
返回提取的 JSON 字符串
```

### 5.2 评估响应解析

**文件**: `response_parser.py:30-43` - `parse_eval_response()`

```
JSON 解析
    ↓
成功 → 返回 {'important': bool, 'summary': str}
    ↓
失败 (JSONDecodeError / AttributeError)
    ↓
返回默认值: {'important': False, 'summary': ''}
```

**注意**: 解析失败时默认 `important=False`，意味着不会触发通知。这是一个保守的安全默认值。

### 5.3 预览响应解析

**文件**: `response_parser.py:46-59` - `parse_preview_response()`

```
JSON 解析
    ↓
成功 → 返回 {'found': bool, 'answer': str}
    ↓
失败
    ↓
返回默认值: {'found': False, 'answer': ''}
```

### 5.4 设置响应解析

**文件**: `response_parser.py:62-84` - `parse_setup_response()`

```
JSON 解析
    ↓
成功
    ↓
额外清理: 拒绝位置选择器 (nth-child, nth-of-type, :eq(), [n], //*[n)
    ↓
返回: {'needs_prefilter': bool, 'selector': str|None, 'reason': str}
    ↓
失败
    ↓
返回默认值: {'needs_prefilter': False, 'selector': None, 'reason': ''}
```

### 5.5 LLM 调用层面的异常处理

**文件**: `evaluator.py:705-709`

```python
except Exception as e:
    logger.warning(f"LLM evaluation failed for {watch.get('uuid')}: {e}")
    # On failure: don't suppress the notification — pass through as important
    watch['llm_last_tokens_used'] = 0
    return {'important': True, 'summary': ''}
```

**设计意图**: LLM 调用失败时，采用"开放失败"策略，即不抑制通知，确保用户不会因为 LLM 故障而错过重要变更。

## 6. 缓存机制

### 6.1 变更摘要缓存

**文件**: `evaluator.py:433-491`

**缓存 Key 组成**:

```
MD5( diff_text + '\x00' + prompt )
    + DiffPrefs (all_changes, ignore_whitespace, show_removed, show_added)
    + 系统 Prompt 内容
    + max_summary_tokens 配置
```

这确保了：
- Diff 内容变化时缓存失效
- 用户 Prompt 变化时缓存失效
- UI 显示偏好变化时缓存失效
- 系统 Prompt 更新时缓存失效
- Token 限制变化时缓存失效

### 6.2 评估结果缓存

**文件**: `evaluator.py:654-658`

**缓存 Key**: `SHA256(intent + '||' + diff)`

每个唯一的 (intent, diff) 组合只评估一次，避免重复消耗 Token。

## 7. 总结

### 7.1 上下文体积控制要点

1. **多层级限制**: 从全局 100k 字符到各场景的精细限制
2. **智能裁剪**: BM25 相关性优先裁剪，保留语义完整性
3. **动态 Token 分配**: 根据 diff 大小动态调整输出 token 预算
4. **本地模型适配**: 为自托管模型提供额外的 Token 乘数
5. **预算保护**: 全局月度预算 + Per-Watch 双重保护

### 7.2 失败处理原则

1. **JSON 解析失败**: 保守默认值 (不触发通知)
2. **LLM 调用失败**: 开放失败 (触发通知，不遗漏变更)
3. **预算超限**: 变更摘要停止，变更评估开放失败
4. **输入过大**: 明确错误提示，不进行部分处理

### 7.3 关键设计决策

| 决策 | 原因 | 影响 |
|-----|------|------|
| BM25 裁剪而非简单截断 | 保留与 intent 相关的内容 | 上下文质量更高，但需要额外依赖 |
| 系统/用户 Prompt 分离 | 用户可完全自定义输出格式 | 灵活性高，用户 Prompt 拥有最终控制权 |
| 移动行预标记 (~前缀) | 避免 LLM 误判重排序为变更 | 减少误报，提高摘要准确性 |
| 调用失败时开放失败 | 不因为 LLM 故障错过重要变更 | 可能产生额外通知，但保证不遗漏 |
| 多级 Prompt 级联 | 灵活的配置继承机制 | Watch > Tag > Global > 硬编码默认 |
