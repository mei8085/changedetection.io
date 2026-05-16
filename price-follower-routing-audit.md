# changedetection.io 价格数据跟踪路由与过滤逻辑审计报告

> **审计范围**：RSS 蓝图路由形态、主 Feed 标签过滤流程、Watch 状态与处理器输出的数据衔接
> **代码版本**：基于当前主干版本深度分析
> **分析日期**：2026-05-16

---

## 目录

1. [RSS 蓝图完整路由形态](#1-rss-蓝图完整路由形态)
2. [主 Feed 标签过滤流程详解](#2-主-feed-标签过滤流程详解)
3. [Watch 状态数据流转](#3-watch-状态数据流转)
4. [处理器输出到 RSS 模板的数据链路](#4-处理器输出到-rss-模板的数据链路)
5. [跨模块依赖边界总结](#5-跨模块依赖边界总结)
6. [关键设计决策与技术债务](#6-关键设计决策与技术债务)

---

## 1. RSS 蓝图完整路由形态

### 1.1 路由注册机制

**文件位置**：`blueprint/rss/blueprint.py:1-26`

```python
def construct_blueprint(datastore: ChangeDetectionStore):
    rss_blueprint = Blueprint('rss', __name__)
    
    # 注册三个路由模块
    main_feed.construct_main_feed_routes(rss_blueprint, datastore)
    single_watch.construct_single_watch_routes(rss_blueprint, datastore)
    tag.construct_tag_routes(rss_blueprint, datastore)
    
    return rss_blueprint
```

**架构特征**：采用模块化路由注册，将主 Feed、单 Watch Feed、标签 Feed 分离为独立模块。

---

### 1.2 完整路由表

| 路由 | HTTP 方法 | 功能描述 | Token 鉴权 |
|------|-----------|---------|-----------|
| `/rss/` | GET | 重定向到主 Feed | - |
| `/rss` | GET | **主聚合 Feed**（所有未读变更） | ✅ |
| `/rss/watch/<uuid>` | GET | **单 Watch 历史变更 Feed** | ✅ |
| `/rss/tag/<tag_uuid>` | GET | **标签维度聚合 Feed** | ✅ |

---

### 1.3 标签 Feed 路由深度解析

**文件位置**：`blueprint/rss/tag.py:10-95`

```python
@rss_blueprint.route("/tag/<uuid_str:tag_uuid>", methods=['GET'])
def rss_tag_feed(tag_uuid):
    # ─── 1. 鉴权 ───
    is_valid, error = validate_rss_token(datastore, request)
    if not is_valid:
        return error
    
    # ─── 2. 标签存在性校验 ───
    tag = datastore.data['settings']['application'].get('tags', {}).get(tag_uuid)
    if not tag:
        return f"Tag with UUID {tag_uuid} not found", 404
    
    tag_title = tag.get('title', 'Unknown Tag')
    
    # ─── 3. Feed 元数据 ───
    fg = FeedGenerator()
    fg.title(f'changedetection.io - {tag_title}')
    fg.description(f'Changes for watches tagged with {tag_title}')
    fg.link(href='https://changedetection.io')
    
    # ─── 4. 遍历 Watch 筛选 ───
    for uuid, watch in datastore.data['watching'].items():
        # 4a. 标签匹配检查
        if tag_uuid not in watch.get('tags', []):
            continue
            
        # 4b. 静音过滤（全局配置开关）
        if datastore.data['settings']['application'].get('rss_hide_muted_watches') and watch.get('notification_muted'):
            continue
            
        # 4c. 历史快照数量检查（至少 2 个才有变更）
        dates = list(watch.history.keys())
        if len(dates) < 2:
            continue
            
        # 4d. 仅未读变更
        if not watch.viewed:
            # 构建 Feed 条目（同主 Feed 逻辑一致）
            diff_link = {'href': url_for('ui.ui_diff.diff_history_page', uuid=uuid, _external=True)}
            watch_label = get_watch_label(datastore, watch)
            timestamp_to = dates[-1]
            timestamp_from = dates[-2]
            
            guid = generate_watch_guid(watch, timestamp_to)
            n_body_template = get_rss_template(datastore, watch, rss_content_format,
                                               RSS_TEMPLATE_HTML_DEFAULT, RSS_TEMPLATE_PLAINTEXT_DEFAULT)
            
            n_object = build_notification_context(watch, timestamp_from, timestamp_to,
                                                 watch_label, n_body_template, rss_content_format)
            
            res = render_notification(n_object, notification_service, watch, datastore)
            
            fe = fg.add_entry()
            title_suffix = f"Change @ {res['original_context']['change_datetime']}"
            populate_feed_entry(fe, watch, res['body'], guid, timestamp_to, link=diff_link, title_suffix=title_suffix)
            add_watch_categories(fe, watch, datastore)  # 将 Watch 所有标签作为 <category>
    
    response = make_response(fg.rss_str())
    response.headers.set('Content-Type', 'application/rss+xml;charset=utf-8')
    return response
```

**关键注意事项**：
1. ❗ **代码注释自曝 Bug**：`#@todo This is wrong, it needs to sort by most recently changed`
2. ❗ **实际行为**：遍历顺序为 Watch 创建时间顺序（字典插入顺序），而非变更时间倒序
3. ✅ **鉴权一致**：所有 RSS 路由使用相同的 Token 校验逻辑

---

## 2. 主 Feed 标签过滤流程详解

### 2.1 标签名到内部 UUID 映射算法

**文件位置**：`blueprint/rss/main_feed.py:43-47`

```python
limit_tag = request.args.get('tag', '').lower().strip()

# 关键：标签名 → UUID 的映射循环
for uuid, tag in datastore.data['settings']['application'].get('tags', {}).items():
    if limit_tag == tag.get('title', '').lower().strip():
        limit_tag = uuid  # 替换为内部 UUID 标识
```

**算法流程图**：
```
请求参数 tag=amazon-products
    │
    ▼
字符串规范化：.lower().strip() → "amazon-products"
    │
    ▼
遍历所有标签字典 {uuid: tag_obj}
    │
    ▼
tag_obj.title.lower().strip() == "amazon-products"?
    ├─ Yes → limit_tag = uuid（如 "a1b2c3-d4e5..."）
    └─ No → limit_tag 保持原值（后续过滤全部失败）
```

**⚠️ 边缘情况**：
- 多个标签同名：取第一个匹配的 UUID（Python 字典插入顺序）
- 无匹配：`limit_tag` 保持为标签名字符串 → 后续 `limit_tag in watch['tags']` 始终为 False → 空 Feed

---

### 2.2 完整过滤流水线

**文件位置**：`blueprint/rss/main_feed.py:43-59`

```python
# ─── 阶段 1: 标签参数规范化与映射 ───
limit_tag = request.args.get('tag', '').lower().strip()

# 标签名 → UUID 映射
for uuid, tag in datastore.data['settings']['application'].get('tags', {}).items():
    if limit_tag == tag.get('title', '').lower().strip():
        limit_tag = uuid  # 关键：替换为内部 UUID 标识

# ─── 阶段 2: Watch 遍历与过滤 ───
sorted_watches = []
for uuid, watch in datastore.data['watching'].items():
    # 过滤器 A：静音过滤（全局配置控制）
    if datastore.data['settings']['application'].get('rss_hide_muted_watches'):
        if watch.get('notification_muted'):
            continue
    
    # 过滤器 B：标签过滤（仅当指定了 tag 参数时）
    if limit_tag and limit_tag not in watch.get('tags', []):
        continue
    
    # 通过所有过滤器
    sorted_watches.append(watch)

# ─── 阶段 3: 排序 ───
sorted_watches.sort(key=lambda x: x.last_changed, reverse=False)
```

**过滤条件真值表**：

| `tag` 参数 | 标签匹配 | 静音状态 | 结果 |
|------------|---------|---------|-----|
| 空/未传 | - | 任意 | ✅ 保留（除非静音过滤开启且已静音） |
| 有效标签名 | ✅ 匹配 | 未静音 | ✅ 保留 |
| 有效标签名 | ✅ 匹配 | 已静音（hide_muted=True） | ❌ 过滤 |
| 有效标签名 | ❌ 不匹配 | 任意 | ❌ 过滤 |
| 无效标签名（无匹配） | - | 任意 | ❌ 全部过滤（空 Feed） |

---

### 2.3 API Watch 列表与 RSS 过滤差异对比

| 维度 | API `/api/v1/watch?tag=X` | RSS `/rss?tag=X` |
|------|-------------------------|----------------|
| **匹配方式** | 标签名精确匹配（lowercase） | 先标签名 → UUID 映射，再 UUID 匹配 |
| **自动标签支持** | ✅ 支持 URL 自动匹配标签（`get_all_tags_for_watch`） | ❌ 仅检查 watch['tags'] 数组（手动分配的标签） |
| **多标签场景** | 任一匹配即可（遍历 tags.values()） | UUID 精确在 watch['tags'] 列表中 |

**关键差异代码**：
```python
# API 列表过滤（api/Watch.py:515-516）
tags = self.datastore.get_all_tags_for_watch(uuid=uuid)
if tag_limit and not any(v.get('title').lower() == tag_limit for k, v in tags.items()):
    continue  # 包含自动匹配标签

# RSS 过滤（main_feed.py:57）
if limit_tag and limit_tag not in watch['tags']:
    continue  # 仅手动分配的标签
```

---

## 3. Watch 状态数据流转

### 3.1 Watch 状态字段全景

| 字段 | 类型 | 持久化 | 更新时机 | RSS 影响 |
|------|------|--------|---------|---------|
| `uuid` | str | ✅ | 创建时 | GUID 生成、路由 |
| `tags` | list[str] | ✅ | 编辑页面提交 | 标签过滤、RSS category |
| `last_checked` | int | ✅ | 每次检查完成 | 调度阈值计算 |
| `last_changed` | int (property) | ❌ | 动态计算（历史快照） | Feed 排序 |
| `viewed` | bool (property) | ❌ | 用户点击查看时 | 变更可见性过滤 |
| `notification_muted` | bool | ✅ | 用户静音操作 | 静音过滤开关 |
| `processor` | str | ✅ | 编辑页面提交 | 处理器选择 |
| `restock` | dict | ✅ | restock_diff 处理器执行 | 价格/库存状态 |
| `previous_md5` | str | ✅ | 变更检测完成 | 变更检测判断 |
| `history` | dict (property) | ❌ | 动态读取快照目录 | 变更差异生成 |

---

### 3.2 状态流转时序图

```
┌─────────────────────────────────────────────────────────────────────┐
│                           调度器触发检查                              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Worker 异步处理队列任务                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 1. deepcopy(watch)  防并发冲突快照                              │  │
│  │ 2. call_browser()  抓取页面内容                                │  │
│  │ 3. run_changedetection()  处理器执行 → update_obj              │  │
│  │    └─ restock = {price, original_price, in_stock, currency}   │  │
│  │ 4. datastore.update_watch(uuid, update_obj)  写入状态         │  │
│  │ 5. save_history_blob()  保存快照文件                           │  │
│  │ 6. watch.reset_watch_edited_flag()  清除编辑标记              │  │
│  └───────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Watch 状态持久化完成                            │
│  - restock 字段更新（价格/库存）                                     │
│  - last_checked 更新（时间戳）                                       │
│  - previous_md5 更新（内容哈希）                                     │
│  - 快照文件写入磁盘                                                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        RSS Feed 请求到达                             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 1. validate_rss_token()  鉴权                                 │  │
│  │ 2. 标签过滤（tag 参数 → UUID 映射）                            │  │
│  │ 3. 静音过滤（rss_hide_muted_watches 配置）                    │  │
│  │ 4. viewed == False  仅未读变更过滤                            │  │
│  │ 5. 读取 history 快照 → 生成 diff → 渲染模板 → 输出 RSS XML   │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.3 `viewed` 属性计算逻辑

**文件位置**：`model/Watch.py:255-261`

```python
@property
def viewed(self):
    # last_viewed >= 最新快照时间戳 → 已读
    if int(self['last_viewed']) and int(self['last_viewed']) >= int(self.newest_history_key):
        return True
    return False

@property
def newest_history_key(self):
    # 历史快照中最大的时间戳（最新变更）
    if self.__newest_history_key is not None:
        return self.__newest_history_key
    if len(self.history) <= 1:
        return 0
    bump = self.history  # 触发历史读取计算
    return self.__newest_history_key
```

**关键机制**：
- `viewed` 是**计算属性**，非持久化字段
- 基于 `last_viewed`（持久化时间戳）与最新快照时间比较
- 用户点击 "Mark all viewed" 时更新 `last_viewed` 为当前时间

---

## 4. 处理器输出到 RSS 模板的数据链路

### 4.1 处理器输出结构（restock_diff）

**文件位置**：`processors/restock_diff/processor.py:547-659`

```python
def run_changedetection(self, watch, force_reprocess=False):
    # ─── 输出对象 update_obj ───
    update_obj = {
        'last_notification_error': False,
        'last_error': False,
        'restock': Restock({  # 核心价格/库存数据
            'price': float,           # 当前提取价格
            'original_price': float,  # 首次检测基准价
            'in_stock': bool,         # 库存布尔状态
            'availability': str,      # 原始可用性字符串
            'currency': str           # 货币符号
        }),
        'content-type': str,
        'last_check_status': int,  # HTTP 状态码
        'previous_md5': str        # 快照内容哈希
    }
    
    # ─── 快照内容（用于 diff） ───
    snapshot_content = f"In Stock: {update_obj['restock'].get('in_stock')} - Price: {price}"
    
    return changed_detected, update_obj, snapshot_content.strip()
```

---

### 4.2 Restock 对象到通知 Token 的注入

**文件位置**：`processors/restock_diff/__init__.py:90-116`

```python
def extra_notification_token_values(self):
    """扩展通知模板可用的 Token 变量"""
    values = super().extra_notification_token_values()
    values['restock'] = self.get('restock', {})
    
    # 从历史快照解析前一次价格
    values['restock']['previous_price'] = None
    if self.history_n >= 2:
        sorted_keys = sorted(list(self.history), key=lambda x: int(x))
        sorted_keys.reverse()
        price_str = self.get_history_snapshot(timestamp=sorted_keys[-1])
        if price_str:
            values['restock']['previous_price'] = get_price_from_history_str(price_str)
    
    return values

def extra_notification_token_placeholder_info(self):
    """模板变量文档"""
    values = super().extra_notification_token_placeholder_info()
    values.append(('restock.price', "Price detected"))
    values.append(('restock.in_stock', "In stock status"))
    values.append(('restock.original_price', "Original price at first check"))
    values.append(('restock.previous_price', "Previous price in history"))
    return values
```

---

### 4.3 完整数据链路（处理器 → RSS）

```
处理器执行 (restock_diff.run_changedetection())
    │
    ├─ 输出：changed_detected (bool)
    ├─ 输出：update_obj.restock
    │   ├─ price (当前价格)
    │   ├─ original_price (基准价)
    │   └─ in_stock (库存状态)
    └─ 输出：snapshot_content
        │
        ▼
datastore.update_watch(uuid, update_obj)  →  写入 Watch 对象
    │
    ▼
save_history_blob(snapshot_content)  →  写入快照文件（如 1715000000.txt）
    │
    ▼
RSS 渲染请求到达
    │
    ▼
watch.history  →  读取快照目录（包含 timestamp → 文件名映射）
    │
    ▼
notification_service.queue_notification_for_watch()
    │
    ├─ watch.extra_notification_token_values()
    │   └─ 注入 restock.* token
    │
    ▼
process_notification()  →  渲染 Jinja2 模板
    │
    ├─ {{ restock.price }} → 当前价格
    ├─ {{ restock.original_price }} → 基准价
    ├─ {{ restock.in_stock }} → 库存状态
    ├─ {{ restock.previous_price }} → 前次价格
    └─ {{ diff }} → 快照内容差异
        │
        ▼
        "In Stock: True - Price: 99.99"  vs
        "In Stock: True - Price: 89.99"
    │
    ▼
render_notification() 返回渲染后的 HTML/纯文本
    │
    ▼
populate_feed_entry() → 写入 RSS <description>
```

---

### 4.4 快照内容解析函数

**文件位置**：`processors/restock_diff/__init__.py:75-88`

```python
def get_price_from_history_str(s):
    """从历史快照字符串中解析价格
    
    快照格式："In Stock: True - Price: 99.99"
    """
    import re
    # "In Stock" could also show a variation of this 
    # e.g. "In Stock: False - Price: 99.99"
    m = re.search(r"Price: ([\d.]+)", s)
    if m:
        return float(m.group(1))
    return None
```

---

## 5. 跨模块依赖边界总结

### 5.1 模块依赖有向图

```
┌─────────────┐
│  RSS Blueprint  │
└──────┬──────┘
       │ 只读
       ▼
┌─────────────┐     读写     ┌──────────────┐
│  Watch Model   │ ◄────────── │  Processor   │
└──────┬──────┘              └──────────────┘
       │
       │ 只读
       ▼
┌──────────────────┐
│  Notification Svc  │  模板渲染
└──────────────────┘
       ▲
       │ 配置读取
┌──────────────────┐
│  Datastore (Settings)  │
└──────────────────┘
```

---

### 5.2 关键边界约束

| 边界 | 约束类型 | 实现方式 | 说明 |
|------|---------|---------|-----|
| RSS → Watch Model | **只读** | 直接属性访问 | RSS 从不修改 Watch 状态，纯输出层 |
| Processor → Watch Model | **读写** | `update_obj` 模式 | 处理器不直接修改 Watch，返回变更对象 |
| Tag System → Watch | **弱关联** | UUID 列表引用 | 标签与 Watch 仅通过 UUID 字符串关联，无强引用 |
| Notification Template → Watch | **扩展点** | `extra_notification_token_values()` | 处理器可注入自定义模板变量 |
| REST API → RSS | **过滤逻辑不统一** | 见 2.3 对比表 | API 列表与 RSS 过滤算法不一致 |

---

### 5.3 模板渲染流程中的边界控制

**文件位置**：`blueprint/rss/_util.py:114-127`

```python
def render_notification(n_object, notification_service, watch, datastore,
                       date_index_from=None, date_index_to=None):
    """安全边界：
    1. notification_service 无副作用调用（notification_q=False）
    2. Watch 对象只读访问
    3. 不触发任何状态变更
    """
    kwargs = {'n_object': n_object, 'watch': watch}
    
    if date_index_from is not None and date_index_to is not None:
        kwargs['date_index_from'] = date_index_from
        kwargs['date_index_to'] = date_index_to
    
    n_object = notification_service.queue_notification_for_watch(**kwargs)
    n_object['watch_mime_type'] = None
    
    res = process_notification(n_object=n_object, datastore=datastore)
    return res[0]  # [body, title]
```

**关键保障**：
- RSS 路由中 `NotificationService(datastore=datastore, notification_q=False)`
- `notification_q=False` 确保不会实际发送通知，仅用于模板渲染
- 纯计算操作，无副作用

---

## 6. 关键设计决策与技术债务

### 6.1 设计决策：标签名 → UUID 运行时映射

**决策点**：`/rss?tag=名称` 参数在运行时映射到 UUID

**优点**：
- URL 友好、人类可读
- 标签重命名后 URL 仍然有效

**缺点**：
- 每次请求遍历所有标签（O(n) 性能开销）
- 同名标签取第一个（非确定性）
- 标签名变更后历史 RSS 订阅可能失效

**改进建议**：
```python
# 建议：支持 UUID 直接传入以绕过映射
limit_tag = request.args.get('tag', '').strip()
if limit_tag:
    # 先尝试 UUID 精确匹配
    tag_exists = limit_tag in datastore.data['settings']['application'].get('tags', {})
    if not tag_exists:
        # 不是 UUID，尝试名称映射
        for uuid, tag in datastore.data['settings']['application'].get('tags', {}).items():
            if limit_tag.lower() == tag.get('title', '').lower().strip():
                limit_tag = uuid
                break
```

---

### 6.2 技术债务：标签 Feed 排序 Bug

**代码自曝**：`blueprint/rss/tag.py:48`
```python
#@todo This is wrong, it needs to sort by most recently changed and then limit it
```

**问题**：
- 当前实现按 Watch 创建时间遍历
- 应为按变更时间倒序（最新变更在前）

**修复方案**：
```python
# 收集 + 过滤 + 按最后变更时间排序
tagged_watches = []
for uuid, watch in datastore.data['watching'].items():
    if tag_uuid in watch.get('tags', []):
        # 其他过滤...
        tagged_watches.append(watch)

# 按最后变更时间倒序
tagged_watches.sort(key=lambda w: w.last_changed, reverse=True)
```

---

### 6.3 设计决策：标签与自动标签系统双轨制

**现状**：
- Watch 手动标签：`watch['tags'] = [uuid1, uuid2]`
- URL 自动匹配标签：`datastore.get_all_tags_for_watch()` 返回手动 + 自动

**不一致点**：
- RSS 过滤仅检查手动标签
- API 列表过滤检查所有标签（包括自动匹配）

**架构影响**：
- 用户困惑："为什么 API 有结果 RSS 是空的？"
- 功能不一致：两个接口同参数行为不同

---

### 6.4 设计决策：快照内容格式复用

**机制**：
```python
# 处理器生成简单文本快照
snapshot_content = f"In Stock: {in_stock} - Price: {price}"

# 存入历史快照文件
# 后续 diff 计算基于此字符串比较

# RSS 模板中解析使用
previous_price = get_price_from_history_str(previous_snapshot)
```

**优点**：
- 简单、可人类阅读
- 与通用 diff 算法兼容
- 向后兼容（所有处理器都输出字符串快照）

**缺点**：
- 解析依赖正则表达式（脆弱）
- 结构化数据丢失（序列化/反序列化成本）
- 国际化变更可能破坏解析

---

## 附录 A：关键文件索引

| 功能模块 | 文件路径 | 关键行数 |
|---------|---------|---------|
| RSS 蓝图主路由 | `changedetectionio/blueprint/rss/main_feed.py` | 43-105 |
| RSS 标签 Feed | `changedetectionio/blueprint/rss/tag.py` | 10-95 |
| RSS 工具函数 | `changedetectionio/blueprint/rss/_util.py` | 全文件 |
| Watch 状态模型 | `changedetectionio/model/Watch.py` | 255-295 |
| restock 处理器 | `changedetectionio/processors/restock_diff/processor.py` | 547-659 |
| restock Token 注入 | `changedetectionio/processors/restock_diff/__init__.py` | 90-116 |
| 通知服务 | `changedetectionio/notification_service.py` | 17-54 |
| API Watch 列表过滤 | `changedetectionio/api/Watch.py` | 511-528 |

---

## 附录 B：价格数据 Token 速查

| 模板变量 | 来源 | 说明 |
|---------|------|-----|
| `{{ restock.price }}` | `watch['restock']['price']` | 当前提取的价格 |
| `{{ restock.original_price }}` | `watch['restock']['original_price']` | 首次检测的基准价 |
| `{{ restock.previous_price }}` | 历史快照解析 | 上一次检查的价格 |
| `{{ restock.in_stock }}` | `watch['restock']['in_stock']` | 库存状态（True/False） |
| `{{ restock.currency }}` | `watch['restock']['currency']` | 货币符号 |
| `{{ diff }}` | 快照字符串比较 | 包含 In Stock + Price 的完整差异 |

---

**审计结论**：
1. ✅ 路由体系结构清晰，模块化设计良好
2. ⚠️ 标签过滤存在性能与一致性问题（双轨制、O(n) 映射）
3. ⚠️ 标签 Feed 存在已知排序 Bug（代码自曝）
4. ✅ 数据链路完整，处理器输出到 RSS 模板闭环清晰
5. ✅ 边界控制良好，RSS 纯只读无副作用
