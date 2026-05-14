# 网页补货状态识别处理器 (Restock Processor) 工作原理

## 一、处理器概述

Restock Processor 是一个专门用于监控电商网站商品库存状态和价格变化的处理器。它能够自动检测商品是否从"缺货"变为"有货"，以及跟踪价格变动。

- **位置**: `changedetectionio/processors/restock_diff/`
- **核心文件**:
  - `processor.py` - 核心处理器逻辑
  - `__init__.py` - Restock 数据结构定义
  - `forms.py` - 配置表单
  - `pure_python_extractor.py` - 纯 Python 元数据提取器
  - `plugins/llm_restock.py` - LLM 插件支持

## 二、依赖输入

### 2.1 网页内容输入

| 输入类型 | 来源 | 描述 |
|---------|------|------|
| **HTML 原始内容** | `self.fetcher.content` | 从网站获取的完整 HTML 页面 |
| **HTTP 状态码** | `self.fetcher.get_last_status_code()` | 响应状态码，用于验证请求成功 |
| **截图 (可选)** | `self.fetcher.screenshot` | Playwright/Puppeteer 抓取的页面截图 |
| **JavaScript 库存数据** | `self.fetcher.instock_data` | 浏览器脚本 `stock-not-in-stock.js` 的分析结果 |
| **XPath 数据** | `self.fetcher.xpath_data` | 提取的 XPath 元素数据 |

### 2.2 配置输入

#### 2.2.1 Watch 级别配置 (`restock_diff.json`)

```python
{
    'follow_price_changes': True,           # 是否跟踪价格变化
    'in_stock_processing': 'in_stock_only', # 库存检测模式
    'price_change_min': None,               # 最低价格阈值
    'price_change_max': None,               # 最高价格阈值
    'price_change_threshold_percent': None  # 价格变化百分比阈值
}
```

#### 2.2.2 库存检测模式

| 模式值 | 行为 |
|-------|------|
| `in_stock_only` | 仅在"缺货 → 有货"时触发（默认） |
| `all_changes` | 任何库存状态变化都触发 |
| `off` | 关闭库存检测 |

#### 2.2.3 配置继承链

配置可以从多个来源获取，优先级从高到低：

1. **Watch 自身配置** (watch 级别的 restock_diff.json)
2. **Tag 覆盖配置** (watch 关联的 tag 中 `overrides_watch=True`)
3. **系统默认配置**

```python
# processor.py:454-467
_extra_config = self.get_extra_watch_config('restock_diff.json')
restock_settings = _extra_config.get('restock_diff') or {
    'follow_price_changes': True,
    'in_stock_processing': 'in_stock_only',
}

# Check if any tags have override enabled
for tag_uuid in watch.get('tags'):
    tag = self.datastore.data['settings']['application']['tags'].get(tag_uuid, {})
    if tag.get('overrides_watch'):
        restock_settings = tag.get('processor_config_restock_diff') or {}
        break
```

## 三、判定规则

### 3.1 库存状态判定

### 3.1.1 基于 Schema.org 元数据的判定

处理器首先从页面元数据中提取 `availability` 字段，并进行标准化处理：

```python
# processor.py:553-562
if any(substring.lower() in itemprop_availability['availability'].lower() for substring in [
    'instock',
    'instoreonly',
    'limitedavailability',
    'onlineonly',
    'presale']
   ):
    update_obj['restock']['in_stock'] = True
else:
    update_obj['restock']['in_stock'] = False
```

#### 标准化处理

原始 availability 值会经过 URL 前缀去除处理：

```python
# processor.py:367-369
value['availability'] = re.sub(r'(?i)^(https|http)://schema.org/', '',
                               value.get('availability').strip(' "\'').lower())
```

**示例转换**:
- `https://schema.org/InStock` → `instock`
- `InStock` → `instock`

### 3.1.2 基于浏览器 JavaScript 的判定

当元数据不可用时，使用浏览器脚本 `stock-not-in-stock.js` 分析页面文本：

```python
# processor.py:583-587
if self.fetcher.instock_data and itemprop_availability.get('availability') is None:
    update_obj['restock']["in_stock"] = True if self.fetcher.instock_data == 'Possibly in stock' else False
```

**JS 分析结果**:
- `Possibly in stock` → 判定为有货
- 其他结果 → 判定为缺货

### 3.1.3 信任优先级规则

处理器实现了"防欺骗"机制：当浏览器脚本判定为缺货时，即使元数据显示有货，也以浏览器判定为准。

```python
# processor.py:589-595
if self.fetcher.instock_data and self.fetcher.instock_data != 'Possibly in stock':
    if update_obj['restock'].get('in_stock'):
        # Override: trust browser scraping over metadata
        update_obj['restock']["in_stock"] = False
```

**判定优先级**:
1. **浏览器脚本判定缺货** → 最终判定缺货（最高优先级，防止元数据欺骗）
2. **元数据判定** → 作为基础判定
3. **浏览器脚本判定可能有货** → 补充判定

### 3.2 价格变化判定

### 3.2.1 基础价格比较

```python
# processor.py:618-625
if restock_settings.get('follow_price_changes') and watch.get('restock') and update_obj.get('restock') and update_obj['restock'].get('price'):
    price = float(update_obj['restock'].get('price'))
    if watch['restock'].get('original_price'):
        previous_price = float(watch['restock'].get('original_price'))
        if price != previous_price:
            changed_detected = True
```

### 3.2.2 价格区间过滤

使用 `is_between()` 函数判断价格是否在允许区间内：

```python
# processor.py:388-400
def is_between(number, lower=None, upper=None):
    return (lower is None or lower <= number) and (upper is None or number <= upper)
```

**区间规则** (`processor.py:627-641`):
- 当价格在 `[min, max]` 区间内时 → **不触发**变更
- 当价格超出区间时 → 可能触发变更

**示例**:
- 设置 `price_change_min=900`, `price_change_max=1100`
- 价格变为 1000 → 在区间内，不触发
- 价格变为 850 → 低于下限，触发
- 价格变为 1200 → 高于上限，触发

### 3.2.3 价格变化百分比阈值

```python
# processor.py:646-654
if watch['restock'].get('original_price') and changed_detected and restock_settings.get('price_change_threshold_percent'):
    previous_price = float(watch['restock'].get('original_price'))
    pc = float(restock_settings.get('price_change_threshold_percent'))
    change = abs((price - previous_price) / previous_price * 100)
    if change and change <= pc:
        # 变化幅度未超过阈值，不触发
        changed_detected = False
```

**示例**:
- 原价：$1,000
- 阈值：2%
- 现价：$1,015 → 变化 1.5% ≤ 2% → **不触发**
- 现价：$1,030 → 变化 3% > 2% → **触发**

### 3.3 多价格检测规则

Restock 处理器专为**单个商品页面**设计，不支持多商品页面：

```python
# processor.py:347-355
price_result = _deduplicate_prices(price_parse.find(data))
if price_result:
    if len(price_result) > 1:
        # 检查所有价格是否只是格式不同（如 $121.95 vs 121.95）
        # 如果确实是多个不同价格，抛出异常
        raise MoreThanOnePriceFound()
    value['price'] = price_result[0]
```

**异常处理**:
- 当检测到多个价格且插件也无法解析时 → 抛出 `ProcessorException`
- 提示用户使用普通的 content-change detection 模式

## 四、元数据提取机制

### 4.1 提取策略

处理器采用**双层提取策略**，优先使用快速的纯 Python 提取，失败后回退到完整的 extruct 库。

#### 层次 1: 纯 Python 提取器 (Pure Python Extractor)

文件: `pure_python_extractor.py`

**支持的格式**:
1. **JSON-LD** - `<script type="application/ld+json">` 标签
2. **OpenGraph** - `<meta property="og:*">` 标签
3. **Microdata** - `itemprop` 属性

**优点**:
- 不依赖 lxml，无内存泄漏问题
- 速度快，覆盖 80%+ 现代网站

#### 层次 2: Extruct 回退 (Fallback)

文件: `processor.py:315-385`

使用 `extruct` 库提取多种格式:
- JSON-LD
- Microdata
- OpenGraph
- RDFa
- Dublin Core

**适用场景**:
- 纯 Python 提取失败或不完整时
- 复杂的嵌套结构

### 4.2 Linux 内存管理优化

在 Linux 平台上，extruct 提取在独立子进程中运行，以避免 lxml 的 C 级内存泄漏。

```python
# processor.py:249-310
if platform.system() == 'Linux':
    # 使用 spawn 方式创建子进程
    ctx = multiprocessing.get_context('spawn')
    parent_conn, child_conn = ctx.Pipe()
    p = ctx.Process(target=_extract_itemprop_availability_worker, args=(child_conn,))
    p.start()
    # 通过管道传递 HTML 数据，避免 pickle 问题
    parent_conn.send_bytes(html_bytes)
    # 接收结果
    result_bytes = parent_conn.recv_bytes()
    result = json.loads(result_bytes.decode('utf-8'))
    p.join()
```

**内存问题背景** (`processor.py:60-128`):
- extruct 内部使用 lxml，lxml 依赖 libxml2 (C 库)
- Python GC 无法释放 C 级内存
- 多次解析大 HTML 导致内存持续增长
- 子进程退出时 OS 强制回收所有内存

### 4.3 插件扩展机制

当内置提取器无法获取完整数据时，可以使用插件（如 LLM 插件）增强提取能力。

```python
# processor.py:489-534
if not (has_price and has_availability):
    # 尝试插件提取
    plugin_availability = get_itemprop_availability_from_plugin(
        self.fetcher.content,
        fetcher_name,
        self.fetcher,
        watch.link,
        llm_intent=_llm_intent or None
    )
    
    if plugin_availability:
        # 使用插件提供的数据
        itemprop_availability = plugin_availability
```

**插件能力示例**:
- `llm_restock.py` - 使用大语言模型解析复杂页面
- 可根据不同 fetcher (html_requests, html_webdriver) 选择不同插件

## 五、决策流程

### 5.1 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        执行周期开始                                  │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  1. 检查内容是否变化 (Checksum 校验)                                  │
│     ──────────────────────────────────────────────────────────      │
│     • 计算当前 raw HTML 的 MD5 checksum                              │
│     • 与上次 checksum 比较                                           │
│     • 相同且 watch 未被编辑 → 跳过 (checksumFromPreviousCheckWasTheSame)│
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  2. 加载配置                                                         │
│     ──────────────────────────────────────────────────────────      │
│     • 读取 restock_diff.json                                         │
│     • 检查是否有 tag override                                        │
│     • 确定最终的 restock_settings                                    │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  3. 提取商品元数据                                                    │
│     ──────────────────────────────────────────────────────────      │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 3.1 纯 Python 提取 (优先)                              │        │
│     │     • JSON-LD                                         │        │
│     │     • OpenGraph                                       │        │
│     │     • Microdata                                       │        │
│     │     • 成功且完整 (价格+库存状态) → 完成提取              │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼ (失败/不完整)                    │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 3.2 Extruct 回退                                      │        │
│     │     • Linux: 子进程隔离执行 (防内存泄漏)                │        │
│     │     • 其他: 直接执行                                   │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼ (失败/不完整)                    │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 3.3 插件回退 (如 LLM)                                  │        │
│     │     • 根据 fetcher 类型选择插件                        │        │
│     │     • LLM 可解析复杂页面结构                           │        │
│     └──────────────────────────────────────────────────────┘        │
│                                                                      │
│     • 检测到多价格? → MoreThanOnePriceFound                          │
│     • 无任何提取结果 → ProcessorException                            │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  4. 判定库存状态                                                      │
│     ──────────────────────────────────────────────────────────      │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 4.1 基于元数据的判定                                   │        │
│     │     • 检查 availability 字段                           │        │
│     │     • 匹配关键字: instock, instoreonly,                │        │
│     │       limitedavailability, onlineonly, presale        │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼ (元数据不可用时)                 │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 4.2 基于浏览器 JS 的判定                               │        │
│     │     • 使用 stock-not-in-stock.js 分析                 │        │
│     │     • "Possibly in stock" → 有货                      │        │
│     │     • 其他 → 缺货                                      │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼                                  │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 4.3 信任优先级修正 (防欺骗)                             │        │
│     │     • JS 判定缺货 > 元数据判定有货                      │        │
│     │     • 防止网站元数据撒谎                                │        │
│     └──────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  5. 变更判定 (是否需要通知)                                           │
│     ──────────────────────────────────────────────────────────      │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 5.1 库存状态变更检查                                    │        │
│     │     • in_stock_only: 仅 缺货→有货 触发                 │        │
│     │     • all_changes: 任何状态变更都触发                   │        │
│     │     • off: 不检查                                      │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼                                  │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 5.2 价格变更检查 (如启用 follow_price_changes)          │        │
│     │     • 当前价格 ≠ 原始价格 → 标记可能变更                │        │
│     │     • 检查价格区间 [min, max] → 在区间内则取消变更      │        │
│     │     • 检查变化百分比阈值 → 未超阈值则取消变更            │        │
│     └──────────────────────────────────────────────────────┘        │
│                                                                      │
│     changed_detected = (库存变更符合条件) OR (价格变更符合条件)        │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  6. 保存结果并返回                                                    │
│     ──────────────────────────────────────────────────────────      │
│     • 生成 snapshot: "In Stock: True - Price: 10.99"                │
│     • 计算 snapshot MD5: 用作本次检查的 previous_md5                │
│     • 构建 update_obj: restock 数据 + 状态信息                       │
│     • 返回 (changed_detected, update_obj, snapshot_content)         │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 核心判定函数 `run_changedetection()`

位置: `processor.py:403-659`

#### 函数签名

```python
def run_changedetection(self, watch, force_reprocess=False):
    """
    执行补货状态检测
    
    Args:
        watch: Watch 配置对象
        force_reprocess: 强制重新处理 (忽略 checksum)
    
    Returns:
        tuple: (changed_detected, update_obj, snapshot_content)
            - changed_detected: 是否检测到变更
            - update_obj: 需要保存到 watch 的数据更新
            - snapshot_content: 本次检查的快照内容
    """
```

#### 返回值说明

| 返回值 | 类型 | 说明 |
|-------|------|------|
| `changed_detected` | `bool` | 是否应该触发通知 |
| `update_obj` | `dict` | 需要更新的 watch 数据 |
| `snapshot_content` | `str` | 人类可读的快照文本 |

## 六、触发链路

### 6.1 完整触发链

```
用户/定时任务触发
       │
       ▼
┌──────────────────┐
│   worker.py      │
│  任务调度器       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ worker.py:156-175│
│ 1. 根据 watch.processor 加载处理器模块  │
│ 2. 创建 perform_site_check 实例        │
│ 3. 插件钩子: apply_update_handler_alter │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│ base.py:difference_detection_processor   │
│ ──────────────────────────────────────── │
│ 1. 初始化 fetcher (html_requests/webdriver) │
│ 2. 调用 call_browser() → 获取网页内容      │
│ 3. 内容后处理 (编码转换、UTF-8 修复)        │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│ restock_diff/processor.py:403-659         │
│ ──────────────────────────────────────── │
│ run_changedetection() 核心判定逻辑        │
│  → 提取元数据                              │
│  → 判定库存状态                            │
│  → 判定价格变化                            │
│  → 生成 changed_detected                  │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│ worker.py: 结果处理                       │
│ ──────────────────────────────────────── │
│ 1. changed_detected=True?                │
│    → process_changedetection_results     │
│ 2. 保存 update_obj 到 watch              │
│ 3. 保存 snapshot_content 到历史           │
│ 4. 触发通知 (如需要)                      │
└──────────────────────────────────────────┘
```

### 6.2 触发条件

#### 6.2.1 库存状态变更触发

| 配置 | 前状态 | 后状态 | 是否触发 |
|-----|--------|--------|---------|
| `in_stock_only` | 缺货 | 有货 | **是** |
| `in_stock_only` | 有货 | 缺货 | 否 |
| `in_stock_only` | 有货 | 有货 | 否 |
| `in_stock_only` | 缺货 | 缺货 | 否 |
| `all_changes` | 缺货 | 有货 | **是** |
| `all_changes` | 有货 | 缺货 | **是** |
| `all_changes` | 有货 | 有货 | 否 |
| `all_changes` | 缺货 | 缺货 | 否 |

#### 6.2.2 价格变更触发 (需启用 `follow_price_changes`)

| 条件 | 是否触发 |
|-----|---------|
| 价格 ≠ 原始价格，且超出区间 [min, max] | **是** |
| 价格 ≠ 原始价格，但在区间内 | 否 |
| 价格 ≠ 原始价格，但变化百分比 ≤ 阈值 | 否 |
| 价格 ≠ 原始价格，且变化百分比 > 阈值 | **是** |

### 6.3 跳过条件

#### 6.3.1 内容未变化时的跳过

```python
# processor.py:413-423
current_raw_document_checksum = self.get_raw_document_checksum()

# 跳过条件：
# 1. force_reprocess=False (非强制)
# 2. watch.was_edited=False (配置未修改)
# 3. 上次 checksum 存在
# 4. 两次 checksum 相同
if (not force_reprocess and
    not watch.was_edited and
    self.last_raw_content_checksum and
    self.last_raw_content_checksum == current_raw_document_checksum):
    raise checksumFromPreviousCheckWasTheSame()
```

#### 6.3.2 无法提取数据时的跳过

当页面既没有元数据，也没有 JS 库存数据时：

```python
# processor.py:572-579
if not self.fetcher.instock_data and not itemprop_availability.get('availability') and not itemprop_availability.get('price'):
    raise ProcessorException(
        message=f"Unable to extract restock data for this page unfortunately. (Got code {self.fetcher.get_last_status_code()} from server), no embedded stock information was found and nothing interesting in the text, try using this watch with Chrome.",
        url=watch.get('url'),
        status_code=self.fetcher.get_last_status_code(),
        screenshot=self.fetcher.screenshot,
        xpath_data=self.fetcher.xpath_data
    )
```

## 七、数据结构

### 7.1 Restock 对象

位置: `restock_diff/__init__.py:17-67`

```python
class Restock(dict):
    """
    补货/价格状态数据结构
    
    默认值:
        'in_stock': None      # 布尔值或 None
        'price': None         # 价格 (float)
        'currency': None      # 货币 (str)
        'original_price': None # 首次检测到的价格
    """
```

#### 价格字符串解析

```python
def parse_currency(self, raw_value: str) -> Union[float, None]:
    """
    解析各种格式的价格字符串
    
    处理示例:
        "$1,234.56" → 1234.56
        "1.234,56"  → 1234.56  (欧洲格式)
        "1234"      → 1234.0
    """
```

### 7.2 snapshot 快照格式

```python
# processor.py:598-599
price = update_obj.get('restock').get('price') if update_obj.get('restock').get('price') else ""
snapshot_content = f"In Stock: {update_obj.get('restock').get('in_stock')} - Price: {price}"
```

**示例快照**:
- `In Stock: True - Price: 10.99`
- `In Stock: False - Price: 10.99`
- `In Stock: True - Price: ` (价格未知)

### 7.3 历史价格提取

```python
# restock_diff/__init__.py:69-77
_price_re = re.compile(r"Price:\s*(\d+(?:\.\d+)?)", re.IGNORECASE)

def get_price_from_history_str(history_str):
    """
    从历史快照字符串中提取价格
    
    输入: "In Stock: True - Price: 10.99"
    输出: "10.99" (Decimal 字符串)
    """
```

## 八、异常处理

### 8.1 自定义异常

| 异常类型 | 触发场景 | 处理方式 |
|---------|---------|---------|
| `MoreThanOnePriceFound` | 页面检测到多个价格 | 尝试插件解析，失败则抛出 `ProcessorException` |
| `ProcessorException` | 无法提取数据/多价格等 | 保存截图和 XPath 数据，记录错误，停止本次处理 |
| `checksumFromPreviousCheckWasTheSame` | 内容未变化 | 跳过处理，不生成通知 |

### 8.2 异常处理流程

```python
# worker.py:182-249
except ProcessorException as e:
    # 保存截图
    if e.screenshot:
        watch.save_screenshot(screenshot=e.screenshot)
    # 保存 XPath 数据
    if e.xpath_data:
        watch.save_xpath_data(data=e.xpath_data)
    # 记录错误信息
    datastore.update_watch(uuid=uuid, update_obj={'last_error': e.message})
    # 不处理本次结果
    process_changedetection_results = False
```

## 九、配置示例

### 9.1 典型配置

#### 场景 1: 仅监控补货

```python
{
    'follow_price_changes': False,
    'in_stock_processing': 'in_stock_only',
    'price_change_min': None,
    'price_change_max': None,
    'price_change_threshold_percent': None
}
```

**行为**: 商品从缺货变为有货时通知，不关心价格。

#### 场景 2: 补货 + 降价监控

```python
{
    'follow_price_changes': True,
    'in_stock_processing': 'in_stock_only',
    'price_change_max': 900.0,  # 价格降到 900 以下时通知
    'price_change_threshold_percent': 5
}
```

**行为**:
- 补货时通知
- 价格超过 5% 的下降且低于 900 时通知

#### 场景 3: 全状态监控

```python
{
    'follow_price_changes': True,
    'in_stock_processing': 'all_changes',
    'price_change_threshold_percent': 2
}
```

**行为**:
- 任何库存变化都通知 (有货→缺货也通知)
- 价格变化超过 2% 时通知

## 十、关键代码位置

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 主判定函数 | `restock_diff/processor.py` | 403-659 |
| 库存状态判定 | `restock_diff/processor.py` | 553-562 |
| 价格变化判定 | `restock_diff/processor.py` | 618-654 |
| 纯 Python 提取器 | `restock_diff/pure_python_extractor.py` | 1-289 |
| 元数据提取 (extruct) | `restock_diff/processor.py` | 315-385 |
| 子进程内存管理 | `restock_diff/processor.py` | 249-310 |
| 配置表单 | `restock_diff/forms.py` | 1-82 |
| Restock 数据结构 | `restock_diff/__init__.py` | 17-67 |
| 任务触发入口 | `worker.py` | 156-175 |
| 基础处理器 | `processors/base.py` | 200-365 |

---

**文档版本**: 1.0
**基于代码版本**: 当前仓库版本
**生成日期**: 2026-05-14
