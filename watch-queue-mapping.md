# Watch 到抓取队列任务元数据转换映射关系

## 1. 架构概览

### 1.1 核心流程
```
用户配置 Watch → 调度器/手动触发 → 队列任务元数据 → Worker 从 datastore 加载完整 Watch → 执行检查
```

1.2 关键文件位置
- Watch 模型: `changedetectionio/model/__init__.py` (watch_base) 和 `changedetectionio/model/Watch.py` (model)
- 队列元数据: `changedetectionio/queuedWatchMetaData.py`
- 队列管理: `changedetectionio/queue_handlers.py`
- Worker 处理: `changedetectionio/worker.py`
- 调度器: `changedetectionio/flask_app.py` (ticker thread)

## 2. Watch 模型字段定义

Watch 模型继承自 `dict` (技术债务)，包含以下核心字段:

### 2.1 核心标识字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| uuid | str | 唯一标识符，UUID格式 |
| url | str | 监控URL |
| title | str\|None | 自定义标题 |
| page_title | str\|None | 从页面 `<title>` 提取的标题 |
| tags | List[str] | 标签 UUID 列表 |

### 2.2 检查配置字段
| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| processor | str | 'text_json_diff' | 处理器类型 |
| fetch_backend | str | 'system' | 抓取后端 |
| method | str | 'GET' | HTTP 方法 |
| headers | dict | {} | 自定义 HTTP 头 |
| proxy | str\|None | None | 代理服务器 |
| paused | bool | False | 是否暂停 |

### 2.3 调度字段
| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| time_between_check | dict | {'weeks': None, ...} | 检查间隔 |
| time_between_check_use_default | bool | True | 是否使用默认间隔 |
| time_schedule_limit | dict | {...} | 时间调度限制 |
| last_checked | int | 0 | 上次检查时间戳 |
| jitter_seconds | float | 0 | 抖动时间 |

### 2.4 内容过滤字段
| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| include_filters | List[str] | [] | CSS/XPath 选择器 |
| subtractive_selectors | List[str] | [] | 要移除的选择器 |
| ignore_text | List[str] | [] | 忽略的文本模式 |
| trigger_text | List[str] | [] | 触发变更的文本 |
| extract_text | List[str] | [] | 提取文本正则表达式 |

### 2.5 浏览器自动化字段
| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| browser_steps | List[dict] | [] | 浏览器步骤列表 |
| browser_steps_last_error_step | int\|None | None | 出错的步骤 |
| webdriver_delay | int\|None | None | 页面加载后等待时间 |
| webdriver_js_execute_code | str\|None | None | 执行的 JS 代码 |

### 2.6 通知相关字段
| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| notification_urls | List[str] | [] | 通知 URL |
| notification_title | str\|None | None | 通知标题 |
| notification_body | str\|None | None | 通知内容 |
| notification_format | str | 'System default' | 通知格式 |
| notification_muted | bool | False | 是否静音 |
| notification_screenshot | bool | False | 是否包含截图 |

### 2.7 历史与状态字段
| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| last_checked | int | 0 | 上次检查时间戳 |
| last_viewed | int | 0 | 上次查看时间戳 |
| last_error | str\|bool | False | 上次错误信息 |
| check_count | int | 0 | 检查次数 |
| fetch_time | float | 0.0 | 抓取耗时 |
| previous_md5 | str\|bool | False | 上次内容 MD5 |
| consecutive_filter_failures | int | 0 | 连续过滤失败次数 |
| history_snapshot_max_length | int\|None | None | 历史快照最大数量 |

## 3. 队列任务元数据定义

### 3.1 PrioritizedItem 数据类

**定义位置**: `changedetectionio/queuedWatchMetaData.py

```python
@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any=field(compare=False)
```

### 3.2 队列任务的 item 结构

**重要发现**: **队列任务只传递 `uuid` 字段！

```python
# 所有入队操作都使用相同模式:
PrioritizedItem(priority=?, item={'uuid': uuid})
```

**示例** (来自代码搜索结果):
- `__init__.py:441 - `{'uuid': watch_uuid}`
- `flask_app.py:1253 - `{'uuid': uuid}`
- `blueprint/ui/views.py:41` - `{'uuid': new_uuid}`
- `blueprint/ui/__init__.py:66` - `{'uuid': uuid}`
- `api/Watch.py:81` - `{'uuid': uuid}`
- 等等...

### 3.3 队列优先级规则

| 优先级值 | 含义 | 来源 |
|--------|------|------|
| 1 | 立即执行 | 手动触发、新建、编辑后 | UI/API/手动 |
| 5 | 克隆操作 | `blueprint/ui/__init__.py:257 |
| >100 | 调度执行 | `flask_app.py:1249 - `int(time.time())` |
| 1000+ | 延迟重试 | worker 中处理冲突时 |

## 4. 完整转换流程

### 4.1 调度器入队流程 (ticker thread)

**位置**: `changedetectionio/flask_app.py:1170-1270

```python
def queue_watches_for_recheck():
    for uuid, watch in datastore.data['watching'].items():
        # 1. 检查是否暂停
        if watch.get('paused') 或全局暂停:
            continue
            
        # 2. 检查时间调度
        if time_schedule_limit:
            if not is_within_schedule(...):
                continue
                
        # 3. 计算检查间隔
        threshold = watch.threshold_seconds()
        jitter = watch.jitter_seconds
        
        # 4. 检查是否达到检查时间
        if seconds_since_last_recheck >= threshold + jitter:
            if not uuid in running_uuids and uuid not in queued_uuids:
                
                # 5. 检查代理限制
                if watch_proxy:
                    if time_since_proxy_used < proxy_reuse_time_minimum:
                        continue
                        
                # 6. 入队（只传 uuid！
                priority = int(time.time())
                PrioritizedItem(priority=priority, item={'uuid': uuid})
```

### 4.2 Worker 处理流程

**位置**: `changedetectionio/worker.py

```python
async def async_update_worker():
    # 1. 从队列获取任务
    queued_item_data = await q.async_get(...)
    uuid = queued_item_data.item.get('uuid')
    
    # 2. 从 datastore 加载完整 Watch 对象
    watch = datastore.data['watching'].get(uuid)
    
    # 3. 使用完整 Watch 配置执行检查
    processor = watch.get('processor')
    fetch_backend = watch.get('fetch_backend')
    headers = watch.get('headers')
    browser_steps = watch.get('browser_steps')
    # ... 等等所有字段
```

## 5. 跨模块字段传递约定

### 5.1 核心原则

**最小传递原则**: 队列只传递 `uuid`，其他所有字段都通过 `datastore` 传递

### 5.2 数据流

```
队列任务 (只传 uuid
    ↓
Worker 从 datastore['watching'][uuid] 获取完整 Watch
    ↓
使用完整 Watch 配置执行抓取和变更检测
```

### 5.3 为什么只传 uuid?

1. **内存效率**: 完整 Watch 对象可能很大（包含历史记录等
2. **一致性**: 确保使用最新的 Watch 配置
3. **简单性**: 避免序列化/反序列化复杂对象
4. **可扩展性**: 不需要修改队列结构当 Watch 配置

### 5.4 优先级计算约定

| 场景 | 优先级值 | 说明 |
|------|----------|------|
| 立即执行 | 1 | 最高优先级，立即执行 |
| 克隆 | 5 | 次高优先级 |
| 调度执行 | `int(time.time()) | 时间戳作为优先级，实现 FIFO |
| 延迟重试 | max(1000, 原优先级*10) | 避免立即重试冲突 |

## 6. 入队触发点

### 6.1 调度器自动触发

**位置**: `flask_app.py

- 定时检查所有 watch 调度器线程
- 检查间隔: `WAIT_TIME_BETWEEN_LOOP` 秒循环
- 根据 `time_between_check` + jitter 判断

### 6.2 UI 手动触发

**位置**:

| 文件 | 行号 | 场景 |
|------|------|------|
| `blueprint/ui/views.py` | 41 | 新建 watch |
| `blueprint/ui/__init__.py | 66 | 手动重新检查 |
| `blueprint/ui/__init__.py` | 276 | 批量重新检查 |
| `blueprint/ui/__init__.py` | 305 | 标签操作 |
| `blueprint/ui/edit.py` | 277 | 编辑后 |

### 6.3 API 触发

**位置**:

| 文件 | 行号 | 场景 |
|------|------|------|
| `api/Watch.py` | 81 | 创建 watch |
| `api/Watch.py` | 554, 576 | 重新检查 |
| `api/Tags.py` | 42, 50 | 标签操作 |

### 6.4 其他触发

**位置**:

| 文件 | 行号 | 场景 |
|------|------|------|
| `__init__.py` | 441, 475, 537 | 命令行 -r 选项 |
| `realtime/events.py` | 44 | 实时事件 |
| `blueprint/price_data_follower/__init__.py` | 24 | 价格数据 |

## 7. 队列管理

### 7.1 队列类型

**位置**: `queue_handlers.py:RecheckPriorityQueue

- 线程安全优先级队列
- 支持同步/异步接口
- 信号通知队列长度变化

### 7.2 去重机制

```python
# 防止重复入队检查
if not uuid in running_uuids and uuid not in queued_uuids:
    # 入队
```

**位置**: `flask_app.py:1227

### 7.3 冲突处理

**位置**: `worker.py:70-77

```python
# 声明 UUID 防止重复处理
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # 已在处理中，延迟重试
    deferred_priority = max(1000, queued_item_data.priority * 10)
```

## 8. 关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| Watch 基类 | `model/__init__.py` | 15-687 |
| Watch 子类 | `model/Watch.py` | 136-1300+ |
| 队列元数据 | `queuedWatchMetaData.py` | 1-10 |
| 队列实现 | `queue_handlers.py` | 15-411 |
| 调度器入队 | `flask_app.py` | 1170-1270 |
| Worker 处理 | `worker.py` | 23-300+ |
| 入队调用点 | 多个文件 | 见上文 |

## 9. 注意事项

1. **字段传递**: 队列只传 `uuid`，其他字段都通过 datastore['watching'][uuid] 获取
2. **优先级**: 1=立即, 5=克隆, 时间戳=调度
3. **去重**: running_uuids 和 queued_uuids 防止重复
4. **冲突处理**: claim_uuid_for_processing 防止并发处理
5. **延迟重试**: 已在处理中的任务延迟到低优先级重试

## 10. 示例

```python
# 用户配置 watch
watch = {
    'uuid': '123e4567-e89b-12d3-a456-426614174000',
    'url': 'https://example.com',
    'processor': 'text_json_diff',
    'fetch_backend': 'html_requests',
    'time_between_check': {'hours': 1},
    # ... 更多字段
}

# 入队时只传
PrioritizedItem(
    priority=1715680000,  # 时间戳
    item={'uuid': '123e4567-e89b-12d3-a456-426614174000'}
)

# Worker 获取后从 datastore 加载完整 watch = datastore.data['watching']['123e4567-e89b-12d3-a456-426614174000']
# 使用完整配置执行检查
```