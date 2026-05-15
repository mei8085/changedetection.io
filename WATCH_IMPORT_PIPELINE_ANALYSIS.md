# 监控列表导入链路分析报告

## 概述

changedetection.io 的监控列表导入流程包含四个核心阶段：**解析** → **归并** → **入库** → **调度**。本文档详细分析每个阶段的容错策略、字段对齐方式、批量处理行为及合并语义，重点澄清不同入口的差异、合并逻辑及入库与调度的分离。

---

## 一、解析阶段：不同入口的重复监控处理差异

### 1.1 入口渠道总览

| 入口类型 | 位置 | 数据源格式 | 重复检测 | 处理方式 |
|---------|------|-----------|---------|---------|
| **API批量导入** | `api/Import.py` | 纯文本URL列表 | **有** | 跳过已存在项 |
| **网页URL列表导入** | `blueprint/imports/importer.py` | URL列表 | 无 | 直接创建（可能重复） |
| **网页Distill.io导入** | `blueprint/imports/importer.py` | JSON | 无 | 直接创建（可能重复） |
| **网页Wachete导入** | `blueprint/imports/importer.py` | XLSX | 无 | 直接创建（可能重复） |
| **网页自定义XLSX导入** | `blueprint/imports/importer.py` | XLSX | 无 | 直接创建（可能重复） |
| **命令行** | `__init__.py` | `-u`参数 | 无 | 直接创建（可能重复） |
| **CreateWatch API** | `api/Watch.py` | JSON | 无 | 直接创建（可能重复） |

### 1.2 各入口重复处理详解

#### 1.2.1 API批量导入（唯一有去重的入口）

```python
# api/Import.py:121-122
dedupe = strtobool(request.args.get('dedupe', 'true'))

# api/Import.py:179-195
urls = request.get_data().decode('utf8').splitlines()
urls_to_import = []
for url in urls:
    url = url.strip()
    if not len(url):
        continue

    if not is_safe_valid_url(url):
        return f"Invalid or unsupported URL - {url}", 400

    # 关键：去重检查
    if dedupe and self.datastore.url_exists(url):
        continue  # 跳过已存在的URL

    urls_to_import.append(url)
```

**去重特性**：
- **默认开启**：`dedupe=true`，可通过参数关闭
- **比较方式**：不区分大小写
- **跳过策略**：已存在的URL直接跳过，不创建也不更新

#### 1.2.2 网页导入（无去重）

```python
# blueprint/imports/importer.py:47-76
for url in urls:
    url = url.strip()
    if not len(url):
        continue

    tags = ""
    if ' ' in url:
        url, tags = url.split(" ", 1)

    # 无去重检查！直接调用add_watch
    if len(url) and 'http' in url.lower() and good < 5000:
        extras = None
        if processor:
            extras = {'processor': processor}
        new_uuid = datastore.add_watch(url=url.strip(), tag=tags, 
                                      save_immediately=False, extras=extras)
```

#### 1.2.3 命令行导入（无去重）

```python
# __init__.py:421-426
for idx, url in enumerate(urls_to_add):
    extras = url_options.get(idx, {})
    new_uuid = datastore.add_watch(url=url, extras=extras)
    if new_uuid:
        added_watch_uuids.append(new_uuid)
```

#### 1.2.4 CreateWatch API（无去重）

```python
# api/Watch.py:489
new_uuid = self.datastore.add_watch(url=url, extras=extras, tag=tags)
```

### 1.3 去重检测底层实现

```python
# store/__init__.py:660-666
def url_exists(self, url):
    for watch in self.data['watching'].values():
        if watch['url'].lower() == url.lower():
            return True
    return False
```

**特点**：
- 仅比较URL，不比较其他属性
- O(n)复杂度，遍历所有现有监控项

---

## 二、归并阶段：既有监控项与新增监控项的合并语义

### 2.1 URL合并语义

| 场景 | 行为 | 说明 |
|-----|------|-----|
| URL已存在 + API导入(dedupe=true) | **跳过** | 不创建、不更新、不合并 |
| URL已存在 + 其他入口 | **创建重复项** | 生成新UUID，产生重复监控 |
| URL不存在 | **创建新项** | 正常流程 |

**核心原则**：系统**不执行URL级别的合并更新**。若需更新现有监控，必须通过编辑入口（PUT /api/v1/watch/{uuid}）。

### 2.2 标签合并语义

```python
# store/__init__.py:751-766
if tag and type(tag) == str:
    # 标签名称转换为UUID
    for t in tag.split(','):
        for a_t in t.split(','):
            tag_uuid = self.add_tag(a_t)  # 不存在则创建
            apply_extras['tags'].append(tag_uuid)

if tag_uuids:
    # 直接使用UUID
    for t in tag_uuids:
        apply_extras['tags'] = list(set(apply_extras['tags'] + [t.strip()]))

# 最终去重
if apply_extras.get('tags'):
    apply_extras['tags'] = list(set(apply_extras.get('tags')))
```

**合并规则**：
1. **名称→UUID转换**：输入标签名称自动转换为UUID，不存在则创建
2. **自动去重**：最终标签列表通过 `set()` 去重
3. **合并模式**：新标签追加到列表，不覆盖现有标签（适用于编辑场景）

### 2.3 扩展配置(extras)合并语义

```python
# store/__init__.py:679-680
apply_extras = deepcopy(extras)

# store/__init__.py:783
new_watch.update(apply_extras)
```

**合并规则**：
- **新增监控**：extras直接应用到新创建的Watch对象
- **无冲突处理**：因为URL重复时要么跳过要么创建新项，不存在配置冲突场景
- **特殊字段过滤**：以下字段会被自动移除（系统管理）：
  ```python
  # store/__init__.py:776-778
  for k in ['uuid', 'history', 'last_checked', 'last_changed', 
            'newest_history_key', 'previous_md5', 'viewed']:
      if k in apply_extras:
          del apply_extras[k]
  ```

### 2.4 共享链接解析的配置合并

```python
# store/__init__.py:684-728
if (url.startswith("https://changedetection.io/share/")):
    r = requests.request(method="GET", url=url, timeout=5.0)
    res = r.json()
    
    # 白名单属性合并
    for k in ['body', 'browser_steps', 'css_filter', 'extract_text', 
              'headers', 'ignore_text', 'include_filters', 'method', 
              'paused', 'previous_md5', 'processor', 'subtractive_selectors',
              'tag', 'tags', 'text_should_not_be_present', 'title', 
              'trigger_text', 'url', 'use_page_title_in_list', 
              'webdriver_js_execute_code']:
        if res.get(k):
            if k != 'css_filter':
                apply_extras[k] = res[k]
            else:
                apply_extras['include_filters'] = [res['css_filter']]
```

**安全策略**：
- **白名单限制**：仅接受预定义的属性列表
- **字段映射**：`css_filter` → `include_filters`（字段重命名兼容）
- **超时保护**：5秒超时防止阻塞

---

## 三、入库阶段：持久化机制

### 3.1 入库核心流程

```python
# store/__init__.py:674-794
def add_watch(self, url, tag='', extras=None, tag_uuids=None, save_immediately=True):
    # 1. 参数准备（深拷贝避免引用问题）
    apply_extras = deepcopy(extras)
    
    # 2. URL安全验证
    if not is_safe_valid_url(url):
        return None
    
    # 3. 数量限制检查（PAGE_WATCH_LIMIT）
    page_watch_limit = os.getenv('PAGE_WATCH_LIMIT')
    if page_watch_limit and current_watch_count >= page_watch_limit:
        return None
    
    # 4. 标签处理（名称转UUID）
    if tag and type(tag) == str:
        for t in tag.split(','):
            tag_uuid = self.add_tag(t)
            apply_extras['tags'].append(tag_uuid)
    
    # 5. 创建Watch对象
    watch_class = get_custom_watch_obj_for_processor(apply_extras.get('processor'))
    new_watch = watch_class(datastore_path=self.datastore_path, 
                           __datastore=self.__data, url=url)
    
    # 6. 应用配置
    new_watch.update(apply_extras)
    new_watch.ensure_data_dir_exists()
    
    # 7. 注册到内存
    self.__data['watching'][new_uuid] = new_watch
    
    # 8. 持久化（可延迟）
    if save_immediately:
        new_watch.commit()
    
    return new_uuid
```

### 3.2 立即保存 vs 延迟保存

| 模式 | `save_immediately` | 适用场景 | 调用方 |
|-----|-------------------|---------|-------|
| 立即保存 | `True`（默认） | 单条添加 | CreateWatch API、命令行 |
| 延迟保存 | `False` | 批量导入 | 网页导入 |

```python
# blueprint/imports/importer.py:65
new_uuid = datastore.add_watch(url=url.strip(), tag=tags, 
                                save_immediately=False, extras=extras)
```

### 3.3 原子写入机制

```python
# store/file_saving_datastore.py:36-175
def save_json_atomic(file_path, data_dict, label="file", max_size_mb=10):
    """
    原子写入流程：
    1. 创建临时文件（同一目录）
    2. 写入数据
    3. 可选fsync（FORCE_FSYNC_DATA_IS_CRITICAL=true时启用）
    4. 原子rename替换原文件
    5. 新文件时fsync目录确保文件名持久化
    """
    fd, temp_path = tempfile.mkstemp(suffix='.tmp', prefix='json-', dir=parent_dir)
    os.write(fd, data)
    if FORCE_FSYNC_DATA_IS_CRITICAL:
        os.fsync(fd)
    os.close(fd)
    os.replace(temp_path, file_path)
```

### 3.4 存储结构

```
datastore/
├── changedetection.json      # 全局配置
├── {watch-uuid}/
│   ├── watch.json            # 监控项配置（原子写入）
│   ├── history.txt           # 历史索引
│   ├── {timestamp}.txt(.br)  # 快照内容（Brotli压缩）
│   ├── last-screenshot.png   # 截图
│   └── last-error.txt        # 错误信息
└── {tag-uuid}/
    └── tag.json              # 标签配置
```

---

## 四、调度阶段：入库与入队的分离

### 4.1 核心原则：入库 ≠ 入队

**入库**：将监控项持久化到磁盘并注册到内存  
**入队**：将监控项添加到调度队列等待检查  

这是两个**独立**的操作，入库完成后**不会自动入队**。

### 4.1.1 入队判定流程图

```
                              ┌─────────────────────────────┐
                              │      触发入队请求           │
                              └───────────┬─────────────────┘
                                          │
                              ┌───────────▼─────────────────┐
                              │   全局暂停检查(all_paused)   │──Yes──▶ 拒绝入队
                              └───────────┬─────────────────┘
                                         No│
                              ┌───────────▼─────────────────┐
                              │   单项暂停检查(watch.paused) │──Yes──▶ 拒绝入队
                              └───────────┬─────────────────┘
                                         No│
                              ┌───────────▼─────────────────┐
                              │   时间窗口检查(is_in_schedule)│──No──▶ 拒绝入队
                              └───────────┬─────────────────┘
                                        Yes│
                              ┌───────────▼─────────────────┐
                              │  运行中/已入队去重检查      │──Yes──▶ 拒绝入队
                              └───────────┬─────────────────┘
                                         No│
                              ┌───────────▼─────────────────┐
                              │   代理复用冷却检查           │──No──▶ 拒绝入队
                              └───────────┬─────────────────┘
                                        Yes│
                              ┌───────────▼─────────────────┐
                              │    队列入队                   │
                              └─────────────────────────────┘
```

### 4.2 入队触发条件

| 触发场景 | 代码位置 | 优先级 | 条件 |
|---------|---------|-------|-----|
| **编辑保存** | `blueprint/ui/edit.py:275-277` | 1 | 非暂停状态 + 在时间窗口内 |
| **手动触发检查** | `api/Watch.py:81` | 1 | `?recheck=true` 参数 |
| **批量重检查** | `api/Watch.py:536-589` | 1 | `?recheck_all=true` 参数 |
| **命令行批量模式** | `__init__.py:437-445` | 1 | `-b` + `-u` 参数 |
| **定时调度** | ticker线程 | timestamp | 达到检查时间 |

### 4.3 各入口的入队行为

#### 4.3.1 API批量导入（不入队）

```python
# api/Import.py:198-227
if len(urls_to_import) < IMPORT_SWITCH_TO_BACKGROUND_THRESHOLD:
    added = []
    for url in urls_to_import:
        new_uuid = self.datastore.add_watch(...)
        added.append(new_uuid)
    return added, 200  # 仅返回UUID，不入队
else:
    # 后台线程导入，同样不入队
    thread = threading.Thread(target=import_watches_background, ...)
    thread.start()
    return {'status': 'Importing in background'}, 202
```

#### 4.3.2 网页导入（不入队）

```python
# blueprint/imports/importer.py:65-70
new_uuid = datastore.add_watch(url=url.strip(), tag=tags, 
                                save_immediately=False, extras=extras)
if new_uuid:
    self.new_uuids.append(new_uuid)  # 仅记录UUID，不入队
    good += 1
```

#### 4.3.3 命令行（条件入队）

```python
# __init__.py:430-446
# Step 2: Queue newly added watches (if -u was provided in batch mode)
if batch_mode and added_watch_uuids:
    from changedetectionio.flask_app import update_q
    from changedetectionio import queuedWatchMetaData, worker_pool

    for watch_uuid in added_watch_uuids:
        worker_pool.queue_item_async_safe(
            update_q,
            queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid})
        )
```

**条件**：必须同时满足 `-b`（批量模式）和 `-u`（添加URL）

#### 4.3.4 编辑保存（条件入队）

```python
# blueprint/ui/edit.py:275-277
if not datastore.data['watching'][uuid].get('paused') and is_in_schedule:
    worker_pool.queue_item_async_safe(update_q, 
        queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': uuid}))
```

**条件**：
1. 监控未暂停 (`paused=False`)
2. 当前时间在调度窗口内 (`is_in_schedule=True`)

#### 4.3.5 CreateWatch API（不入队）

```python
# api/Watch.py:491-496
new_uuid = self.datastore.add_watch(url=url, extras=extras, tag=tags)
# Dont queue because the scheduler will check that it hasnt been checked before anyway
# worker_pool.queue_item_async_safe(...)
return {'uuid': new_uuid}, 201
```

**设计意图**：调度器会自动检查新监控项并安排首次检查

### 4.4 批量导入的入队时机

批量导入（API或网页）完成后，新监控项的首次检查由**调度器（ticker线程）**负责，而非导入流程本身：

```
导入完成 → 监控项入库 → 调度器轮询 → 发现新项(last_checked=0) → 安排首次检查
```

调度器逻辑：
- 定期扫描所有监控项
- 检查 `last_checked` 是否为0（从未检查过）
- 若是，则计算下次检查时间并加入队列

### 4.5 队列优先级体系

```python
# custom_queue.py
# priority=1:     立即检查（最高优先级）
# priority=5:     克隆操作
# priority>100:   定时任务（使用timestamp作为优先级）
```

| 优先级值 | 含义 | 场景 |
|---------|-----|-----|
| 1 | 立即执行 | 编辑保存、手动触发、API调用 |
| 5 | 中等优先级 | 克隆操作 |
| timestamp | 定时执行 | 调度器安排的周期性检查 |

### 4.8 优先级队列实现

```python
# queue_handlers.py:15-411
class RecheckPriorityQueue:
    def __init__(self, maxsize: int = 0):
        # 优先级存储（最小堆）
        self._priority_items = []
        self._lock = threading.RLock()
        
        # 通知队列（用于唤醒workers）
        self._notification_queue = queue.Queue(maxsize=maxsize)
    
    def put(self, item, block=True, timeout=None):
        with self._lock:
            heapq.heappush(self._priority_items, item)
            self._notification_queue.put(True, block=True, timeout=5.0)
```

---

## 五、完整链路流程图

```
用户提交监控列表
        │
        ▼
┌─────────────────────┐
│   1. 解析阶段        │
│  - URL验证          │
│  - [仅API]去重检测   │
│  - 参数类型转换      │
│  - 字段映射对齐      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   2. 归并阶段        │
│  - URL存在性判断     │
│  - 标签合并去重      │
│  - 扩展配置应用      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   3. 入库阶段        │
│  - 创建Watch对象    │
│  - 应用配置         │
│  - 原子写入磁盘      │
│  - 发送创建信号      │
└─────────┬───────────┘
          │
          ▼ (分离点)
┌─────────────────────┐
│   4. 调度阶段        │
│  - [条件触发]入队    │
│  - 优先级排序        │
│  - Worker消费执行    │
│                     │
│  [调度器独立流程]    │
│  - 扫描新监控项      │
│  - 安排首次检查      │
└─────────────────────┘
```

---

## 六、容错与可靠性总结

| 阶段 | 容错机制 | 可靠性保障 |
|-----|---------|-----------|
| 解析 | URL验证、参数校验、格式容错 | 拒绝非法输入 |
| 归并 | 去重检测（仅API）、字段白名单 | 数据一致性 |
| 入库 | 原子写入、批量限制、内存注册 | 数据完整性 |
| 调度 | 异步后台线程、优先级队列、条件入队 | 系统稳定性 |

---

## 七、关键配置项

| 配置项 | 环境变量 | 默认值 | 作用 |
|-------|---------|-------|------|
| 批量导入阈值 | `IMPORT_SWITCH_TO_BACKGROUND_THRESHOLD` | 20 | 超过此值切换异步 |
| 单次导入上限 | 硬编码 | 5000 | 防止服务器过载 |
| 监控数量限制 | `PAGE_WATCH_LIMIT` | 无 | 全局监控数量上限 |
| 原子写入fsync | `FORCE_FSYNC_DATA_IS_CRITICAL` | False | 强制数据刷盘 |

---

## 八、核心差异总结表

### 8.1 各入口重复处理差异

| 入口 | 去重检查 | 重复行为 | 适用场景 |
|-----|---------|---------|---------|
| API批量导入 | 有（可关闭） | 跳过已存在项 | 大规模导入 |
| 网页导入 | 无 | 创建重复项 | 手动操作 |
| 命令行 | 无 | 创建重复项 | 脚本/自动化 |
| CreateWatch API | 无 | 创建重复项 | 程序化创建 |

### 8.2 各入口入队行为

| 入口 | 自动入队 | 入队条件 |
|-----|---------|---------|
| API批量导入 | 否 | 需手动调用 `?recheck_all=true` |
| 网页导入 | 否 | 需手动触发检查 |
| 命令行(-u) | 否 | 需同时指定 `-b` |
| 命令行(-b -u) | 是 | 自动入队新添加项 |
| CreateWatch API | 否 | 调度器自动安排 |
| 编辑保存 | 条件入队 | 非暂停 + 在时间窗口内 |

---

## 九、触发源 × 门槛条件判定矩阵

### 9.1 四类触发路径的门槛执行情况

| 门槛条件 | Ticker周期调度 | 单项recheck | recheck_all | Batch mode自动入队 |
|---------|---------------|------------|-------------|-------------------|
| **全局暂停(all_paused)** | ✅ 执行 | ❌ 不执行 | ❌ 不执行 | ❌ 不执行 |
| **单项暂停(watch.paused)** | ✅ 执行 | ✅ 执行 | ✅ 执行 | ✅ 执行 |
| **时间窗口(is_in_schedule)** | ✅ 执行 | ❌ 不执行 | ❌ 不执行 | ❌ 不执行 |
| **运行中去重(running_uuids)** | ✅ 执行 | ✅ 执行 | ✅ 执行 | ✅ 执行 |
| **已入队去重(queued_uuids)** | ✅ 执行 | ✅ 执行 | ✅ 执行 | ✅ 执行 |
| **队列上限(maxsize)** | ✅ 执行 | ✅ 执行 | ✅ 执行 | ✅ 执行 |
| **代理复用冷却(reuse_time_minimum)** | ✅ 执行(常规) / ❌ 跳过(首次) | ✅ 执行 | ✅ 执行 | ✅ 执行(常规) / ❌ 跳过(首次) |

### 9.2 判定顺序差异：首次检查 vs 常规周期检查

#### 9.2.1 Ticker周期调度 - 首次检查判定顺序

```
触发 → 全局暂停检查 → 单项暂停检查 → 时间窗口检查 → 运行中/已入队去重 → 时间阈值检查(自动通过) → 队列入队
                                                                           ↑
                                                                  last_checked=0，必然满足
```

**特点**：
- 代理复用冷却**跳过**（首次检查无前序调用）
- 时间阈值检查**自动通过**（`now - 0` 远大于阈值）

#### 9.2.2 Ticker周期调度 - 常规周期检查判定顺序

```
触发 → 全局暂停检查 → 单项暂停检查 → 时间窗口检查 → 运行中/已入队去重 → 时间阈值检查 → 代理复用冷却检查 → 队列入队
                                                                              ↑
                                                                     需要等待threshold秒
```

**特点**：
- 代理复用冷却**执行**（需检查时间间隔）
- 时间阈值检查**需满足**（等待threshold秒后）

#### 9.2.3 单项recheck判定顺序

```
触发 → 单项暂停检查 → 运行中/已入队去重 → 队列入队
```

**特点**：
- **绕过**全局暂停、时间窗口限制
- **跳过**时间阈值检查、代理复用冷却检查
- 优先级为 **1**（立即执行）

#### 9.2.4 recheck_all判定顺序

```
触发 → 单项暂停检查 → 运行中/已入队去重 → 队列入队（逐项）
```

**特点**：
- **绕过**全局暂停、时间窗口限制
- **跳过**时间阈值检查、代理复用冷却检查
- 优先级为 **1**（立即执行）
- 20项以下同步处理，20项以上后台异步处理

#### 9.2.5 Batch mode自动入队判定顺序

```
触发 → 单项暂停检查 → 运行中/已入队去重 → 队列入队（逐项）
```

**特点**：
- **绕过**全局暂停、时间窗口限制
- **跳过**时间阈值检查、代理复用冷却检查（首次检查）
- 优先级为 **1**（立即执行）

---

## 十、改造验收Checklist

### 10.1 功能正确性验证

| 检查项 | 验证方法 | 预期结果 |
|-------|---------|---------|
| 全局暂停生效 | 设置 `all_paused=true`，等待ticker执行 | 所有监控项不应被调度 |
| 全局暂停不影响手动触发 | 设置 `all_paused=true`，调用 `?recheck=true` | 监控项应立即执行检查 |
| 单项暂停生效 | 暂停单个监控项，等待ticker执行 | 该监控项不应被调度 |
| 时间窗口生效 | 配置非当前时间段的窗口，等待ticker执行 | 监控项不应被调度 |
| 时间窗口在窗口期内生效 | 配置当前时间段的窗口，等待ticker执行 | 监控项应被正常调度 |
| 运行中去重生效 | 手动触发检查，立即再次触发 | 第二次触发应被跳过 |
| 已入队去重生效 | 快速连续两次调用 `?recheck=true` | 第二次入队应被跳过 |
| 代理复用冷却生效 | 配置 `reuse_time_minimum=10`，连续触发两次 | 第二次应等待10秒后执行 |
| 队列上限生效 | 配置较小的maxsize，持续入队 | 队列满时应拒绝入队 |

### 10.2 首次检查验证

| 检查项 | 验证方法 | 预期结果 |
|-------|---------|---------|
| 新导入项自动被调度 | 导入新URL，等待ticker执行 | 监控项应被自动触发首次检查 |
| 首次检查不触发代理冷却 | 配置代理冷却，导入新URL | 首次检查应立即执行，不受冷却限制 |
| 首次检查受时间窗口限制 | 配置非当前时间段窗口，导入新URL | 监控项不应立即执行，需等待窗口期 |

### 10.3 批量导入验证

| 检查项 | 验证方法 | 预期结果 |
|-------|---------|---------|
| API批量导入去重 | 导入包含重复URL的列表 | 重复URL应被跳过 |
| API批量导入可关闭去重 | 导入时设置 `dedupe=false` | 重复URL应被创建 |
| 网页导入不去重 | 通过网页导入重复URL | 重复URL应被创建 |
| 批量导入后入队时机 | API导入100条URL | 新项由ticker线程自动发现并调度 |
| Batch mode自动入队 | 命令行 `-b -u` 参数 | 新添加的URL应立即入队 |

### 10.4 性能与可靠性验证

| 检查项 | 验证方法 | 预期结果 |
|-------|---------|---------|
| 批量导入阈值 | 导入21条URL | 应切换到后台异步处理 |
| recheck_all阈值 | 调用 `?recheck_all=true`（>20项） | 应切换到后台异步处理 |
| 原子写入 | 写入时断电/终止进程 | 数据不应损坏 |
| 队列线程安全 | 多线程并发入队 | 应无数据竞争问题 |

### 10.5 边界条件验证

| 检查项 | 验证方法 | 预期结果 |
|-------|---------|---------|
| 空URL列表 | 导入空列表 | 应返回成功，无错误 |
| 超大URL列表(>5000) | 导入5500条URL | 仅前5000条应被处理 |
| 无效URL | 导入无效格式URL | 应返回错误，拒绝导入 |
| 监控数量上限 | 达到 `PAGE_WATCH_LIMIT` 后添加 | 应返回错误，拒绝添加 |

---

**文档版本**: v3.0  
**生成时间**: 2026-05-15  
**代码版本**: changedetection.io v0.55.3