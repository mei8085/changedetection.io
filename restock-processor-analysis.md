# Restock Processor 分析报告

## 概述

`restock_diff` 是 changedetection.io 项目中的补货检测处理器，专门用于监控电商产品页面的库存状态和价格变化。当产品从"缺货"变为"有货"时，或者价格发生变化时，触发通知。

**定位**: 适用于单个产品页面，检测产品补货和价格变动场景。

---

## 核心架构

### 文件结构

```
changedetectionio/processors/restock_diff/
├── __init__.py              # Restock 数据模型定义
├── processor.py             # 主处理器逻辑
├── forms.py                 # 配置表单定义
├── api.yaml                 # API 规范扩展
├── pure_python_extractor.py # 纯 Python 元数据提取器
└── plugins/
    └── llm_restock.py       # LLM 回退提取插件
```

### 类继承关系

```
difference_detection_processor (base.py)
    └── perform_site_check (processor.py)
```

---

## 数据提取策略

### 三层提取架构

处理器采用**渐进式三层提取策略**，优先使用轻量级方法，必要时回退到重量级方案：

#### 第一层: 纯 Python 提取器 (pure_python_extractor.py)

**优点**: 无内存泄漏、无外部依赖（C 扩展）

**支持的格式**:
1. **JSON-LD** - 最可靠的结构化数据格式
2. **OpenGraph** meta 标签
3. **Microdata** 属性

**工作流程**:
- 使用 Python 内置 `html.parser` 解析 HTML
- 通过正则和字符串匹配提取元数据
- 使用 `jsonpath_ng` 查询提取的数据

#### 第二层: extruct 提取器 (get_itemprop_availability)

**触发条件**: 纯 Python 提取器未能获取完整的 price + availability 数据

**支持格式**:
- dublincore
- json-ld
- microdata
- microformat
- opengraph

**内存管理策略 (Linux)**:
- 使用 `spawn` 多进程模式
- 通过 Pipe 传递 HTML 字节数据
- 子进程退出时 OS 强制回收所有内存（包括 lxml C 级分配）

#### 第三层: LLM 回退插件 (llm_restock.py)

**触发条件**: 内置提取器无法获取有效的 price 和 availability 数据

**处理流程**:
1. 移除网站 chrome（导航、页脚等）
2. 提取 JSON-LD 数据块
3. 清理 HTML，转换为纯文本
4. 发送给 LLM 生成结构化 JSON

**LLM 返回格式**:
```json
{
  "price": 29.99,
  "currency": "USD",
  "availability": "instock"
}
```

---

## 核心数据结构

### Restock 类 (__init__.py)

```python
class Restock(dict):
    default_values = {
        'in_stock': None,      # 库存状态
        'price': None,         # 当前价格
        'currency': None,      # 货币代码
        'original_price': None # 首次检测时的价格
    }
```

**特殊处理**:
- `price` 和 `original_price` 设置时自动调用 `parse_currency()` 解析
- 支持多种货币格式（1,400.00 → 1400.00）

### parse_currency() 货币解析逻辑

```python
def parse_currency(self, raw_value: str) -> Union[float, None]:
    # 1. 处理千分位和小数位混淆的情况
    # 2. 移除非数字字符（保留小数点和负号）
    # 3. 使用 babel.parse_decimal 解析
```

---

## 变更检测逻辑 (run_changedetection)

### 检测触发条件

#### 1. 库存状态变化

```python
if watch['restock']['in_stock'] != update_obj['restock']['in_stock']:
    # 缺货 → 有货 且配置为 "in_stock_only"
    if restock_settings['in_stock_processing'] == 'in_stock_only' and new_in_stock:
        changed_detected = True
    # 或配置为 "all_changes"
    if restock_settings['in_stock_processing'] == 'all_changes':
        changed_detected = True
```

#### 2. 价格变化 (follow_price_changes)

```python
# 比较当前价格与原始价格
if current_price != original_price:
    changed_detected = True
```

**可选阈值限制**:
- `price_change_min`: 价格低于此值才触发
- `price_change_max`: 价格高于此值才触发
- `price_change_threshold_percent`: 价格变化百分比阈值

### 元数据可用性检测

```python
# 支持的库存状态关键词
in_stock_keywords = [
    'instock', 'instoreonly', 'limitedavailability',
    'onlineonly', 'presale'
]
```

### 谎言检测机制

```python
# 当网页内容明确显示缺货时，覆盖元数据的"有货"声明
if scraper.says_NOT_in_stock_but_metadata.says_in_stock:
    update_obj['restock']['in_stock'] = False
```

---

## 配置选项 (forms.py)

### RestockSettingsForm

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `in_stock_processing` | Radio | `in_stock_only`(仅缺货→有货) / `all_changes`(所有变化) / `off`(关闭) |
| `follow_price_changes` | Boolean | 是否跟踪价格变化 |
| `price_change_min` | Float | 最低价格阈值 |
| `price_change_max` | Float | 最高价格阈值 |
| `price_change_threshold_percent` | Float | 价格变化百分比阈值 (0-100) |

### 标签覆盖机制

支持通过标签（Tag）覆盖单个监控的配置：
```python
if tag.get('overrides_watch'):
    restock_settings = tag.get('processor_config_restock_diff')
```

---

## 内存管理策略

### lxml/extruct 内存泄漏问题

**问题根源**:
1. lxml 使用 libxml2 C 库分配内存
2. Python GC 无法释放 C 级内存分配
3. 大量 HTML 解析后内存持续增长

**解决方案: 多进程隔离 (Linux)**

```python
# 使用 spawn 创建子进程
ctx = multiprocessing.get_context('spawn')
p = ctx.Process(target=_extract_itemprop_availability_worker, args=(child_conn,))

# 子进程退出时 OS 强制回收所有内存
p.join()
```

### 纯 Python 提取器优化

未来可能的优化方向：
1. 使用正则提取 JSON-LD 块（避免解析整个 HTML）
2. 减少 lxml 的使用场景
3. 仅解析元数据相关的小部分 HTML

---

## 通知令牌 (Notification Tokens)

### 可用令牌

| 令牌 | 说明 |
|------|------|
| `restock.price` | 当前价格 |
| `restock.in_stock` | 库存状态 |
| `restock.original_price` | 首次检测价格 |
| `restock.previous_price` | 上次历史价格 |

### extra_notification_token_values() 实现

```python
def extra_notification_token_values(self):
    values = super().extra_notification_token_values()
    values['restock'] = self.get('restock', {})
    # 从历史快照中获取上一次价格
    if history_n >= 2:
        values['restock']['previous_price'] = get_price_from_history_str(
            self.get_history_snapshot(timestamp=sorted_keys[-1])
        )
    return values
```

---

## 快照内容格式

```python
snapshot_content = f"In Stock: {in_stock} - Price: {price}"
# 示例: "In Stock: True - Price: 29.99"
```

快照内容用于生成 MD5 校验和，作为变更检测的依据。

---

## 性能考量

### 1. 校验跳过机制

```python
# 仅当 HTML 内容未变化且配置未修改时跳过处理
if (not force_reprocess and
    not watch.was_edited and
    last_raw_content_checksum == current_raw_document_checksum):
    raise checksumFromPreviousCheckWasTheSame()
```

### 2. 提取优先级

| 层级 | 方法 | 性能 | 覆盖率 |
|------|------|------|--------|
| 1 | 纯 Python | 最快 | 80%+ |
| 2 | extruct | 较慢 | 90%+ |
| 3 | LLM | 最慢 | ~100% |

### 3. Linux 多进程开销

- 每次 extruct 提取约 35MB 额外开销
- 但避免了 500MB+ 的内存泄漏累积

---

## 错误处理

### 异常类型

| 异常 | 说明 | 处理方式 |
|------|------|----------|
| `UnableToExtractRestockData` | 无法提取数据 | 记录状态码 |
| `MoreThanOnePriceFound` | 发现多个价格 | 仅支持单产品页面 |
| `checksumFromPreviousCheckWasTheSame` | 内容未变化 | 跳过处理 |

### 错误恢复策略

```python
# 多个价格时的处理流程
1. 记录警告
2. 尝试插件提取
3. 插件也失败 → 抛出 ProcessorException
```

---

## 测试覆盖

### 单元测试
- `test_restock_logic.py` - 核心逻辑测试（is_between 函数）

### 集成测试
- `test_restock.py` - 完整补货检测流程测试
- `test_restock_itemprop.py` - itemprop 提取测试
- `test_llm_restock_plugin.py` - LLM 插件测试

---

## API 扩展 (api.yaml)

处理器通过 `api.yaml` 扩展了 Watch 和 Tag 的 API schema：

```yaml
processor_config_restock_diff:
  in_stock_processing: string (enum)
  follow_price_changes: boolean
  price_change_min: number
  price_change_max: number
  price_change_threshold_percent: number (0-100)
```

---

## 使用场景

### 适用场景
- 电商产品页面监控
- 限量商品补货提醒
- 价格波动跟踪
- 多个商品同时监控

### 不适用场景
- 分类页面/搜索结果页
- 多个产品混合页面
- 需要登录才能看到库存的页面
- SPA 动态渲染页面（建议配合 html_webdriver 使用）

---

## 技术债务与未来优化

1. **正则提取替代 lxml**: 避免 C 级内存泄漏
2. **locale 感知的价格解析**: 当前硬编码 `locale='en'`
3. **更细粒度的价格阈值**: 支持百分比和绝对值的组合
4. **历史价格图表**: 可视化价格走势
5. **多产品支持**: 扩展为监控页面上的多个产品

---

## 总结

restock_diff 处理器是一个设计良好的电商监控解决方案，具有：

1. **健壮的三层提取架构** - 平衡性能和覆盖率
2. **完善的内存管理** - 特别是 Linux 下的多进程隔离
3. **灵活的配置选项** - 支持细粒度的触发控制
4. **智能的谎言检测** - 优先信任网页内容而非元数据
5. **完整的错误处理** - 多层次的重试和回退机制
