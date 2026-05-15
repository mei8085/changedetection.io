# Restock & Price Detection Processor 分析报告

## 1. 概述

**处理器名称**: Re-stock & Price Detection for pages with a SINGLE product  
**文件位置**: `changedetectionio/processors/restock_diff/`  
**核心功能**: 专门针对单一产品页面，检测商品库存状态变化和价格波动。

---

## 2. 架构设计

### 2.1 核心类结构

```
difference_detection_processor (base.py)
    └── perform_site_check (processor.py)
        ├── Restock 数据模型
        ├── 多层提取策略
        └── 变更检测逻辑
```

### 2.2 主要模块文件

| 文件 | 功能 |
|------|------|
| `__init__.py` | Restock 数据模型定义、Watch 扩展、价格解析 |
| `processor.py` | 核心处理器、提取管道、检测逻辑 |
| `pure_python_extractor.py` | 纯Python元数据提取器（无lxml） |
| `forms.py` | WTForms 配置表单 |
| `plugins/llm_restock.py` | LLM 备用提取插件 |
| `api.yaml` | API 规范定义 |

---

## 3. 核心数据模型

### 3.1 Restock 类

```python
class Restock(dict):
    default_values = {
        'in_stock': None,        # 布尔值 - 是否有库存
        'price': None,           # float - 当前价格
        'currency': None,        # str - 货币代码 (USD/EUR等)
        'original_price': None   # float - 首次检测的价格
    }
```

**关键功能**:
- 自动将字符串价格转换为浮点数
- 处理不同地区的数字格式（逗号/小数点）
- 使用 Babel 库进行本地化解析

---

## 4. 多层提取策略（关键设计）

### 4.1 提取管道优先级

```
┌─────────────────────────────────────────────────────────────┐
│                    提取管道优先级排序                        │
├─────────────────────────────────────────────────────────────┤
│  1. 纯 Python 元数据提取  (pure_python_extractor.py)        │
│     → JSON-LD / OpenGraph / Microdata                       │
│     → 无 lxml，避免内存泄漏                                 │
│     → 覆盖 80%+ 现代电商网站                                │
├─────────────────────────────────────────────────────────────┤
│  2. 内置 Extruct 提取  (processor.py)                       │
│     → 使用 extruct 库解析所有结构化数据                      │
│     → JSON-LD / Microdata / OpenGraph / RDFa                │
│     → Linux 下通过 subprocess 隔离防止内存泄漏               │
├─────────────────────────────────────────────────────────────┤
│  3. LLM 备用提取  (plugins/llm_restock.py)                  │
│     → 当前两种方法均失败时触发                               │
│     → 发送清理后的页面文本给 LLM 进行结构化提取              │
│     → 可配置开关                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 纯 Python 提取器详解

**设计目标**:
- 避免使用 lxml 导致的 C 级内存泄漏
- 快速、轻量级，覆盖大部分场景
- 使用 Python 内置 html.parser

**提取顺序**:
1. **JSON-LD** - `<script type="application/ld+json">` 块
2. **OpenGraph** - `<meta property="og:*">` 标签
3. **Microdata** - itemprop 属性元素

### 4.3 内存管理优化（Linux）

**问题背景**:
- lxml 解析大型 HTML 时会分配大量 C 级内存
- Python GC 无法释放这些内存，导致进程内存持续增长
- 5MB+ 页面可能导致每次解析增长 50-500MB

**解决方案**:
```python
# Linux 平台使用 multiprocessing spawn 模式
# subprocess 完成后 OS 自动回收所有内存
ctx = multiprocessing.get_context('spawn')
parent_conn, child_conn = ctx.Pipe()
p = ctx.Process(target=_extract_itemprop_availability_worker, args=(child_conn,))
p.start()
# 通过管道传输 HTML 和结果
# 进程退出后 OS 强制回收所有内存
```

**性能权衡**:
- ✅ 完全消除内存泄漏
- ✅ 内存增长稳定在 35MB 左右
- ⚠️ 额外的进程启动开销（约 100-200ms）

---

## 5. 库存状态检测规则

### 5.1 结构化数据中的 Availability 值

```python
# 视为 "有库存" 的值
in_stock_keywords = [
    'instock', 'instoreonly', 'limitedavailability',
    'onlineonly', 'presale'
]

# 视为 "无库存" 的值
out_of_stock_keywords = [
    'outofstock', 'soldout', 'discontinued',
    'temporarilyoutofstock'
]
```

### 5.2 浏览器 JS 级检测（fallback）

当页面没有结构化数据时，使用前端 JS 检测页面文本：

```javascript
// stock-not-in-stock.js 检测关键字
// 忽略页面顶部 300px 内的内容（避免导航栏误判）
检测关键词:
  - 有库存: "Add to cart", "Buy now", "Available", "In stock"
  - 无库存: "Out of stock", "Sold out", "Unavailable"
```

### 5.3 条件补位加负向覆盖机制

**核心逻辑**（processor.py 第 583-595 行）：
1. **结构化数据优先**：元数据中有 availability 时，先以元数据为准
2. **条件补位**：元数据无 availability 时，用 JS 检测结果填充
3. **负向覆盖**：JS 检测到明确缺货时，强制覆盖为 False（无论元数据怎么说）

**instock_data 的两种取值说明**：
| JS 返回值 | 含义 |
|-----------|------|
| `'Possibly in stock'` | 页面首屏未找到缺货关键词，推测可能有货 |
| 其他值（如 `'Out of stock'`） | JS 明确检测到缺货文本 |

---

### 5.4 最终 in_stock 落值状态矩阵

```
┌───────────────────────────┬────────────────────────────────────────────────────────┬────────────────────────────────────────────────────────┐
│                           │           JS: instock_data = 'Possibly in stock'        │           JS: instock_data = 明确缺货                 │
│                           │              (JS推测有货，未发现缺货文本)                 │                 (JS明确检测到缺货)                      │
├───────────────────────────┼────────────────────────────────────────────────────────┼────────────────────────────────────────────────────────┤
│                           │  ① 结构化数据有值 → 以结构化数据为准                     │  ③ 负向覆盖生效！强制 = False                          │
│  元数据: availability 有值 │                                                         │  无论元数据说有货还是缺货，只要JS明确检测到缺货，        │
│                           │  (不执行补位逻辑，直接用元数据解析结果)                   │  就强制覆盖为 False（防止网站元数据撒谎）                │
├───────────────────────────┼────────────────────────────────────────────────────────┼────────────────────────────────────────────────────────┤
│                           │  ② 条件补位 = True                                       │  ④ 条件补位 = False                                    │
│  元数据: availability 无值 │  JS: 'Possibly in stock' → in_stock = True              │  JS: 明确缺货 → in_stock = False                       │
│                           │  (JS推测有货，直接作为结果)                               │  (JS检测到缺货，直接作为结果)                           │
└───────────────────────────┴────────────────────────────────────────────────────────┴────────────────────────────────────────────────────────┘
```

**代码分支对应说明**：
- 分支1（第583行）：`if self.fetcher.instock_data and itemprop_availability.get('availability') is None`
  → 元数据无值时，用 JS 结果条件补位
- 分支2（第590行）：`if self.fetcher.instock_data and self.fetcher.instock_data != 'Possibly in stock'`
  → JS明确缺货时，无条件负向覆盖为 False

> **关键设计意图**：网站经常在元数据中撒谎（明明缺货，metadata却标有货），但页面上必然要显示"缺货"给用户看。因此只要 JS 在页面可见区域检测到缺货文本，就强制覆盖为 False，这是可信度最高的信号。

---

## 6. 价格变更检测（真实分支校准版）

### 6.1 检测参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `follow_price_changes` | Boolean | 是否启用价格追踪 |
| `price_change_min` | Float | 价格下限（价格低于此值触发） |
| `price_change_max` | Float | 价格上限（价格高于此值触发） |
| `price_change_threshold_percent` | Float | 价格变化百分比阈值（变化幅度超过此值才触发） |

### 6.2 价格去重逻辑

```python
def _deduplicate_prices(data):
    # 处理以下情况:
    # 1. 同一价格的不同表示: "$159", "159", 159
    # 2. 多个数据源重复: JSON-LD 和 Microdata 都有价格
    # 3. 千位分隔符差异: "1,299" vs "1299"
    #
    # 返回去重后的唯一价格集合
```

### 6.3 真实变更触发分支（按代码顺序）

```python
# ┌── 步骤 1: 启用价格追踪且有历史数据 ─────────────────────────┐
# │  IF follow_price_changes AND watch.restock EXISTS          │
# │     AND current.restock.price EXISTS                       │
# └────────────────────────────────────────────────────────────┘
#                           │
#                           ▼
# ┌── 步骤 2: 价格变化初步检测 ────────────────────────────────┐
# │  IF original_price EXISTS AND price != original_price      │
# │      changed_detected = True                               │
# └────────────────────────────────────────────────────────────┘
#                           │
#                           ▼
# ┌── 步骤 3: 价格区间过滤（反向逻辑） ──────────────────────────┐
# │  IF min_limit OR max_limit IS SET                          │
# │      AND price IS BETWEEN min_limit AND max_limit          │
# │          FORCE changed_detected = False                    │
# │                                                            │
# │  注意: 设置区间后，只有价格跳出区间才触发，区间内不触发      │
# └────────────────────────────────────────────────────────────┘
#                           │
#                           ▼
# ┌── 步骤 4: 百分比阈值过滤 ───────────────────────────────────┐
# │  IF original_price EXISTS AND changed_detected IS TRUE      │
# │     AND threshold_percent IS SET                            │
# │      change_percent = abs((price - original) / original * 100) │
# │      IF change_percent <= threshold_percent                 │
# │          FORCE changed_detected = False                     │
# └────────────────────────────────────────────────────────────┘
```

**关键注意事项**:
- 价格变化比较始终以 `original_price`（首次检测价格）为基准，而非上一次价格
- min/max 是**反向过滤器**：价格在区间内时**不触发**，跳出区间才触发
- 百分比阈值是**最小变化量**：变化幅度必须超过阈值才触发
- 所有过滤条件均可叠加，任一条件将 `changed_detected` 设为 False 就不会触发通知

---

## 7. LLM 备用提取插件

### 7.1 触发条件

```python
# 当以下条件满足时启用 LLM 提取:
1. 纯 Python 提取未同时获得 price + availability
2. Extruct 提取也未同时获得两者
3. 系统配置中启用了 LLM fallback
4. LLM API key 已配置
```

### 7.2 预处理优化

**降噪步骤** (`_strip_html`):
1. 提取 JSON-LD 块前置（可靠结构化数据）
2. 移除 nav/header/footer/aside 标签
3. 移除推荐商品区块（避免混淆价格）
4. 删除 script/style 块和 HTML 注释
5. 剥离所有 HTML 标签
6. 压缩空白字符
7. 截断至 8000 字符

### 7.3 系统 Prompt 关键点

- **价格提取规则**: 忽略导航栏购物车价格、推荐商品价格、价格区间
- **库存判断**: 只看产品区域的 "Add to cart" 等按钮，忽略导航栏
- **输出格式**: 严格的 JSON 格式 `{price, currency, availability}`
- **分类网站特殊处理**: eBay/分类网站有价格 + 卖家联系方式 = 有库存

---

## 8. 处理器能力声明

```python
supports_visual_selector = True           # 支持视觉选择器
supports_browser_steps = True             # 支持浏览器步骤
supports_text_filters_and_triggers = True # 支持文本过滤触发器
supports_request_type = True              # 支持自定义请求方法
```

---

## 9. 关键状态流转详解

### 9.1 original_price 与 previous_price 的本质区别

| 字段 | 存储位置 | 生命周期 | 用途 |
|------|----------|----------|------|
| **original_price** | `watch['restock']` 对象中 | 首次检测设置后永久不变 | 价格变化检测的基准值 |
| **previous_price** | 仅通知模板动态计算 | 不持久化，每次通知时从历史快照解析 | 通知内容展示，不参与检测逻辑 |

**original_price 设置时机** (processor.py 第 568-570 行):
```python
# 首次检测到价格且 original_price 未设置时，将当前价格设为原始价格
if itemprop_availability and itemprop_availability.get('price') 
   and not itemprop_availability.get('original_price'):
    itemprop_availability['original_price'] = itemprop_availability.get('price')
    update_obj['restock']["original_price"] = itemprop_availability.get('price')
```

**previous_price 计算方式** (`__init__.py` 第 94-104 行):
```python
# 从历史快照中解析
if self.history_n >= 2:
    sorted_keys = sorted(list(history), key=lambda x: int(x))
    sorted_keys.reverse()  # 最新在前
    # 取倒数第一条（即上一次快照）解析价格
    price_str = self.get_history_snapshot(timestamp=sorted_keys[-1])
    values['restock']['previous_price'] = get_price_from_history_str(price_str)
```

> **重要提示**: `previous_price` 仅用于通知模板展示，**不参与任何检测逻辑**。所有价格变化检测都以 `original_price` 为基准。

### 9.2 in_stock 状态流转模式

#### 模式 1: in_stock_only（默认）

```
触发条件: 库存状态发生变化 AND 当前状态为 in_stock=True

状态流转矩阵:
┌─────────────┬──────────────────┬──────────────────┐
│  前状态 \ 后状态  │    缺货(False)   │   有货(True)     │
├─────────────┼──────────────────┼──────────────────┤
│  缺货(False)  │  状态未变，不触发  │  ✅ 触发通知      │
├─────────────┼──────────────────┼──────────────────┤
│  有货(True)   │  ❌ 不触发通知    │  状态未变，不触发  │
└─────────────┴──────────────────┴──────────────────┘
```

**设计意图**: 补货提醒场景，用户只关心"从无货到有货"这一瞬间。

#### 模式 2: all_changes

```
触发条件: 只要库存状态发生任何变化就触发

状态流转矩阵:
┌─────────────┬──────────────────┬──────────────────┐
│  前状态 \ 后状态  │    缺货(False)   │   有货(True)     │
├─────────────┼──────────────────┼──────────────────┤
│  缺货(False)  │  状态未变，不触发  │  ✅ 触发通知      │
├─────────────┼──────────────────┼──────────────────┤
│  有货(True)   │  ✅ 触发通知      │  状态未变，不触发  │
└─────────────┴──────────────────┴──────────────────┘
```

**设计意图**: 需要完整监控库存状态变化的场景（如竞品监控）。

#### 模式 3: off

```
完全关闭库存状态检测，仅保留价格变化检测功能
```

### 9.3 快照 MD5 计算机制

每次检测生成快照内容并计算 MD5：
```python
snapshot_content = f"In Stock: {in_stock} - Price: {price}"
fetched_md5 = hashlib.md5(snapshot_content.encode('utf-8')).hexdigest()
```

- MD5 变化是最终判定 `changed_detected` 的依据
- 价格或库存任一变化都会导致 MD5 变化
- 后续的过滤逻辑在此基础上进行二次确认

---

## 10. 检测流程时序图

```
调用 perform_site_check()
    │
    ├─► 计算 HTML MD5 校验和
    │   └─► 如果校验和相同 + 未编辑配置 → 跳过处理
    │
    ├─► 读取 restock_diff.json 配置
    │   └─► 检查是否有 Tag 覆盖本 watch 配置
    │
    ├─► 第一层: 纯 Python 元数据提取
    │   └─► 成功(price+avail)? → 用此结果
    │
    ├─► 第二层: Extruct 提取 (Linux subprocess)
    │   └─► 成功(price+avail)? → 用此结果
    │
    ├─► 第三层: 检查 LLM 插件覆盖
    │   └─► LLM 可用且需要? → 调用 LLM 提取
    │
    ├─► 检查多价格异常
    │   └─► 多价格且插件也失败 → 抛出异常
    │
    ├─► 浏览器 JS 库存检测结果覆盖
    │   └─► JS 检测无库存 → 强制覆盖 in_stock=False
    │
    ├─► 首次检测初始化 original_price
    │   └─► 未设置过原始价格? → 将当前价格保存为 original_price
    │
    ├─► 生成快照内容与 MD5
    │   └─► "In Stock: {x} - Price: {y}" → MD5
    │
    └─► 变更检测判断
        ├─► 库存状态变更检测
        │   └─► in_stock_only / all_changes 模式判定
        ├─► 价格变更检测（按顺序执行）
        │   ├─► 与 original_price 比较是否变化
        │   ├─► 应用 min/max 区间过滤（区间内强制不触发）
        │   └─► 应用百分比阈值过滤（变化过小强制不触发）
        └─► 返回 (changed_detected, update_obj, snapshot)
```

---

## 11. 关键配置选项

### 11.1 库存检测模式

| 模式 | 行为 |
|------|------|
| `in_stock_only` | **默认** - 仅当从"缺货"变为"有货"时触发 |
| `all_changes` | 任何库存状态变更都触发通知 |
| `off` | 完全关闭库存检测 |

### 11.2 价格检测配置

- **follow_price_changes**: 开启/关闭价格追踪
- **price_change_min/max**: 价格区间过滤（跳出区间才触发）
- **price_change_threshold_percent**: 相对于原始价格的变化百分比阈值（超过才触发）

---

## 12. 异常处理

### 12.1 异常类型

```python
class UnableToExtractRestockData(Exception):
    # 无法从页面提取任何库存/价格数据
    # 可能原因: 页面非产品页、结构化数据缺失

class MoreThanOnePriceFound(Exception):
    # 页面检测到多个不同价格
    # 触发条件: 去重后仍有多个价格
    # 处理: 尝试用 LLM 确定正确价格，仍失败则报错

class ProcessorException(基类):
    # 通用处理器异常
```

### 12.2 错误恢复策略

1. **多价格错误**: 先尝试 LLM 插件提取正确价格，仍失败才报错
2. **提取失败降级**: 结构化数据失败 → 降级到 JS 文本检测
3. **内存隔离失败**: subprocess 失败 → 降级到直接调用

---

## 13. 测试覆盖（路径对齐版）

### 13.1 现有测试文件（与仓库目录精确对齐）

| 相对仓库路径 | 覆盖内容 |
|----------|----------|
| `changedetectionio/tests/restock/test_restock.py` | 端到端库存检测流程 |
| `changedetectionio/tests/test_restock_itemprop.py` | 结构化数据提取 |
| `changedetectionio/tests/unit/test_restock_logic.py` | 核心逻辑单元测试 |
| `changedetectionio/tests/llm/test_llm_restock_plugin.py` | LLM 插件测试 |

### 13.2 关键测试场景

- 缺货 → 有货 通知触发（默认行为）
- 有货 → 缺货 不通知（默认）
- 价格变化检测与阈值
- 多价格页面的错误处理
- 标签配置覆盖 watch 配置
- LLM fallback 触发条件

---

## 14. 设计亮点与优化点

### 14.1 优秀设计

1. **分层提取策略**: 从快到慢、从可靠到 fallback 的多级提取
2. **内存隔离**: Linux 下 subprocess 彻底解决 lxml 内存泄漏
3. **真相源优先级**: 浏览器实际可见内容优先级高于元数据
4. **降噪处理**: LLM 提取前智能移除导航栏/推荐区块噪声
5. **配置可覆盖**: 支持 Tag 级别配置覆盖单个 watch
6. **价格基准设计**: 使用 original_price 作为基准避免价格波动噪声

### 14.2 潜在优化方向

1. **正则预提取**: 在 extruct 之前先用正则提取 JSON-LD 等小块，避免解析整个 5MB HTML
2. **价格历史**: 当前只比较原始价格，可增加历史波动分析
3. **变体支持**: 增加对多变体产品（颜色/尺码）的库存检测支持
4. **缓存优化**: 提取的元数据可缓存，避免重复解析相同内容
5. **previous_price 参与检测**: 考虑增加相对于上一次价格的变化检测模式

---

## 15. 通知占位符

```jinja2
{{ restock.price }}              # 当前价格
{{ restock.in_stock }}           # 库存状态布尔值
{{ restock.original_price }}     # 首次检测的价格（检测基准）
{{ restock.previous_price }}     # 从历史快照解析的上一次价格（仅展示用）
```

---

## 总结

Restock 处理器是一个高度优化的电商产品监控模块，其核心优势在于：

1. **多层提取架构**确保了对各种网站的广泛兼容性
2. **内存泄漏防护**使其适合大规模部署（Linux subprocess 隔离）
3. **智能降噪和优先级机制**提高了检测准确率（浏览器检测 > 元数据）
4. **灵活的阈值配置**满足不同用户的监控需求（区间 + 百分比双重过滤）
5. **清晰的状态流转设计**：original_price 作为稳定检测基准，previous_price 用于通知展示，各司其职

该处理器特别适合电商价格监控、补货提醒等场景，是 changedetection.io 中设计最复杂也最成熟的处理器之一。
