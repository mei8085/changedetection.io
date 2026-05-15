# 监控列表导入链路分析报告

## 概述

changedetection.io 的监控列表导入流程包含四个核心阶段：**解析** → **归并** → **入库** → **调度**。本文档详细分析每个阶段的容错策略、字段对齐方式、批量处理行为及合并语义。

---

## 一、解析阶段

### 1.1 入口渠道

系统支持多种导入入口：

| 入口类型 | 位置 | 数据源格式 |
|---------|------|-----------|
| API导入 | `api/Import.py` | 纯文本URL列表（每行一个） |
| 网页导入 | `blueprint/imports/importer.py` | URL列表、Distill.io JSON、Wachete XLSX、自定义XLSX |
| 命令行 | `__init__.py` | `-u` 参数指定URL |

### 1.2 容错策略

#### 1.2.1 URL验证
```python
# api/Import.py:188-189
if not is_safe_valid_url(url):
    return f"Invalid or unsupported URL - {url}", 400
```

**验证规则**：
- 检查协议白名单（http/https）
- 防止URL注入攻击
- 拒绝不安全的URL格式

#### 1.2.2 重复检测（去重）
```python
# api/Import.py:191-193
if dedupe and self.datastore.url_exists(url):
    continue
```

- 默认开启去重（`dedupe=true`）
- 通过 `datastore.url_exists()` 检查
- 不区分大小写比较

#### 1.2.3 批量大小限制
```python
# blueprint/imports/importer.py:44-46
if (len(urls) > 5000):
    flash(gettext("Importing 5,000 of the first URLs from your list..."))
```

**限制策略**：
- 单次导入最多处理 **5000** 条记录
- 超出部分被跳过，用户可分批导入

#### 1.2.4 JSON解析容错
```python
# blueprint/imports/importer.py:95-99
try:
    data = json.loads(data.strip())
except json.decoder.JSONDecodeError:
    flash(gettext("Unable to read JSON file, was it broken?"), 'error')
    return
```

#### 1.2.5 参数校验
```python
# api/Import.py:150-168
# 验证processor
if 'processor' in extras:
    available = [p[0] for p in available_processors()]
    if extras['processor'] not in available:
        return f"Invalid processor '{extras['processor']}'", 400

# 验证fetch_backend
if 'fetch_backend' in extras:
    is_valid = (
        extras['fetch_backend'] == 'system' or
        extras['fetch_backend'] in available or
        extras['fetch_backend'].startswith('extra_browser_')
    )
```

### 1.3 字段对齐方式

#### 1.3.1 参数类型转换
```python
# api/Import.py:26-93
def convert_query_param_to_type(value, schema_property):
    """
    支持类型：
    - array: 逗号分隔或JSON数组
    - object: JSON对象
    - boolean: strtobool转换
    - integer: 整数
    - number: 浮点数
    - string: 保持原样
    """
```

#### 1.3.2 外部格式映射

**Distill.io JSON → 内部格式**：
```python
# blueprint/imports/importer.py:111-127
if d_config['selections'][0]['frames'][0]['excludes'][0]['type'] == 'css':
    extras['subtractive_selectors'] = d_config['selections'][0]['frames'][0]['excludes'][0]['expr']

if d_config['selections'][0]['frames'][0]['includes'][0]['type'] == 'xpath':
    extras['include_filters'].append('xpath:' + expr)
else:
    extras['include_filters'].append(expr)
```

**Wachete XLSX → 内部格式**：
| Wachete字段 | 内部字段 | 转换逻辑 |
|------------|---------|---------|
| `dynamic wachet` | `fetch_backend` | true→html_webdriver, false→html_requests |
| `xpath` | `include_filters` | 直接映射为列表 |
| `name` | `title` | 直接映射 |
| `interval (min)` | `time_between_check` | 转换为{weeks, days, hours, minutes, seconds} |
| `folder` | `tag` | 直接映射 |

---

## 二、归并阶段

### 2.1 合并语义

#### 2.1.1 去重策略
```python
# store/__init__.py:660-666
def url_exists(self, url):
    for watch in self.data['watching'].values():
        if watch['url'].lower() == url.lower():
            return True
    return False
```

**语义规则**：
- **URL唯一性约束**：基于URL（不区分大小写）判断重复
- **去重时机**：解析阶段立即过滤
- **保留策略**：已存在的监控项保持不变，新导入的重复项被跳过

#### 2.1.2 标签合并
```python
# store/__init__.py:751-766
if tag and type(tag) == str:
    for t in tag.split(','):
        for a_t in t.split(','):
            tag_uuid = self.add_tag(a_t)
            apply_extras['tags'].append(tag_uuid)

if tag_uuids:
    for t in tag_uuids:
        apply_extras['tags'] = list(set(apply_extras['tags'] + [t.strip()]))

if apply_extras.get('tags'):
    apply_extras['tags'] = list(set(apply_extras.get('tags')))
```

**语义规则**：
- 支持**标签名称**和**标签UUID**两种格式
- 自动去重，保证标签列表唯一性
- 标签不存在时自动创建

#### 2.1.3 共享链接解析
```python
# store/__init__.py:684-728
if (url.startswith("https://changedetection.io/share/")):
    r = requests.request(method="GET", url=url, 
                        headers={'App-Guid': self.__data['app_guid']}, timeout=5.0)
    res = r.json()
    
    # 白名单属性列表
    for k in ['body', 'browser_steps', 'css_filter', 'extract_text', ...]:
        if res.get(k):
            if k != 'css_filter':
                apply_extras[k] = res[k]
            else:
                apply_extras['include_filters'] = [res['css_filter']]
```

**安全策略**：
- 仅接受白名单内的属性
- 字段名映射（如`css_filter`→`include_filters`）
- 5秒超时防止阻塞

---

## 三、入库阶段

### 3.1 核心流程

```python
# store/__init__.py:674-794
def add_watch(self, url, tag='', extras=None, tag_uuids=None, save_immediately=True):
    # 1. 参数准备
    apply_extras = deepcopy(extras)
    
    # 2. URL验证
    if not is_safe_valid_url(url):
        return None
    
    # 3. 数量限制检查
    page_watch_limit = os.getenv('PAGE_WATCH_LIMIT')
    if page_watch_limit and current_watch_count >= page_watch_limit:
        return None
    
    # 4. 标签处理
    if tag and type(tag) == str:
        # 添加标签UUID
    
    # 5. 创建Watch对象
    watch_class = get_custom_watch_obj_for_processor(apply_extras.get('processor'))
    new_watch = watch_class(datastore_path=self.datastore_path, ...)
    
    # 6. 应用额外配置
    new_watch.update(apply_extras)
    new_watch.ensure_data_dir_exists()
    
    # 7. 注册到内存
    self.__data['watching'][new_uuid] = new_watch
    
    # 8. 持久化
    if save_immediately:
        new_watch.commit()
    
    return new_uuid
```

### 3.2 持久化机制

#### 3.2.1 原子写入
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

#### 3.2.2 存储结构
```
datastore/
├── changedetection.json      # 全局配置
├── {watch-uuid}/
│   ├── watch.json            # 监控项配置
│   ├── history.txt           # 历史索引
│   ├── {timestamp}.txt(.br)  # 快照内容
│   ├── last-screenshot.png   # 截图
│   └── last-error.txt        # 错误信息
└── {tag-uuid}/
    └── tag.json              # 标签配置
```

### 3.3 立即保存 vs 延迟保存

| 模式 | `save_immediately` | 适用场景 | 行为 |
|-----|-------------------|---------|-----|
| 立即保存 | `True`（默认） | 单条添加 | 立即写入磁盘 |
| 延迟保存 | `False` | 批量导入 | 先写入内存，后续统一commit |

```python
# blueprint/imports/importer.py:65
new_uuid = datastore.add_watch(url=url.strip(), tag=tags, 
                                save_immediately=False, extras=extras)
```

---

## 四、调度阶段

### 4.1 批量处理行为

#### 4.1.1 同步 vs 异步阈值
```python
# api/Import.py:10-11
IMPORT_SWITCH_TO_BACKGROUND_THRESHOLD = 20

# api/Import.py:198-227
if len(urls_to_import) < IMPORT_SWITCH_TO_BACKGROUND_THRESHOLD:
    # 同步处理
    added = []
    for url in urls_to_import:
        new_uuid = self.datastore.add_watch(...)
        added.append(new_uuid)
    return added, 200
else:
    # 异步后台线程处理
    def import_watches_background():
        for url in urls_to_import:
            try:
                self.datastore.add_watch(...)
            except Exception as e:
                logger.error(f"Error importing URL {url}: {e}")
    
    thread = threading.Thread(target=import_watches_background, 
                            daemon=True, name="ImportWatches-Background")
    thread.start()
    return {'status': 'Importing in background', 'count': len(urls_to_import)}, 202
```

#### 4.1.2 队列优先级

```python
# custom_queue.py:271-295
# 优先级定义
immediate_items = 0  # priority 1 - 立即检查
clone_items = 0      # priority 5 - 克隆操作
scheduled_items = 0  # priority > 100 - 定时任务（timestamp）

# 优先级数值越小越优先
# priority=1: 最高优先级（立即执行）
# priority=5: 中等优先级（克隆）
# priority=timestamp: 定时任务（时间戳作为优先级）
```

### 4.2 队列入口

#### 4.2.1 优先级队列实现
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

#### 4.2.2 调度触发时机

| 触发事件 | 优先级 | 代码位置 |
|---------|-------|---------|
| 新添加监控 | 1（立即） | `add_watch()` 后自动入队 |
| 编辑监控 | 1（立即） | 更新后触发 |
| 定时检查 | timestamp | ticker线程调度 |
| 手动触发 | 1（立即） | 用户操作 |
| 克隆操作 | 5 | `clone()` 方法 |

---

## 五、完整链路流程图

```
用户提交监控列表
        │
        ▼
┌─────────────────────┐
│   1. 解析阶段        │
│  - URL验证          │
│  - 去重检测          │
│  - 参数类型转换      │
│  - 字段映射对齐      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   2. 归并阶段        │
│  - URL去重判断      │
│  - 标签合并去重      │
│  - 共享链接解析      │
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
          ▼
┌─────────────────────┐
│   4. 调度阶段        │
│  - 批量大小判断      │
│  - 同步/异步处理     │
│  - 优先级队列入队    │
│  - Worker消费执行    │
└─────────────────────┘
```

---

## 六、容错与可靠性总结

| 阶段 | 容错机制 | 可靠性保障 |
|-----|---------|-----------|
| 解析 | URL验证、参数校验、格式容错 | 拒绝非法输入 |
| 归并 | 去重检测、字段白名单 | 数据一致性 |
| 入库 | 原子写入、批量限制 | 数据完整性 |
| 调度 | 异步后台线程、优先级队列 | 系统稳定性 |

---

## 七、关键配置项

| 配置项 | 环境变量 | 默认值 | 作用 |
|-------|---------|-------|------|
| 批量导入阈值 | `IMPORT_SWITCH_TO_BACKGROUND_THRESHOLD` | 20 | 超过此值切换异步 |
| 单次导入上限 | 硬编码 | 5000 | 防止服务器过载 |
| 监控数量限制 | `PAGE_WATCH_LIMIT` | 无 | 全局监控数量上限 |
| 原子写入fsync | `FORCE_FSYNC_DATA_IS_CRITICAL` | False | 强制数据刷盘 |

---

**文档版本**: v1.0  
**生成时间**: 2026-05-15  
**代码版本**: changedetection.io v0.55.3