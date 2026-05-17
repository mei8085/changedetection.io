# LLM 变更摘要 - 上下文体积控制机制分析

## 1. 概述

本文档分析 changedetection.io 项目中 LLM 变更摘要功能的上下文体积控制机制，包括 Prompt 组装流程、Token Budget 裁剪顺序、五条路径的预算生效时机、超限后行为、双重累加 Bug 分析、restock plugin 计数链路、测试覆盖情况以及响应解析失败的处理方式。

## 2. 上下文体积控制机制

### 2.1 多层次字符限制机制

系统实现了多层级的字符限制控制：

| 限制级别 | 配置位置 | 默认值 | 说明 |
|---------|---------|--------|------|
| 全局输入字符限制 | `_get_max_input_chars()` | 100,000 | 通过环境变量 `LLM_MAX_INPUT_CHARS` 或 UI 设置可配置 |
| 快照上下文字符限制 | `prompt_builder.SNAPSHOT_CONTEXT_CHARS` | 3,000 | 评估调用时的当前页面状态摘录限制 |
| BM25 裁剪默认限制 | `bm25_trim.MAX_CONTEXT_CHARS` | 15,000 | BM25 相关性裁剪的默认字符数 |
| 预览内容限制 | `build_preview_prompt()` | 6,000 | 实时预览提取的页面内容限制 |
| 设置调用摘录限制 | `build_setup_prompt()` | 4,000 | 预过滤器设置调用的页面内容摘录限制 |
| restock 内容限制 | `llm_restock._MAX_CONTENT_CHARS` | 8,000 | 库存价格提取页面内容限制 |

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
- ⚠️ `run_setup()` - **完全不进行**输入大小检查
- ⚠️ `restock plugin` - **完全不进行**输入大小检查

## 3. 五条 LLM 路径的 Token Budget 生效顺序与超限行为

### 3.1 关键发现 1：`_check_token_budget` 的行为

**重要结论**：`_check_token_budget()` 函数只返回 `bool`，从不抛出异常。在大多数场景下，其返回值甚至被忽略，仅用于日志记录。

```python
def _check_token_budget(watch, cfg, tokens_this_call: int = 0) -> bool:
    """
    Check token budget limits.  Returns True if within budget, False if exceeded.
    Also accumulates tokens_this_call into watch['llm_tokens_used_cumulative'].
    """
    if tokens_this_call > 0:
        current = watch.get('llm_tokens_used_cumulative') or 0
        watch['llm_tokens_used_cumulative'] = current + tokens_this_call
```

### 3.2 关键发现 2：`summarise_change` 存在双重累加 Bug

**严重 Bug**：`summarise_change()` 中对 `llm_tokens_used_cumulative` 执行了**两次累加**，导致累计值是实际使用量的 2 倍，会提前触发累计阈值。

**代码位置**：`evaluator.py:559-561`

```python
# 第 559 行：第一次累加 — 在 _check_token_budget() 内部执行
_check_token_budget(watch, cfg, tokens)  # → 内部执行 watch['llm_tokens_used_cumulative'] += tokens

# 第 561 行：第二次累加 — 手动重复执行
watch['llm_tokens_used_cumulative'] = (watch.get('llm_tokens_used_cumulative') or 0) + tokens
```

**影响**：实际 token 使用量被翻倍计算，导致 Per-Watch 累计预算阈值提前触发。

**对比验证**：
- `evaluate_change()`：仅通过 `_check_token_budget()` 累加一次 ✓ (evaluator.py:712)
- `run_setup()`：仅通过 `_check_token_budget()` 累加一次 ✓ (evaluator.py:403)
- `preview_extract()`：完全不累加 Watch Token ✓ (evaluator.py 第 579-626 行无 _check_token_budget 调用)
- `restock plugin`：在 processor.py 中手动累加，无重复累加问题 ✓ (processor.py:522-523)

#### 可复现实例分析

**场景**：假设配置了 `max_tokens_cumulative = 1000`

```python
# 第一次调用 summarise_change，实际使用 400 tokens
_check_token_budget(watch, cfg, 400)  # → watch['llm_tokens_used_cumulative'] = 400
watch['llm_tokens_used_cumulative'] += 400  # → 第二次累加 → 800 ❌

# 第二次调用 summarise_change，再使用 300 tokens
_check_token_budget(watch, cfg, 300)  # → 800 + 300 = 1100
watch['llm_tokens_used_cumulative'] += 300  # → 1100 + 300 = 1400 ❌

# 结果：累计值显示 1400（已超过 1000 阈值），但实际仅使用 700 tokens
```

**后果**：
1. `evaluate_change()` 调用前的预检查会错误地认为预算已超限（1400 > 1000）
2. 提前触发"开放失败"逻辑，`important=True`，导致不应发送的通知被发送
3. 用户认为已使用 1400 tokens，但实际只消耗了 700 tokens
4. restock plugin 正常累加的 token 会加剧这个问题，导致阈值更快被触发

## 4. 统一计数链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           LLM 调用计数与预算检查链路                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ run_setup   │  │ summarise   │  │ evaluate    │  │ preview     │  │
│  │ setup       │  │ change      │  │ change      │  │ extract     │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
│         │                │                  │                  │           │
│  调用前检查              │                  │                  │           │
│  ─────────               │                  │                  │           │
│  ✗ 全局预算              │ ✓ 全局预算        │ ✓ 全局预算        │ ✗ 全局预算   │
│  ✗ Per-Watch             │ ✗ Per-Watch       │ ✓ Per-Watch       │ ✗ Per-Watch  │
│  ✗ 输入大小              │ ✓ 输入大小        │ ✓ 输入大小        │ ✓ 输入大小   │
│         │                │                  │                  │           │
│         ▼                ▼                  ▼                  ▼           │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │                         LLM API 调用                                    │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                │                  │                  │           │
│  调用后记账              │                  │                  │           │
│  ─────────               │                  │                  │           │
│  ✓ accumulate_global      │ ✓ accumulate_global │ ✓ accumulate_global │ ✓ accumulate_global│
│  ✓ _check_token_budget    │ ✓ _check_token_budget │ ✓ _check_token_budget │ ✗ 不调用        │
│    (返回值忽略)           │   (返回值忽略)       │   (返回值忽略)       │                 │
│  ✗ 无手动累加            │ ❌ 双重累加 Bug      │ ✗ 无手动累加        │ ✗ Watch不累加   │
│         │                │                  │                  │           │
│         └───────────┬────┴──────────┬───────┘          ┌──────────────┘│
│                     │                │                   │              │
│                     ▼                ▼                   ▼              │
│            ┌─────────────┐  ┌─────────────┐   ┌─────────────┐          │
│            │ 全局 token  │  │ llm_last    │   │ llm_tokens  │          │
│            │ 计数        │  │ _tokens_used│   │ _used_      │          │
│            │             │  │ (单次)      │   │ cumulative  │          │
│            │ (所有路径)  │  │ 除 preview  │   │ (累计)       │          │
│            └─────────────┘  └─────────────┘   └─────────────┘          │
│                                                                             │
│  restock plugin                                                            │
│  ─────────────                                                              │
│  ✗ 调用前：完全无检查（全局、Per-Watch、输入大小都不检查）                 │
│  ✓ 调用后：processor.py 手动写入 llm_last_tokens_used                      │
│  ✓ 调用后：processor.py 手动写入 llm_tokens_used_cumulative                │
│  ✓ 调用后：plugin 内部调用 accumulate_global_tokens                        │
│  ✗ 完全不调用 _check_token_budget()                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 5. 五条路径的详细对比

### 5.1 run_setup (设置预过滤器)

**代码位置**：`evaluator.py:373-410`

**生效顺序**：
```
调用 LLM 前
    ↓
1. 检查 LLM 是否配置 → 未配置: return
    ↓
2. 检查是否有 intent → 无 intent: return
    ↓
3. ⚠️ 完全不检查全局月度预算
    ↓
4. ⚠️ 完全不检查 Per-Watch 累计预算
    ↓
5. ⚠️ 完全不检查输入字符大小
    ↓
6. 执行 LLM 调用
    ↓
调用 LLM 后（记账阶段）
    ↓
7. 调用 _check_token_budget() → 返回值被 IGNORED!
   - 仅记录 WARNING 日志
   - tokens 仍会被累加到 watch['llm_tokens_used_cumulative']
    ↓
8. 累加全局 Token 计数
```

### 5.2 summarise_change (变更摘要)

**代码位置**：`evaluator.py:494-572`

**生效顺序**：
```
调用 LLM 前
    ↓
1. 检查 LLM 是否配置 → 未配置: 返回空字符串
    ↓
2. 检查全局月度预算是否超限 → 超限: 返回空字符串，记录 WARNING
    ↓
3. 检查输入字符大小 → 超限: 抛出 LLMInputTooLargeError
    ↓
4. ⚠️ 完全不检查 Per-Watch 累计预算
    ↓
5. 执行 LLM 调用
    ↓
调用 LLM 后（记账阶段）
    ↓
6. 调用 _check_token_budget() → 返回值被 IGNORED!
   - 第一次累加 tokens 到 watch['llm_tokens_used_cumulative']
    ↓
7. ⚠️ 手动第二次累加 tokens（双重累加 Bug!）
   - watch['llm_tokens_used_cumulative'] += tokens 再次执行
    ↓
8. 累加全局 Token 计数
```

### 5.3 evaluate_change (变更评估)

**代码位置**：`evaluator.py:633-728`

**生效顺序**：
```
调用 LLM 前
    ↓
1. 检查 LLM 是否配置 → 未配置: 返回 None
    ↓
2. 检查是否有 intent → 无 intent: 返回 None
    ↓
3. 检查输入字符大小 → 超限: 抛出 LLMInputTooLargeError
    ↓
4. 缓存命中检查 → 命中: 直接返回缓存结果
    ↓
5. 检查全局月度预算是否超限 → 超限: 返回 {'important': True, 'summary': ''}
    ↓
6. 检查 Per-Watch 累计预算是否超限 → 超限: 返回 {'important': True, 'summary': ''}
    ↓
7. 执行 LLM 调用
    ↓
调用 LLM 后（记账阶段）
    ↓
8. 调用 _check_token_budget() → 返回值被 IGNORED!
   - 仅记录 WARNING 日志
   - tokens 仍会被累加
    ↓
9. 累加全局 Token 计数
    ↓
10. 缓存结果
```

### 5.4 preview_extract (实时预览)

**代码位置**：`evaluator.py:579-626`

**生效顺序**：
```
调用 LLM 前
    ↓
1. 检查 LLM 是否配置 → 未配置: 返回 None
    ↓
2. 检查是否有 intent / 内容是否为空 → 返回 None
    ↓
3. 检查输入字符大小 → 超限: 抛出 LLMInputTooLargeError
    ↓
4. ⚠️ 完全不检查全局月度预算
    ↓
5. 执行 LLM 调用
    ↓
调用 LLM 后（记账阶段）
    ↓
6. ⚠️ 完全不检查 Per-Watch Token Budget!
   - 无 _check_token_budget() 调用
   - 不累加 watch['llm_tokens_used_cumulative']
    ↓
7. 仅累加全局 Token 计数
```

### 5.5 restock plugin (库存价格提取)

**代码位置**：
- `llm_restock.py:190-299` - plugin 实现
- `processor.py:508-533` - 调用与记账

**生效顺序**：
```
调用 LLM 前
    ↓
1. 检查 datastore 是否已注入 → 未注入: return None
    ↓
2. 检查用户设置是否启用 fallback 提取 → 未启用: return None
    ↓
3. 检查 LLM 是否配置 → 未配置: return None
    ↓
4. 检查内容是否为空 → 为空: return None
    ↓
5. ⚠️ 完全不检查全局月度预算
    ↓
6. ⚠️ 完全不检查 Per-Watch 累计预算
    ↓
7. ⚠️ 完全不检查输入字符大小（但内部有 8,000 chars 截断）
    ↓
8. 执行 LLM 调用
    ↓
调用 LLM 后（记账阶段）
    ↓
9. plugin 内部调用 accumulate_global_tokens() → ✓ 累加全局计数
    ↓
10. processor.py 手动写入 watch['llm_last_tokens_used']
    ↓
11. processor.py 手动累加 watch['llm_tokens_used_cumulative']
    ↓
12. ⚠️ 完全不调用 _check_token_budget()
    ↓
13. ⚠️ 完全不检查预算是否超限
```

### 5.6 五条路径统一对照表

| 检查项 | run_setup | summarise_change | evaluate_change | preview_extract | restock_plugin |
|-------|-----------|-----------------|-----------------|-----------------|----------------|
| **调用前检查** | | | | | |
| 全局月度预算 | ✗ 不检查 | ✓ 返回 '' | ✓ important=True | ✗ 不检查 | ✗ 不检查 |
| Per-Watch 累计预算 | ✗ 不检查 | ✗ 不检查 | ✓ important=True | ✗ 不检查 | ✗ 不检查 |
| 输入字符大小限制 | ✗ 不检查 | ✓ 抛异常 | ✓ 抛异常 | ✓ 抛异常 | ⚠️ 内部截断 8k |
| **调用后记账** | | | | | |
| 全局 Token 累计 | ✓ | ✓ | ✓ | ✓ | ✓ (plugin内部) |
| Watch Token 累计 | ✓ (via _check) | ⚠️ **双重累加 Bug** | ✓ (via _check) | ✗ | ✓ (processor手动) |
| llm_last_tokens_used | ✗ 不设置 | ✓ | ✓ | ✗ | ✓ (processor手动) |
| 调用 _check_token_budget | ✓ (返回值忽略) | ✓ (返回值忽略) | ✓ (返回值忽略) | ✗ | ✗ |
| **异常处理** | | | | | |
| 异常处理方式 | 静默失败, prefilter=None | 抛出异常 | 开放失败, important=True | 返回 None | 返回 None |
| **累计阈值提前触发风险** | 中 | **极高** | 高 | 低 | 高 |

## 6. 累计阈值提前触发风险分析

### 6.1 风险来源矩阵

| 风险因素 | 影响路径 | 风险等级 | 说明 |
|---------|---------|---------|------|
| summarise_change 双重累加 | summarise → evaluate | **极高** | 单次调用计数翻倍，阈值提前 50% 触发 |
| run_setup 无前置检查 | run_setup | 中 | setup 调用的 token 直接累计，无 gatekeeping |
| restock plugin 无前置检查 | restock → evaluate | 高 | restock 调用的 token 直接累计，无 gatekeeping |
| preview 不累加 Watch Token | preview | 低 | 仅影响全局预算，不影响 Per-Watch 阈值 |
| _check_token_budget 返回值忽略 | 所有路径 | 高 | 单次超限仅记录日志，不影响后续调用 |

### 6.2 典型风险场景

**场景：同时启用 AI 变更摘要 + 变更评估 + restock fallback**

```
第 1 次检查：
  summarise_change: 实际 400 tokens → 累计显示 800 (×2)
  restock_plugin: 实际 300 tokens → 累计 +300 → 1100
  已超过 max_tokens_cumulative=1000 阈值 ❌

第 2 次检查：
  evaluate_change 预检查发现 1100 > 1000
  → 开放失败 important=True
  → 即使变更不重要也会发送通知
  → 实际仅使用 700 tokens，远未达到阈值
```

**后果**：
1. Per-Watch 预算限制失去意义
2. 通知抑制功能提前失效
3. 用户信任度下降（"为什么没达到限额也发通知？"）

## 7. Prompt 组装流程

### 7.1 变更摘要 Prompt 组装

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

### 7.2 变更摘要为何不使用页面快照上下文

**关键设计决策**：`build_change_summary_prompt()` 接受 `current_snapshot` 参数是为了调用者兼容性，但**故意不使用**该参数。

```python
NOTE: current_snapshot is accepted for caller compatibility but intentionally
unused. A wholesale page excerpt caused the LLM to report unchanged page
content (e.g. old release-note bullets) as "what changed" — hallucinations
drawn from the excerpt rather than the diff. The in-diff context lines give
the model enough surrounding text to describe each change accurately.
```

**原因分析**：

1. **幻觉问题**：完整的页面摘录会导致 LLM 将未变更的内容（如旧的发布说明 bullet）错误地报告为"变更了什么"
2. **上下文已足够**：diff 本身通过 unified diff 的 `n=3` 上下文行（变更前后各 3 行）已经提供了足够的周边文本来准确描述每个变更
3. **准确性优先**：变更摘要的核心是描述 diff 中的实际变化，而非整个页面的状态。提供过多未变更的上下文反而会干扰 LLM 的判断

### 7.3 系统 Prompt 分离设计

系统 Prompt 和用户 Prompt 分离：
- **系统 Prompt**: `build_change_summary_system_prompt()` - 通用 diff 读取规则和准确性要求
- **用户 Prompt**: `build_change_summary_prompt()` - 包含具体内容、URL、标题、用户自定义指令

这种设计允许用户完全自定义输出格式，而不会被硬编码的格式规则覆盖。

### 7.4 Prompt 级联解析

**文件**: `evaluator.py:156-173` - `resolve_llm_field()`

用户自定义 Prompt 的级联解析顺序：
```
1. Watch 级别的配置 → 2. Tag 级别的配置 → 3. 全局默认配置 → 4. 硬编码默认值
```

## 8. Token Budget 裁剪顺序

### 8.1 BM25 相关性裁剪

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

### 8.2 输出 Token 动态调整

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

### 8.3 本地产 Token 乘数

**文件**: `evaluator.py:121-149` - `apply_local_token_multiplier()`

针对自托管 OpenAI 兼容端点（如 vLLM、LM Studio、llama.cpp）的特殊处理：
- **激活条件**: `llm_cfg['provider_kind'] == 'openai_compatible'`
- **默认乘数**: 5x
- **配置范围**: 1-20x (UI 强制范围，代码中也有防御性限制)
- **目的**: 为推理模型（如 Qwen3、DeepSeek-R1、Gemma 3）提供足够的思考空间，避免因 `finish_reason='length'` 导致响应被截断

### 8.4 全局月度 Token 预算

**文件**: `evaluator.py:232-335`

**预算配置优先级**:
1. 环境变量 `LLM_TOKEN_BUDGET_MONTH` (最高优先级)
2. UI 设置中的 token budget
3. 0 = 无限制 (默认)

## 9. 响应解析失败处理

### 9.1 JSON 提取与清理

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

### 9.2 评估响应解析

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

### 9.3 预览响应解析

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

### 9.4 设置响应解析

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

### 9.5 restock 响应解析

**文件**: `llm_restock.py:251-291`

```
JSON 解析
    ↓
成功
    ↓
price 规范化: 转换为 float，异常时设为 None
    ↓
返回 {'price': float|None, 'currency': str|None, 'availability': str|None, '_tokens': int, ...}
    ↓
失败 (JSONDecodeError / Exception)
    ↓
返回 None
```

## 10. 缓存机制

### 10.1 变更摘要缓存

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

### 10.2 评估结果缓存

**文件**: `evaluator.py:654-658`

**缓存 Key**: `SHA256(intent + '||' + diff)`

每个唯一的 (intent, diff) 组合只评估一次，避免重复消耗 Token。

## 11. 测试覆盖分析

### 11.1 测试文件覆盖矩阵

| 测试文件 | 覆盖路径 | 核心测试内容 |
|---------|---------|-------------|
| `tests/llm/test_evaluator.py` | evaluate_change, summarise_change, run_setup | 缓存、token 预算、级联解析、失败开放策略 |
| `tests/test_llm_change_summary.py` | summarise_change | 表单持久化、级联、通知 Token 替换、错误处理、全局默认、AJAX 集成 |
| `tests/test_llm_token_budget.py` | evaluate_change (全局) | 全局月度预算、成本追踪、防篡改、月份翻转 |
| `tests/test_llm_preview.py` | preview_extract | 意图匹配、无意图时不调用 LLM、LLM 失败不破坏预览 |
| `tests/llm/test_llm_restock_plugin.py` | restock_plugin | 基本功能测试（token 记账无测试） |

### 11.2 五条路径的测试覆盖映射

| 代码行为 | 对应测试用例 | 是否覆盖 | 备注 |
|---------|-------------|---------|------|
| **run_setup** | | | |
| 不配置 LLM 时直接 return | 无专门测试 | ✗ | |
| 无 intent 时直接 return | 无专门测试 | ✗ | |
| 调用后执行 token 累计 | 无专门测试 | ✗ | |
| 异常时静默设置 prefilter=None | 无专门测试 | ✗ | |
| **summarise_change** | | | |
| LLM 未配置时返回 '' | `test_returns_empty_when_llm_not_configured` | ✓ | `test_evaluator.py:441-446` |
| 全局预算超限返回 '' | `test_summarise_change_global_budget_exceeded` | ✓ | `test_evaluator.py` |
| 输入字符超限抛出异常 | `test_llm_summary_ajax_surfaces_rate_limit_error` | ✓ | `test_llm_change_summary.py:150-193` |
| 空 diff 不调用 LLM | `test_returns_empty_when_diff_empty` | ✓ | `test_evaluator.py:463-470` |
| 使用默认 prompt | `test_uses_default_prompt_when_no_summary_prompt` | ✓ | `test_evaluator.py:448-461` |
| LLM 调用失败重新抛出 | `test_llm_failure_raises` | ✓ | `test_evaluator.py:493-500` |
| 动态调整 token 上限 | `test_uses_higher_token_limit_than_eval` | ✓ | `test_evaluator.py:502-514` |
| 双重累加 Bug | ❌ 无专门测试 | ✗ | 现有测试可能会误判 |
| 调用前不检查 Per-Watch 累计 | ❌ 无专门测试 | ✗ | |
| **evaluate_change** | | | |
| LLM 未配置返回 None | `test_returns_none_when_llm_not_configured` | ✓ | `test_evaluator.py:163-168` |
| 无 intent 返回 None | `test_returns_none_when_no_intent` | ✓ | `test_evaluator.py:170-175` |
| 缓存命中跳过 LLM 调用 | `test_cache_hit_skips_llm_call` | ✓ | `test_evaluator.py:204-222` |
| 全局预算超限开放失败 | `test_evaluate_change_global_budget_exceeded` | ✓ | `test_evaluator.py` |
| Per-Watch 累计超限开放失败 | `test_evaluate_change_skips_call_when_cumulative_over_budget` | ✓ | `test_evaluator.py:362-375` |
| 调用后单次超限仅记录日志 | `test_evaluate_change_per_check_limit_fails_open` | ✓ | `test_evaluator.py:377-392` |
| LLM 失败开放失败 | `test_llm_failure_returns_important_true` | ✓ | `test_evaluator.py:224-235` |
| last_tokens_used 存储 | `test_last_tokens_used_stored_after_eval` | ✓ | `test_evaluator.py:250-261` |
| 累计 token 累加 | `test_cumulative_tokens_accumulate_across_evals` | ✓ | `test_evaluator.py:263-280` |
| **preview_extract** | | | |
| LLM 未配置返回 None | `test_preview_no_llm_evaluation_when_llm_not_configured` | ✓ | `test_llm_preview.py:183-203` |
| 无 intent 返回 None | `test_preview_no_llm_evaluation_without_intent` | ✓ | `test_llm_preview.py:163-180` |
| 内容为空返回 None | ❌ 无专门测试 | ✗ | |
| LLM 失败不破坏预览 | `test_preview_llm_failure_does_not_break_preview` | ✓ | `test_llm_preview.py:210-232` |
| 完全不累加 Watch Token | ❌ 无专门测试 | ✗ | 设计如此，不检查 |
| 不检查全局月度预算 | ❌ 无专门测试 | ✗ | |
| **restock_plugin** | | | |
| 基本功能测试 | `test_llm_restock_plugin_basic` | ✓ | |
| token 记账正确性 | ❌ 无专门测试 | ✗ | |
| 与 evaluator 计数一致性 | ❌ 无专门测试 | ✗ | |
| 全局预算检查 | ❌ 无专门测试 | ✗ | |

### 11.3 测试缺口总结

1. **run_setup 路径**：完全无覆盖（0 测试）
2. **summarise_change 双重累加 Bug**：无专门测试验证累计值正确性
3. **summarise_change 调用前 Per-Watch 检查**：未验证是否跳过检查
4. **preview_extract 全局预算检查**：未验证是否跳过检查
5. **restock_plugin 计数链路**：无专门测试验证 token 记账
6. **五条路径一致性**：无测试直接对比五条路径的行为差异

## 12. 总结

### 12.1 上下文体积控制要点

1. **多层级限制**: 从全局 100k 字符到各场景的精细限制
2. **智能裁剪**: BM25 相关性优先裁剪，保留语义完整性
3. **动态 Token 分配**: 根据 diff 大小动态调整输出 token 预算
4. **本地模型适配**: 为自托管模型提供额外的 Token 乘数
5. **预算保护**: 全局月度预算 + Per-Watch 双重保护（但实际效果因路径而异）

### 12.2 Token Budget 关键发现

1. **`_check_token_budget` 从不抛异常**: 仅返回 bool，大多数场景下返回值被忽略
2. **五条路径行为高度不一致**:
   - run_setup：完全不进行任何预检查
   - summarise_change：调用前只检查全局预算，存在双重累加 Bug
   - evaluate_change：调用前检查全局和累计预算，采用开放失败策略
   - preview_extract：完全不检查 Per-Watch 预算和全局预算，不累加 Watch Token
   - restock_plugin：完全不进行任何预算检查，processor 手动记账
3. **预算超限的实际效果有限**: 多数情况下仅记录日志，不阻止调用或拦截结果
4. **⚠️ 双重累加 Bug**: `summarise_change` 中 token 被重复累加，导致累计值是实际使用量的 2 倍
5. **⚠️ 累计阈值提前触发风险**: 多种路径（restock、run_setup、summarise）的 token 未经检查直接累计，导致 evaluate_change 的预检查阈值被提前触发

### 12.3 变更摘要不使用快照上下文的核心原因

| 问题 | 影响 | 解决方案 |
|-----|------|---------|
| LLM 幻觉 | 未变更的页面内容被错误报告为变更 | 仅提供 diff，不附加完整页面快照 |
| 上下文冗余 | diff 本身已包含 n=3 上下文行 | 依赖 diff 内置上下文而非额外快照 |
| 准确性干扰 | 未变更内容会分散 LLM 注意力 | 聚焦于实际变更行，减少噪声 |

### 12.4 失败处理原则

1. **JSON 解析失败**: 保守默认值（不触发通知）
2. **LLM 调用失败**: 开放失败（触发通知，不遗漏变更）- evaluate_change 路径
3. **LLM 调用失败**: 抛出异常 - summarise_change 路径
4. **全局预算超限**: 变更摘要停止，变更评估开放失败
5. **输入过大**: 明确错误提示，不进行部分处理
6. **Per-Watch 预算超限**: 多数情况下仅记录日志，不影响功能
7. **restock 失败**: 静默返回 None，不影响主流程

### 12.5 关键设计决策与 Bug

| 决策 / Bug | 原因 / 影响 |
|-----------|------------|
| BM25 裁剪而非简单截断 | 保留与 intent 相关的内容，上下文质量更高 |
| 系统/用户 Prompt 分离 | 用户可完全自定义输出格式，灵活性高 |
| 移动行预标记 (~前缀) | 避免 LLM 误判重排序为变更，减少误报 |
| 调用失败时开放失败 (evaluate) | 不因为 LLM 故障错过重要变更，保证不遗漏 |
| 多级 Prompt 级联 | Watch > Tag > Global > 硬编码默认 |
| 变更摘要不使用快照上下文 | 防止 LLM 将未变更内容报告为变更，提高准确性 |
| _check_token_budget 返回值被忽略 | 预算仅作参考/监控，不阻断核心功能 |
| preview 不累加 Watch Token | 预览操作不计入用户预算，降低门槛 |
| restock plugin 独立记账 | 模块化设计，但缺乏统一预算检查 |
| **summarise_change 双重累加 Bug** | **Token 使用量被翻倍计算，累计阈值提前 50% 触发** |
