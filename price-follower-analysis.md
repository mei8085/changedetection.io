# changedetection.io 价格数据 Follower 协同机制分析

> **分析对象**：RSS 蓝图、Watch 模型、处理器模块的价格变化跟踪协同
> **分析范围**：调度触发点、状态流转、跨模块依赖边界、异常回退路径
> **代码版本**：基于项目源码深度分析

---

## 目录

1. [系统架构总览](#1-系统架构总览)
2. [调度触发点详解](#2-调度触发点详解)
3. [Watch 模型状态流转](#3-watch-模型状态流转)
4. [处理器模块工作流](#4-处理器模块工作流)
5. [RSS 蓝图输出机制](#5-rss-蓝图输出机制)
6. [跨模块依赖边界](#6-跨模块依赖边界)
7. [关键异常与回退路径](#7-关键异常与回退路径)
8. [核心设计决策总结](#8-核心设计决策总结)

---

## 1. 系统架构总览

### 1.1 四层协同架构

```
┌───────────────────────────────────────────────────────────────┐
│                    RSS 蓝图 (输出层)                           │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │  主聚合 Feed     │  │  单 Watch Feed    │                   │
│  │  /rss/           │  │  /rss/watch/<uuid>│                   │
│  └──────────────────┘  └──────────────────┘                   │
└───────────────────────────────────┬───────────────────────────┘
                                    │  只读访问
┌───────────────────────────────────▼───────────────────────────┐
│                    Watch 模型 (状态层)                          │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  监控配置 + 历史快照 + restock{price,in_stock,original_..}│ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────────────┬───────────────────────────┘
                                    │  读/写更新
┌───────────────────────────────────▼───────────────────────────┐
│                   处理器模块 (检测层)                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  restock_diff + 价格提取三级降级 + 变更检测阈值逻辑       │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────────────┬───────────────────────────┘
                                    │  异步执行
┌───────────────────────────────────▼───────────────────────────┐
│                   调度系统 (执行层)                             │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────────────┐ │
│  │ 定时调度器   │→ │ 优先级队列     │→ │ 异步 Worker 池     │ │
│  │ (scheduler)  │  │ (priority Q)  │  │ (async workers)    │ │
│  └──────────────┘  └───────────────┘  └────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### 1.2 数据流向图

```
定时触发 / 手动触发
    ↓
[调度器] → 计算阈值 → 检查排队状态 → 加入优先级队列
    ↓
[Worker池] → 领取任务 → deepcopy Watch (防并发冲突)
    ↓
[处理器] → 抓取页面 → 提取价格/库存 → 变更检测
    ↓
[Watch模型] → 更新 last_checked / restock / previous_md5 → 保存快照
    ↓
[RSS蓝图] → 读取历史快照 → 生成变更 Feed → 输出 XML
```

---

## 2. 调度触发点详解

### 2.1 核心调度循环

**文件位置**：`flask_app.py:1174-1230`

```python
# 调度决策完整流程
def scheduler_loop():
    for uuid in datastore.data['watching']:
        # 1. 队列容量保护
        if current_queue_size >= MAX_QUEUE_SIZE:
            break
            
        watch = datastore.data['watching'][uuid]
        
        # 2. 暂停过滤
        if watch['paused']:
            continue
            
        # 3. 时间窗口调度 (基于时区)
        if watch['time_between_check_use_default']:
            time_schedule_limit = datastore.data['settings']['requests']['time_schedule_limit']
        else:
            time_schedule_limit = watch['time_schedule_limit']
            
        if time_schedule_limit and time_schedule_limit.get('enabled'):
            if not is_within_schedule(time_schedule_limit):
                continue
                
        # 4. 检查间隔阈值
        threshold = system_seconds if use_default else watch.threshold_seconds()
        
        # 5. Jitter 随机偏移 (防请求风暴)
        jitter = datastore.data['settings']['requests'].get('jitter_seconds', 0)
        if jitter > 0 and watch.jitter_seconds == 0:
            watch.jitter_seconds = random.uniform(-abs(jitter), jitter)
            
        seconds_since = now - watch['last_checked']
        
        # 6. 双重排队检查 (防重复)
        if (seconds_since >= threshold + watch.jitter_seconds and
            seconds_since >= recheck_time_minimum_seconds):
            if uuid not in running_uuids and uuid not in queued_uuids:
                # 7. 代理频率限制检查
                proxy = datastore.get_preferred_proxy_for_watch(uuid=uuid)
                queue_watch(uuid)
```

### 2.2 时间调度窗口

**文件位置**：`time_handler.py`, `model/__init__.py:235-279`

```python
# Watch 调度配置结构
time_schedule_limit = {
    "enabled": True,           # 总开关
    "timezone": "Asia/Shanghai",  # 时区
    
    # 每天独立配置
    "monday": {
        "enabled": True,
        "start_time": "09:00",    # 开始时间
        "duration": {             # 持续时长
            "hours": 8,
            "minutes": 0
        }                          # 09:00-17:00 工作时间
    },
    "tuesday": { ... },
    # ... 周六周日
}
```

**跨午夜支持**：
- 例：`start_time="23:00" + duration=120分钟` → 23:00-01:00
- 逻辑：当前时间 >= start_time 或 当前时间 <= start_time + duration

### 2.3 手动触发入口

#### API 触发
**文件位置**：`api/Watch.py:544-566`

```python
# 去重排队逻辑
queued_uuids = set(update_q.get_queued_uuids())
running_uuids = set(worker_pool.get_running_uuids())

watches_to_queue_filtered = [
    uuid for uuid in watches_to_queue
    if uuid not in queued_uuids and uuid not in running_uuids
]

for uuid in watches_to_queue_filtered:
    worker_pool.queue_item_async_safe(
        update_q,
        PrioritizedItem(priority=1, item={'uuid': uuid})
    )
```

#### UI 触发
**文件位置**：`realtime/socket_server.py:274-305`

- Socket.IO 接收 `checkbox-operation` 事件
- 后台 daemon 线程执行，不阻塞 WebSocket
- 支持批量操作：重新检查、暂停、静音

---

## 3. Watch 模型状态流转

### 3.1 核心数据结构

**文件位置**：`model/Watch.py`

```python
# 价格监控专用字段
class Watch(dict):
    uuid: str                          # 唯一标识
    url: str                           # 监控页面
    
    # ─── 调度相关 ───
    time_between_check: dict           # {hours, minutes, seconds...}
    time_between_check_use_default: bool
    time_schedule_limit: dict          # 时间窗口配置
    last_checked: int                  # Unix 时间戳
    jitter_seconds: float              # 调度偏移量 (-jitter ~ +jitter)
    
    # ─── 处理器相关 ───
    processor: str = "restock_diff"    # 处理器类型
    previous_md5: str                  # 内容哈希，快速跳过
    
    # ─── 价格/库存状态 ───
    restock: dict = {
        "price": float,                # 当前价格
        "original_price": float,       # 首次检测价格（基准）
        "in_stock": bool,              # 库存状态
        "currency": str,               # 货币符号
    }
    
    # ─── 处理器配置 ─── (独立 JSON 文件存储)
    # restock_diff.json:
    # {
    #   "restock_diff": {
    #       "in_stock_processing": "in_stock_only",
    #       "follow_price_changes": True,
    #       "price_change_min": 50.0,
    #       "price_change_max": 200.0,
    #       "price_change_threshold_percent": 5.0
    #   }
    # }
    
    # ─── 瞬时状态 (不持久化) ───
    __check_status: str                # "Fetching page.." / "Querying AI.."
    was_edited: bool                   # 配置变更标记
```

### 3.2 持久化边界控制

**文件位置**：`model/Watch.py:1066-1093`

```python
def _get_commit_data(self):
    """准备写入磁盘的数据"""
    snapshot = dict(self)
    
    # 过滤规则：
    # 1. processor_config_* 前缀 → 独立 JSON 文件
    # 2. __ 前缀 → 瞬时状态，不持久化
    watch_dict = {
        k: copy.deepcopy(v) for k, v in snapshot.items()
        if not k.startswith('processor_config_') and not k.startswith('__')
    }
    
    # Browser Steps 空列表规范化
    if not self.has_browser_steps:
        watch_dict['browser_steps'] = []
        
    return watch_dict
```

### 3.3 完整状态流转图

```
                              ┌─────────────┐
                              │   初始创建   │
                              │  last_checked=0
                              └──────┬──────┘
                                     │ 首次调度立即执行
                                     ▼
┌───────────────────────────────────────────────────────────────┐
│                        空闲等待中                               │
│  last_checked = T0                                             │
│  jitter_seconds = 0 或保持上次值                                │
└───────────────────────────────────┬───────────────────────────┘
                                     │
    ┌────────────────────────────────┼────────────────────────┐
    │                                │                        │
    │ T-T0 >= threshold + jitter    │ 手动立即检查            │ 配置编辑
    │ 且不在排队/处理中              │                        │
    ▼                                ▼                        ▼
┌─────────────┐                ┌─────────────┐          ┌─────────────┐
│  队列等待    │                │  队列等待    │          │  标记编辑    │
│  priority=N │                │  priority=1 │          │  was_edited=T│
└──────┬──────┘                └──────┬──────┘          └──────┬──────┘
       │                               │                        │
       │  Worker 领取                  │                        │
       ▼                               ▼                        │
┌─────────────────────────────────────────────────────┐         │
│                    处理中                            │         │
│  UUID 加入 currently_processing_uuids               │         │
│  ┌───────────────────────────────────────────────┐ │         │
│  │ 1. deepcopy Watch 配置 (防并发冲突)            │ │         │
│  │ 2. call_browser() 抓取页面                     │ │         │
│  │    → __check_status = "Fetching page.."        │ │◄────────┘
│  │ 3. 三级降级提取价格/库存                        │ │
│  │ 4. 变更检测逻辑                                 │ │
│  │ 5. LLM 意图过滤 (可选)                         │ │
│  │ 6. LLM 摘要生成 (可选)                         │ │
│  └───────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
    ┌───────────┐                   ┌───────────┐
    │  有变更   │                   │  无变更   │
    └────┬──────┘                   └────┬──────┘
         │                               │
         │  save_history_blob()          │  仅更新
         │  加入通知队列                 │  last_checked
         │  RSS Feed 自动更新            │  fetch_time
         ▼                               ▼
    ┌───────────────────────────────────────────────┐
    │              回到空闲等待状态                   │
    │  UUID 移出 currently_processing_uuids          │
    │  jitter_seconds 重置为 0 (下次重新生成)        │
    │  __check_status 清除                           │
    └───────────────────────────────────────────────┘
```

### 3.4 关键状态字段对照表

| 字段 | 来源 | 持久化 | 说明 |
|------|------|--------|------|
| `last_checked` | Worker | ✅ | 上次检查 Unix 时间戳 |
| `previous_md5` | 处理器 | ✅ | 快照内容哈希，快速跳过 |
| `restock.price` | restock_diff | ✅ | 当前提取价格 |
| `restock.original_price` | restock_diff | ✅ | 首次检测的基准价 |
| `restock.in_stock` | restock_diff | ✅ | 库存布尔状态 |
| `restock.currency` | restock_diff | ✅ | 货币符号 |
| `__check_status` | Worker | ❌ | 瞬时处理状态 |
| `jitter_seconds` | 调度器 | ❌ | 调度偏移量 |
| `check_count` | Worker | ✅ | 累计检查次数 |
| `fetch_time` | Worker | ✅ | 上次抓取耗时(秒) |
| `was_edited` | UI/API | ❌ | 配置变更标记，强制重处理 |

---

## 4. 处理器模块工作流

### 4.1 处理器调用链

**文件位置**：`worker.py:171-198`

```python
# Worker 中的处理器执行流程
async def process_watch(uuid):
    # 1. 处理器发现与实例化
    processor_name = watch.get('processor', 'text_json_diff')
    processor_module = get_processor_module(processor_name)
    update_handler = processor_module.perform_site_check(
        datastore=datastore,
        watch_uuid=uuid
    )
    
    # 2. 插件钩子：允许修改处理器
    update_handler = apply_update_handler_alter(update_handler, watch, datastore)
    
    # 3. 异步页面抓取
    await update_handler.call_browser()
    
    # 4. CPU 密集型：变更检测（线程池执行，不阻塞事件循环）
    changed_detected, update_obj, contents = await loop.run_in_executor(
        executor,
        lambda: update_handler.run_changedetection(watch=watch)
    )
```

### 4.2 restock_diff 价格提取三级降级

**文件位置**：`processors/restock_diff/processor.py:203-310`

```
                        ┌──────────────────────────┐
                        │   页面 HTML 内容获取      │
                        └────────────┬─────────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
                  ▼                  ▼                  ▼
        ┌──────────────────┐   ┌──────────────┐   ┌──────────┐
        │ Level 1: 纯 Python│   │ Level 2:     │   │ Level 3: │
        │ 元数据提取        │   │ extruct+lxml │   │ LLM AI   │
        │ (JSON-LD/OG)     │   │ (子进程隔离)  │   │ 提取     │
        └────────┬─────────┘   └──────┬───────┘   └────┬─────┘
                 │                     │                 │
                 │ 成功？               │ 成功？          │
                 └───┬─────────────────┴───┬─────────────┘
                     │ 失败                │ 失败
                     ▼                     ▼
              ┌─────────────────────────────────────────┐
              │        抛出：无法提取价格数据            │
              └─────────────────────────────────────────┘
```

#### Level 1：纯 Python 元数据提取

```python
# 快速路径：正则提取 JSON-LD / OpenGraph
# 无 lxml 依赖，无内存泄漏，80% 电商页面兼容
from pure_python_extractor import extract_metadata_pure_python, query_price_availability

extracted_data = extract_metadata_pure_python(html_content)
price_data = query_price_availability(extracted_data)

if price_data.get('price') and price_data.get('availability'):
    return Restock(price_data)  # 快速返回
```

#### Level 2：extruct + lxml (Linux 子进程隔离)

**文件位置**：`processors/restock_diff/processor.py:131-201`

```python
# 内存泄漏防护：lxml 的 C 级分配 Python GC 无法回收
# 解决方案：spawn 子进程，完成后自动退出，OS 回收全部内存

def _extract_itemprop_availability_worker(pipe_conn):
    # 子进程内执行
    html_bytes = pipe_conn.recv_bytes()
    html_content = html_bytes.decode('utf-8')
    
    result_data = get_itemprop_availability(html_content)  # 使用 extruct
    
    # 通过管道返回 JSON
    pipe_conn.send_bytes(json.dumps({"success": True, "data": dict(result_data)}).encode())
    # 子进程退出 → OS 自动回收所有内存包括 lxml 的 C 级分配

# 父进程调用
if platform.system() == 'Linux':
    ctx = multiprocessing.get_context('spawn')
    parent_conn, child_conn = ctx.Pipe()
    p = ctx.Process(target=_extract_itemprop_availability_worker, args=(child_conn,))
    p.start()
    
    parent_conn.send_bytes(html_content.encode('utf-8'))
    result_bytes = parent_conn.recv_bytes()
    result = json.loads(result_bytes.decode('utf-8'))
    
    p.join()
    parent_conn.close()
    child_conn.close()
    
    del p, parent_conn, child_conn
    gc.collect()  # 显式回收 Python 级对象
    
    return Restock(result['data'])
```

#### Level 3：LLM AI 价格提取（插件钩子）

```python
# 内建提取失败时的最后防线
from changedetectionio.llm.evaluator import resolve_intent

_llm_intent, _ = resolve_intent(watch, datastore)
plugin_availability = get_itemprop_availability_from_plugin(
    self.fetcher.content,
    fetcher_name,  # "html_requests" / "html_webdriver"
    self.fetcher,
    watch.link,
    llm_intent=_llm_intent
)

# 插件返回格式：
# {
#   "price": 99.99,
#   "availability": "InStock",
#   "_tokens": 450,           # token 消耗
#   "_input_tokens": 400,
#   "_output_tokens": 50,
#   "_model": "gpt-4o"
# }
```

### 4.3 价格变更检测核心逻辑

**文件位置**：`processors/restock_diff/processor.py:604-654`

```python
def run_changedetection(self, watch, force_reprocess=False):
    # ─── 1. Checksum 快速跳过 ───
    current_raw_checksum = self.get_raw_document_checksum()
    if (not force_reprocess and
        not watch.was_edited and
        self.last_raw_content_checksum and
        self.last_raw_content_checksum == current_raw_checksum):
        raise checksumFromPreviousCheckWasTheSame()
        
    # ─── 2. 提取价格/库存 ───
    itemprop_availability = {}
    try:
        itemprop_availability = extract_itemprop_availability_safe(self.fetcher.content)
    except MoreThanOnePriceFound:
        # 多价格 → 尝试插件兜底
        pass  # 插件后仍失败则抛异常
    
    # ─── 3. 存储原始基准价 ───
    if itemprop_availability.get('price') and not itemprop_availability.get('original_price'):
        itemprop_availability['original_price'] = itemprop_availability['price']
        update_obj['restock']['original_price'] = itemprop_availability['price']
        
    # ─── 4. 库存状态映射 ───
    if itemprop_availability.get('availability'):
        # Schema.org 标准值 → 布尔值
        if any(sub in itemprop_availability['availability'].lower() 
               for sub in ['instock', 'instockonly', 'limitedavailability',
                          'onlineonly', 'presale']):
            update_obj['restock']['in_stock'] = True
        else:
            update_obj['restock']['in_stock'] = False
            
    # ─── 5. 浏览器 JS 库存检测兜底（当元数据缺失时） ───
    if self.fetcher.instock_data and itemprop_availability.get('availability') is None:
        update_obj['restock']['in_stock'] = (self.fetcher.instock_data == 'Possibly in stock')
        
    # ─── 6. 防说谎机制：页面文本检测优先级高于元数据 ───
    if self.fetcher.instock_data and self.fetcher.instock_data != 'Possibly in stock':
        if update_obj['restock'].get('in_stock'):
            # 元数据说有货，但页面文本说缺货 → 相信页面文本
            update_obj['restock']['in_stock'] = False
            
    # ─── 7. 库存变更检测 ───
    changed_detected = False
    
    if watch.get('restock') and watch['restock'].get('in_stock') != update_obj['restock'].get('in_stock'):
        # 模式 A：仅缺货→到货触发
        if restock_settings.get('in_stock_processing') == 'in_stock_only':
            if update_obj['restock']['in_stock']:
                changed_detected = True
        # 模式 B：任何库存变更都触发
        elif restock_settings.get('in_stock_processing') == 'all_changes':
            changed_detected = True
            
    # ─── 8. 价格变更检测 ───
    if restock_settings.get('follow_price_changes'):
        if watch.get('restock') and watch['restock'].get('original_price'):
            original_price = float(watch['restock']['original_price'])
            current_price = float(update_obj['restock']['price'])
            
            if current_price != original_price:
                changed_detected = True
                
                # 8a. 价格范围过滤
                min_limit = float(restock_settings.get('price_change_min')) if restock_settings.get('price_change_min') else None
                max_limit = float(restock_settings.get('price_change_max')) if restock_settings.get('price_change_max') else None
                
                if min_limit or max_limit:
                    if is_between(current_price, min_limit, max_limit):
                        # 在允许范围内 → 不告警
                        changed_detected = False
                    else:
                        # 超出范围 → 告警
                        changed_detected = True
                        
                # 8b. 百分比阈值过滤
                if changed_detected and restock_settings.get('price_change_threshold_percent'):
                    threshold = float(restock_settings.get('price_change_threshold_percent'))
                    change_pct = abs((current_price - original_price) / original_price * 100)
                    
                    if change_pct <= threshold:
                        # 变化幅度在阈值内 → 不告警
                        changed_detected = False
                        
    # ─── 9. 生成快照内容（用于哈希） ───
    snapshot_content = f"In Stock: {update_obj['restock'].get('in_stock')} - Price: {update_obj['restock'].get('price')}"
    update_obj['previous_md5'] = hashlib.md5(snapshot_content.encode('utf-8')).hexdigest()
    
    return changed_detected, update_obj, snapshot_content.strip()
```

### 4.4 处理器配置解析链

**文件位置**：`processors/restock_diff/processor.py:455-467`

```python
# 配置优先级（从高到低）：
# 1. Watch 独立 JSON 配置 (restock_diff.json)
# 2. 标签 Tag 配置 (需 overrides_watch=True)
# 3. 处理器默认值

def get_restock_settings(watch, datastore):
    # 1. 读取 Watch 独立配置文件
    extra_config = self.get_extra_watch_config('restock_diff.json')
    restock_settings = extra_config.get('restock_diff', {})
    
    # 2. 检查是否有标签覆盖
    for tag_uuid in watch.get('tags'):
        tag = datastore.data['settings']['application']['tags'].get(tag_uuid, {})
        if tag.get('overrides_watch'):
            restock_settings = tag.get('processor_config_restock_diff', {})
            logger.info(f"Watch {uuid} - 使用标签 '{tag.get('title')}' 的覆盖配置")
            break
            
    # 3. 应用默认值
    defaults = {
        'follow_price_changes': True,
        'in_stock_processing': 'in_stock_only',
    }
    for k, v in defaults.items():
        if k not in restock_settings:
            restock_settings[k] = v
            
    return restock_settings
```

---

## 5. RSS 蓝图输出机制

### 5.1 路由结构

**文件位置**：`blueprint/rss/`

| 路由 | 功能 | Token 鉴权 |
|------|------|-----------|
| `/rss/` | 主聚合 Feed，所有未读变更 | ✅ |
| `/rss/watch/<uuid>` | 单 Watch 完整历史变更 | ✅ |
| `/rss/tag/<tag>` | 按标签过滤的 Feed | ✅ |

### 5.2 Feed 生成流程

**文件位置**：`rss/main_feed.py:21-104`, `rss/single_watch.py:81-115`

```python
def generate_rss_feed():
    # ─── 1. Token 鉴权 ───
    is_valid, error = validate_rss_token(datastore, request)
    if not is_valid:
        return error
        
    # ─── 2. 状态过滤 ───
    filtered_watches = []
    for uuid, watch in datastore.data['watching'].items():
        # 跳过静音
        if datastore.data['settings']['application'].get('rss_hide_muted_watches'):
            if watch.get('notification_muted'):
                continue
                
        # 标签过滤
        if limit_tag and limit_tag not in watch['tags']:
            continue
            
        # 至少 2 个快照才有变更可显示
        if len(watch.history.keys()) < 2:
            continue
            
        filtered_watches.append(watch)
        
    # ─── 3. 按时间排序 ───
    filtered_watches.sort(key=lambda x: x.last_changed, reverse=False)
    
    # ─── 4. 构建 Notification 上下文 ───
    notification_service = NotificationService(datastore=datastore, notification_q=False)
    
    for watch in filtered_watches:
        if not watch.viewed:
            timestamp_to = dates[-1]
            timestamp_from = dates[-2]
            
            n_body_template = get_rss_template(datastore, watch, rss_content_format,
                                              RSS_TEMPLATE_HTML_DEFAULT, RSS_TEMPLATE_PLAINTEXT_DEFAULT)
            
            n_object = build_notification_context(
                watch, timestamp_from, timestamp_to,
                watch_label, n_body_template, rss_content_format
            )
            
            # ─── 5. 渲染通知模板 ───
            res = render_notification(n_object, notification_service, watch, datastore)
            
            # 价格监控专用字段注入：
            # {{ restock_price }} - 当前价格
            # {{ restock_in_stock }} - 库存状态
            # {{ restock_price_changed }} - 价格变化方向
            
            # ─── 6. 生成 Feed 条目 ───
            fe = fg.add_entry()
            guid = generate_watch_guid(watch, timestamp_to)  # uuid/timestamp 唯一性
            populate_feed_entry(fe, watch, res['body'], guid, timestamp_to)
            add_watch_categories(fe, watch, datastore)  # 标签作为 RSS <category>
            
    # ─── 7. 输出 XML ───
    response = make_response(fg.rss_str())
    response.headers.set('Content-Type', 'application/rss+xml;charset=utf-8')
    return response
```

### 5.3 单 Watch 历史 Feed

**文件位置**：`rss/single_watch.py:53-59`

```python
# 显示最近 N 个变更
rss_diff_length = datastore.data['settings']['application'].get('rss_diff_length', 5)
max_possible_diffs = len(dates) - 1
num_diffs = min(rss_diff_length, max_possible_diffs) if rss_diff_length > 0 else max_possible_diffs

# 倒序遍历：最新变更排在最前
for i in range(num_diffs - 1, -1, -1):
    date_index_to = -(i + 1)
    date_index_from = -(i + 2)
    timestamp_to = dates[date_index_to]
    timestamp_from = dates[date_index_from]
    
    # 生成每个历史版本的差异对比
    res = render_notification(..., date_index_from, date_index_to)
    
    guid = f"{uuid}/{timestamp_to}"  # 每个版本独立 GUID
    fe = fg.add_entry()
    populate_feed_entry(fe, watch, res['body'], guid, timestamp_to, ...)
```

---

## 6. 跨模块依赖边界

### 6.1 依赖方向图

```
┌──────────┐                   ┌──────────┐                   ┌──────────────┐
│ RSS 蓝图 │◄──────────────────│ Watch 模型│◄──────────────────│  处理器模块  │
└──────────┘    只读访问        └──────────┘    读/写更新       └──────┬───────┘
     ↑                                                                 │
     │                        只读访问                                 │
     └─────────────────────────────────────────────────────────────────┘
                    
┌──────────┐                   ┌──────────┐                   ┌──────────────┐
│ 调度器   │──────────────────►│  队列    │──────────────────►│ Worker池     │
└──────────┘     任务入队       └──────────┘    任务领取       └──────┬───────┘
                                                                     │
                                                                     ▼
                                                               ┌──────────┐
                                                               │ 处理器   │
                                                               └──────────┘
```

### 6.2 关键边界约束表

| 边界 | 约束 | 实现方式 | 文件位置 |
|------|------|----------|----------|
| 调度器 → Worker | 仅通过队列传递任务 | `PrioritizedItem` + 仅传 UUID | `worker_pool.py` |
| Worker → 处理器 | 只读访问配置 | `deepcopy(watch)` 防止并发修改 | `worker.py:38` |
| 处理器 → Watch | 仅通过 `update_obj` 返回变更 | `datastore.update_watch()` 统一入口 | `worker.py:500+` |
| RSS → 所有模块 | 纯只读，不修改状态 | 直接访问 `datastore.data`，无写操作 | `rss/*.py` |
| 处理器配置 | 独立 JSON 文件 | `restock_diff.json` 不混入主 JSON | `processors/base.py:274-350` |
| 瞬时状态 | 不持久化 | `__` 前缀 + `_get_commit_data()` 过滤 | `model/Watch.py:1066-1093` |

### 6.3 配置优先级链

```
处理器配置解析顺序 (从高到低)：

┌─────────────────────────────────────────────────────────┐
│ 1. Watch 自身 processor_config_restock_diff (独立 JSON) │
│    文件：{datastore}/<watch-uuid>/restock_diff.json      │
└───────────────────────────┬─────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Tag 标签覆盖配置 (需 overrides_watch=True)            │
│    用途：批量配置一组 Watch (如"亚马逊商品"统一阈值)      │
└───────────────────────────┬─────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────┐
│ 3. 处理器默认值                                          │
│    follow_price_changes: True                            │
│    in_stock_processing: "in_stock_only"                 │
└─────────────────────────────────────────────────────────┘
```

---

## 7. 关键异常与回退路径

### 7.1 处理器异常层级

```
┌─────────────────────────────────────────────────────────┐
│ checksumFromPreviousCheckWasTheSame                     │
│ └─ 快速跳过：内容未变 + 配置未编辑                       │
├─────────────────────────────────────────────────────────┤
│ MoreThanOnePriceFound                                    │
│ └─ 页面检测到多个价格 → 尝试 LLM 插件 → 仍失败抛异常      │
├─────────────────────────────────────────────────────────┤
│ ProcessorException                                       │
│ └─ 通用处理器异常（无法提取、网络失败等）                │
├─────────────────────────────────────────────────────────┤
│ Browser Steps 相关异常                                   │
│ ├─ BrowserStepsStepException: 某步骤失败                 │
│ └─ BrowserStepsInUnsupportedFetcher: 需 Chrome 浏览器    │
├─────────────────────────────────────────────────────────┤
│ 内容抓取相关异常                                         │
│ ├─ BrowserConnectError: 浏览器无法连接                   │
│ ├─ BrowserFetchTimedOut: 抓取超时                       │
│ └─ ReplyWithContentButNoText: 抓取成功但无可用文本       │
└─────────────────────────────────────────────────────────┘
```

### 7.2 价格提取失败回退链

**文件位置**：`processors/restock_diff/processor.py:485-534`

```
HTML 内容
    │
    ▼
┌──────────────────┐
│ 纯 Python 提取   │ → 成功? ──┐
└────────┬─────────┘            │
         │ 失败                  │
         ▼                       │
┌──────────────────┐            │
│ extruct + lxml   │ → 成功? ──┤
│ (Linux 子进程)   │            │
└────────┬─────────┘            │
         │ 失败                  │
         ▼                       │
┌──────────────────┐            │
│ LLM AI 插件      │ → 成功? ──┤
│ (GPT-4o/Claude)  │            │
└────────┬─────────┘            │
         │ 失败                  │
         ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│ ProcessorException│    │ 使用提取结果     │
│ 终止本次检查      │    │ 继续变更检测     │
└──────────────────┘    └──────────────────┘
```

### 7.3 Worker 异常处理流程

**文件位置**：`worker.py:199-400`

```python
try:
    await update_handler.call_browser()
    changed_detected, update_obj, contents = await loop.run_in_executor(...)
    
except checksumFromPreviousCheckWasTheSame:
    # 正常跳过路径
    watch.reset_watch_edited_flag()
    datastore.update_watch(uuid=uuid, update_obj={'last_error': False})
    cleanup_error_artifacts(uuid, datastore)
    changed_detected = False
    
except BrowserConnectError as e:
    # 浏览器连接失败
    datastore.update_watch(uuid=uuid, update_obj={'last_error': e.msg})
    process_changedetection_results = False
    
except BrowserFetchTimedOut as e:
    # 抓取超时
    datastore.update_watch(uuid=uuid, update_obj={'last_error': e.msg})
    process_changedetection_results = False
    
except BrowserStepsStepException as e:
    # 某 Browser Step 失败
    step_n = e.step_n + 1
    watch['browser_steps_last_error_step'] = step_n
    datastore.update_watch(uuid=uuid, update_obj={
        'last_error': f"Browser Steps 失败（步骤 #{step_n}）"
    })
    process_changedetection_results = False
    
except BrowserStepsInUnsupportedFetcher as e:
    # 当前 fetcher 不支持 Browser Steps（需要 Chrome）
    datastore.update_watch(uuid=uuid, update_obj={
        'last_error': "需要 Chrome 浏览器执行 Browser Steps"
    })
    process_changedetection_results = False
    
except Exception as e:
    # 兜底异常：记录完整堆栈
    logger.error(f"Worker 处理 UUID {uuid} 异常")
    logger.exception("完整堆栈信息")
    datastore.update_watch(uuid=uuid, update_obj={'last_error': str(e)})
    process_changedetection_results = False
    
finally:
    # 无论成功失败，必须释放 UUID
    worker_pool.release_uuid_from_processing(uuid, worker_id=worker_id)
    watch.pop('__check_status', None)  # 清除瞬时状态
```

### 7.4 多处理器发现机制

**文件位置**：`processors/__init__.py:20-68`

```python
@lru_cache(maxsize=1)
def find_processors():
    """处理器自动发现（支持插件扩展）"""
    package_name = "changedetectionio.processors"
    processors = []
    
    # 1. 内建处理器扫描
    sub_packages = find_sub_packages(package_name)
    from .base import difference_detection_processor
    
    for sub_package in sub_packages:
        module_name = f"{package_name}.{sub_package}.processor"
        try:
            module = importlib.import_module(module_name)
            for name, obj in inspect.getmembers(module, inspect.isclass):
                if (issubclass(obj, difference_detection_processor) and
                    obj is not difference_detection_processor and
                    obj.__module__ == module.__name__):
                    processors.append((module, sub_package))
                    break
        except (ModuleNotFoundError, ImportError) as e:
            logger.warning(f"导入处理器 {sub_package} 失败: {e}")
            
    # 2. 插件处理器（pluggy 钩子）
    try:
        from pluggy_interface import plugin_manager
        plugin_results = plugin_manager.hook.register_processor()
        for result in plugin_results:
            if result and isinstance(result, dict):
                processor_module = result.get('processor_module')
                processor_name = result.get('processor_name')
                if processor_module and processor_name:
                    processors.append((processor_module, processor_name))
    except Exception as e:
        logger.warning(f"加载插件处理器失败: {e}")
        
    return processors
```

---

## 8. 核心设计决策总结

### 8.1 内存管理：Linux 子进程隔离

**问题背景**：
- lxml 是 C 扩展库，其节点分配在 C 堆上
- Python GC 只能回收 Python 对象，无法回收 C 级内存
- 长期运行的 Worker 进程内存会持续增长

**解决方案**：
```
平台 == Linux
    ├─ 使用 multiprocessing.get_context('spawn')
    ├─ 创建独立子进程执行 lxml 解析
    └─ 完成后子进程退出 → OS 强制回收 ALL 内存（包括 C 堆）

平台 != Linux (Windows/macOS)
    └─ 直接在主进程调用（内存泄漏风险，接受这个妥协）
```

**替代方案**：纯 Python 正则提取 JSON-LD（覆盖 80% 电商页面）

### 8.2 防并发冲突：deepcopy + UUID 双重声明

**问题背景**：
- Worker 异步处理时，用户可能同时编辑 Watch 配置
- 多 Worker 同时处理同一 Watch 导致状态混乱

**解决方案**：
1. **处理器实例化时 deepcopy**：`self.watch = deepcopy(datastore.data['watching'].get(uuid))`
2. **队列取出后立即声明**：

```python
# worker.py:90-100
uuid = queued_item_data.item.get('uuid')
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # 已在处理，重新排队 + 降低优先级
    deferred_priority = max(1000, queued_item_data.priority * 10)
    await asyncio.sleep(DEFER_SLEEP_TIME_ALREADY_QUEUED)
    worker_pool.queue_item_async_safe(q, deferred_item, silent=True)
    continue
```

### 8.3 分层降级：从快到慢、从准到稳

```
价格提取三级降级设计原则：
┌─────────────────────────────────────────────────────────┐
│ Level 1: 80% 场景最快路径（纯 Python，无依赖）           │
│ Level 2: 15% 场景完整解析（lxml，内存隔离）              │
│ Level 3: 最后 5% 兜底（AI 大模型，成本高但泛化强）       │
└─────────────────────────────────────────────────────────┘

适用场景：
- 亚马逊、京东等标准化电商 → Level 1 搞定
- 小众网站、复杂页面 → Level 2 处理
- 反爬严格、动态渲染 → Level 3 LLM 理解
```

### 8.4 瞬时状态不持久化：`__` 前缀约定

```python
# 设计原则：
# ────────────────────────────────────────────────────────
# 1. __ 前缀字段仅在内存中存在，不写入磁盘
# 2. _get_commit_data() 在保存前自动过滤
# 3. 用途：
#    - __check_status: 前端显示处理进度
#    - was_edited: 配置变更标记，强制重处理
#    - 各种运行时临时状态

# 好处：
# - 持久化数据干净，无运行时噪声
# - 重启后状态干净（不恢复临时状态）
# - 避免磁盘 I/O 频繁写入瞬时值
```

---

## 附录：关键文件索引

| 模块 | 文件 | 关键行数 |
|------|------|----------|
| Watch 模型 | `changedetectionio/model/Watch.py` | ~1300 |
| Worker 执行 | `changedetectionio/worker.py` | ~790 |
| Worker 池 | `changedetectionio/worker_pool.py` | ~550 |
| 价格处理器 | `changedetectionio/processors/restock_diff/processor.py` | ~660 |
| 调度主循环 | `changedetectionio/flask_app.py:1174-1230` | ~60 |
| 时间处理 | `changedetectionio/time_handler.py` | ~800 |
| RSS 主 Feed | `changedetectionio/blueprint/rss/main_feed.py` | ~100 |
| 单 Watch RSS | `changedetectionio/blueprint/rss/single_watch.py` | ~115 |
| 处理器基类 | `changedetectionio/processors/base.py` | ~360 |
| 处理器发现 | `changedetectionio/processors/__init__.py` | ~190 |

---

**分析完成时间**：2026-05-16
**分析版本**：基于项目当前主干版本
