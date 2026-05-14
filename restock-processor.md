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

#### 2.2.3 配置覆盖优先级（重要）

配置加载采用**后加载覆盖**策略，优先级从高到低：

1. **Tag 覆盖配置** (watch 关联的 tag 中 `overrides_watch=True`)
2. **Watch 自身配置** (watch 级别的 restock_diff.json)
3. **系统默认配置**

```python
# processor.py:455-467
# 步骤 1: 先加载 watch 自身配置
_extra_config = self.get_extra_watch_config('restock_diff.json')
restock_settings = _extra_config.get('restock_diff') or {
    'follow_price_changes': True,
    'in_stock_processing': 'in_stock_only',
}

# 步骤 2: 遍历 tags，如果找到启用了 overrides_watch 的 tag，直接覆盖 restock_settings
for tag_uuid in watch.get('tags'):
    tag = self.datastore.data['settings']['application']['tags'].get(tag_uuid, {})
    if tag.get('overrides_watch'):
        restock_settings = tag.get('processor_config_restock_diff') or {}  # ← 直接覆盖！
        logger.info(f"Watch {watch.get('uuid')} - Tag '{tag.get('title')}' selected for restock settings override")
        break
```

**关键点**: Tag 配置是在 Watch 配置之后加载的，会**完全替换**之前的 `restock_settings`，而不是合并。

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

### 3.2 价格阈值的反向覆盖机制（核心）

**这是最容易被误解的逻辑。价格阈值不是"补充条件"，而是"否决条件"。**

#### 3.2.1 执行顺序和覆盖关系

代码执行顺序 (`processor.py:605-654`):

```
步骤 1: 初始化 changed_detected = False

步骤 2: 库存变化判定 (609-616)
        条件: watch['restock']['in_stock'] != update_obj['restock']['in_stock']
        → 如果满足: changed_detected = True

步骤 3: 价格变化判定 (618-625)
        条件: follow_price_changes=True 且 price != original_price
        → 如果满足: changed_detected = True

步骤 4: 价格区间否决 (637-641)  ← 反向覆盖！
        条件: price 在 [min, max] 区间内
        → 如果满足: changed_detected = False (强制覆盖)

步骤 5: 百分比阈值否决 (646-652)  ← 反向覆盖！
        条件: changed_detected=True 且 变化百分比 ≤ 阈值
        → 如果满足: changed_detected = False (强制覆盖)
```

#### 3.2.2 关键理解

| 阶段 | 代码行 | 对 changed_detected 的操作 | 影响范围 |
|-----|-------|---------------------------|---------|
| 库存变化判定 | 609-616 | 设为 True (条件满足时) | 仅库存变化 |
| 价格变化判定 | 618-625 | 设为 True (条件满足时) | 仅价格变化 |
| 价格区间否决 | 637-641 | **强制设为 False** | **库存变化 + 价格变化** |
| 百分比阈值否决 | 646-652 | **强制设为 False** | **前面已判定的变更** |

**重要结论**: 价格阈值的否决是**无条件覆盖**的，即使已经因为库存变化设置了 `changed_detected=True`，只要价格在区间内，就会被强制改为 `False`。

#### 3.2.3 价格区间否决逻辑

```python
# processor.py:637-641
if min_limit or max_limit:
    if is_between(number=price, lower=min_limit, upper=max_limit):
        # Price was between min/max limit, so there was nothing todo in any case
        logger.trace(f"{watch.get('uuid')} {price} is between {min_limit} and {max_limit}, nothing to check, forcing changed_detected = False (was {changed_detected})")
        changed_detected = False  # ← 不管之前是什么，直接设为 False
```

**`is_between()` 函数**:

```python
# processor.py:388-400
def is_between(number, lower=None, upper=None):
    """
    检查数字是否在区间内
    - lower=None 表示无下限
    - upper=None 表示无上限
    - 两端都是闭区间 (包含边界值)
    """
    return (lower is None or lower <= number) and (upper is None or number <= upper)
```

**区间判定示例** (设置 `price_change_min=900`, `price_change_max=1100`):

| 当前价格 | 判定结果 | 说明 |
|---------|---------|------|
| 1000 | `is_between=True` → `changed_detected=False` | 在区间内，否决变更 |
| 850 | `is_between=False` → 不改变 | 低于下限，不否决 |
| 1200 | `is_between=False` → 不改变 | 高于上限，不否决 |
| 900 | `is_between=True` → `changed_detected=False` | 等于下限，属于区间内 |
| 1100 | `is_between=True` → `changed_detected=False` | 等于上限，属于区间内 |

#### 3.2.4 百分比阈值否决逻辑

```python
# processor.py:646-652
if watch['restock'].get('original_price') and changed_detected and restock_settings.get('price_change_threshold_percent'):
    previous_price = float(watch['restock'].get('original_price'))
    pc = float(restock_settings.get('price_change_threshold_percent'))
    change = abs((price - previous_price) / previous_price * 100)
    if change and change <= pc:
        logger.debug(f"{watch.get('uuid')} Override change-detected to FALSE because % threshold ({pc}%) was {change:.3f}%")
        changed_detected = False  # ← 强制否决
```

**百分比判定示例** (原价 = 1000, 阈值 = 5%):

| 当前价格 | 变化量 | 变化百分比 | 判定结果 |
|---------|--------|-----------|---------|
| 1030 | +30 | 3.0% | ≤ 5% → `changed_detected=False` |
| 960 | -40 | 4.0% | ≤ 5% → `changed_detected=False` |
| 1060 | +60 | 6.0% | > 5% → 保持原值 |
| 940 | -60 | 6.0% | > 5% → 保持原值 |

### 3.3 完整触发决策表

让我们用具体场景说明价格阈值如何**反向覆盖**库存变更：

#### 场景 1: 缺货 → 有货 + 价格在区间内

配置: `in_stock_processing=in_stock_only`, `price_change_min=900`, `price_change_max=1100`

| 前状态 | 前价格 | 后状态 | 后价格 | 执行过程 | 最终结果 |
|-------|--------|-------|--------|---------|---------|
| 缺货 | 1000 | 有货 | 1050 | 1. 库存变化 → changed_detected=True<br>2. 价格未变 → 无影响<br>3. 价格 1050 在 [900, 1100] 区间内 → **changed_detected=False** | **False** (不触发) |

**结论**: 即使商品从缺货变为有货，只要价格仍在区间内，就**不会触发通知**。这意味着价格区间的优先级高于库存变化。

#### 场景 2: 缺货 → 有货 + 价格超出区间

配置: 同上

| 前状态 | 前价格 | 后状态 | 后价格 | 执行过程 | 最终结果 |
|-------|--------|-------|--------|---------|---------|
| 缺货 | 1000 | 有货 | 850 | 1. 库存变化 → changed_detected=True<br>2. 价格变化 → changed_detected=True<br>3. 价格 850 不在 [900, 1100] 区间内 → 无否决<br>4. 百分比 15% > 阈值(默认无) → 无否决 | **True** (触发) |

#### 场景 3: 有货 → 有货 + 价格变化在区间内

配置: `follow_price_changes=True`, 阈值 = 5%

| 前状态 | 前价格 | 后状态 | 后价格 | 执行过程 | 最终结果 |
|-------|--------|-------|--------|---------|---------|
| 有货 | 1000 | 有货 | 1030 | 1. 库存未变 → 无影响<br>2. 价格变化 3% → changed_detected=True<br>3. 无区间限制 → 无否决<br>4. 变化 3% ≤ 5% → **changed_detected=False** | **False** (不触发) |

#### 场景 4: 有货 → 有货 + 价格变化超出阈值

| 前状态 | 前价格 | 后状态 | 后价格 | 执行过程 | 最终结果 |
|-------|--------|-------|--------|---------|---------|
| 有货 | 1000 | 有货 | 1080 | 1. 库存未变 → 无影响<br>2. 价格变化 8% → changed_detected=True<br>3. 无区间限制 → 无否决<br>4. 变化 8% > 5% → 无否决 | **True** (触发) |

### 3.4 多价格检测规则

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

### 5.1 完整流程图（含价格否决机制）

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
│     • 读取 restock_diff.json (Watch 级别)                            │
│     • 遍历 tags，若 tag.overrides_watch=True → 用 tag 配置覆盖       │
│     • Tag 配置 > Watch 配置 > 默认值                                 │
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
│  5. 变更判定 (核心: 两步设置 + 两步否决)                               │
│     ──────────────────────────────────────────────────────────      │
│                                                                      │
│     初始化: changed_detected = False                                 │
│                                                                      │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 5.1 库存变化 → 设置 True                               │        │
│     │ ──────────────────────────────────────────────────   │        │
│     │ 条件:                                                  │        │
│     │   • 前后 in_stock 不同                                  │        │
│     │   • (in_stock_only 且 当前为 True) OR (all_changes)    │        │
│     │                                                        │        │
│     │ 结果: changed_detected = True (条件满足时)              │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼                                  │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 5.2 价格变化 → 设置 True                               │        │
│     │ ──────────────────────────────────────────────────   │        │
│     │ 条件:                                                  │        │
│     │   • follow_price_changes = True                        │        │
│     │   • price != original_price                            │        │
│     │                                                        │        │
│     │ 结果: changed_detected = True (条件满足时)              │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼                                  │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 5.3 价格区间 → 强制否决 (设为 False)                    │        │
│     │ ──────────────────────────────────────────────────   │        │
│     │ ⚠️  反向覆盖！无论前面是什么结果                       │        │
│     │                                                        │        │
│     │ 条件:                                                  │        │
│     │   • min_limit 或 max_limit 已设置                       │        │
│     │   • price ∈ [min, max] (闭区间)                         │        │
│     │                                                        │        │
│     │ 结果: changed_detected = False (强制)                   │        │
│     │ 影响: 库存变化 和 价格变化 都被否决                       │        │
│     └──────────────────────────────────────────────────────┘        │
                                   │                                  │
                                   ▼                                  │
│     ┌──────────────────────────────────────────────────────┐        │
│     │ 5.4 百分比阈值 → 强制否决 (设为 False)                  │        │
│     │ ──────────────────────────────────────────────────   │        │
│     │ ⚠️  反向覆盖！仅当 changed_detected=True 时检查         │        │
│     │                                                        │        │
│     │ 条件:                                                  │        │
│     │   • changed_detected = True (来自前两步)                │        │
│     │   • threshold_percent 已设置                            │        │
│     │   • |变化百分比| ≤ 阈值                                 │        │
│     │                                                        │        │
│     │ 结果: changed_detected = False (强制)                   │        │
│     └──────────────────────────────────────────────────────┘        │
│                                                                      │
│     最终: changed_detected 经过所有否决后的值                         │
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
│  → 判定价格变化 + 价格阈值否决              │
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

### 6.2 触发条件（修正版）

#### 6.2.1 库存状态变更触发（可能被价格阈值否决）

| 配置 | 前状态 | 后状态 | 价格是否在区间 | 是否触发 |
|-----|--------|--------|---------------|---------|
| `in_stock_only`, min=900, max=1100 | 缺货 | 有货 | 是 (1000) | **否** (被价格区间否决) |
| `in_stock_only`, min=900, max=1100 | 缺货 | 有货 | 否 (850) | **是** |
| `in_stock_only` | 有货 | 缺货 | - | 否 |
| `all_changes`, min=900, max=1100 | 有货 | 缺货 | 是 (1000) | **否** (被价格区间否决) |
| `all_changes`, min=900, max=1100 | 有货 | 缺货 | 否 (1200) | **是** |

#### 6.2.2 价格变更触发（可能被阈值否决）

| 条件 | 是否触发 |
|-----|---------|
| 价格 ≠ 原始价格，且超出区间 [min, max]，且变化 > 阈值 | **是** |
| 价格 ≠ 原始价格，但在区间内 | 否 (被区间否决) |
| 价格 ≠ 原始价格，超出区间，但变化 ≤ 阈值 | 否 (被百分比否决) |
| 价格 = 原始价格 | 否 |

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

### 9.1 典型配置（结合反向覆盖机制）

#### 场景 1: 仅监控补货（不关心价格）

```python
{
    'follow_price_changes': False,
    'in_stock_processing': 'in_stock_only',
    'price_change_min': None,
    'price_change_max': None,
    'price_change_threshold_percent': None
}
```

**行为**:
- 商品从缺货变为有货时 → **触发通知**（无价格阈值否决）
- 不关心价格

#### 场景 2: 补货 + 降价到目标价以下

```python
{
    'follow_price_changes': True,
    'in_stock_processing': 'in_stock_only',
    'price_change_max': 900.0,  # 只有价格降到 900 以下才会"不在区间内"
    'price_change_threshold_percent': None
}
```

**关键理解**:
- `price_change_max=900` 表示区间是 `(-∞, 900]`
- 价格 ≤ 900 时 → 在区间内 → **否决变更**（即使补货也不触发）
- 价格 > 900 时 → 不在区间内 → 不否决

**这意味着：**
- 商品缺货时价格是 950，补货后价格降到 850 → 不在区间内 → **触发**
- 商品缺货时价格是 1000，补货后价格还是 950 → 在区间内 → **不触发**（虽然补货了，但价格没降到阈值以下）

#### 场景 3: 补货通知不受价格影响

```python
{
    'follow_price_changes': False,  # 关键：不跟踪价格变化
    'in_stock_processing': 'in_stock_only',
    'price_change_min': None,
    'price_change_max': None,
    'price_change_threshold_percent': None
}
```

**关键点**: 当 `follow_price_changes=False` 时，价格阈值逻辑不会执行（第 3 步的 `if` 条件不满足），所以补货通知不会被价格区间否决。

#### 场景 4: 全状态监控

```python
{
    'follow_price_changes': True,
    'in_stock_processing': 'all_changes',
    'price_change_threshold_percent': 2,
    'price_change_min': None,
    'price_change_max': None
}
```

**行为**:
- 库存任何变化 (缺货↔有货) → 可能触发（无价格区间否决）
- 价格变化 > 2% → 触发
- 价格变化 ≤ 2% → 被百分比阈值否决 → 不触发

## 十、关键代码位置

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 主判定函数 | `restock_diff/processor.py` | 403-659 |
| 配置加载 + Tag 覆盖 | `restock_diff/processor.py` | 455-467 |
| 库存状态判定 | `restock_diff/processor.py` | 553-562 |
| 价格变化判定（设置 True） | `restock_diff/processor.py` | 609-625 |
| 价格区间否决（强制 False） | `restock_diff/processor.py` | 637-641 |
| 百分比阈值否决（强制 False） | `restock_diff/processor.py` | 646-652 |
| is_between 区间判定函数 | `restock_diff/processor.py` | 388-400 |
| 纯 Python 提取器 | `restock_diff/pure_python_extractor.py` | 1-289 |
| 元数据提取 (extruct) | `restock_diff/processor.py` | 315-385 |
| 子进程内存管理 | `restock_diff/processor.py` | 249-310 |
| 配置表单 | `restock_diff/forms.py` | 1-82 |
| Restock 数据结构 | `restock_diff/__init__.py` | 17-67 |
| 任务触发入口 | `worker.py` | 156-175 |
| 基础处理器 | `processors/base.py` | 200-365 |

---

**文档版本**: 2.0 (修正两处关键逻辑)
**基于代码版本**: 当前仓库版本
**更新日期**: 2026-05-14

**修正记录**:
1. 配置覆盖优先级：Tag(overrides_watch) > Watch 自身配置 > 默认值（后加载覆盖前加载）
2. 价格阈值反向覆盖：价格区间和百分比阈值是"否决"逻辑，会强制将 `changed_detected` 设为 `False`，即使库存变化已经设置为 `True`
