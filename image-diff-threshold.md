# 页面截图比对：从差异检测到通知发送的真实调用链

## 核心发现

经过代码追踪，发现几个关键事实与之前的分析不同：

1. **主流程归属**：`worker.py:async_update_worker()` 是真正的调度中心，而非处理器本身
2. **阈值默认值不一致**：`forms.py` 默认 `0.1`，但 `processor.py` 异常回退写死 `1`；`SCREENSHOT_COMPARISON_THRESHOLD_OPTIONS_DEFAULT=0.999` 转 `int()` 后变成 `0`，存在潜在问题
3. **通知前置条件**：必须同时满足 4 个条件才会发送通知
4. **LLM 二次过滤**：即使图像比对判定有变化，LLM 也可能将其标记为不重要而取消通知

---

## 真实调用链

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          完整调用链（自顶向下）                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [1] worker.py:async_update_worker()  ← 主调度者                         │
│       │                                                                 │
│       ├─→ 从队列获取 Watch UUID                                          │
│       ├─→ claim_uuid_for_processing() 防止并发重复处理                    │
│       ├─→ 初始化处理器: get_processor_module(processor)                 │
│       ├─→ await update_handler.call_browser()  ← 获取页面/截图           │
│       │                                                                 │
│       ├─→ [2] run_changedetection()  ← 线程池中执行                      │
│       │       │                                                         │
│       │       └─→ processor.py:perform_site_check.run_changedetection() │
│       │               │                                                 │
│       │               ├─→ MD5 快速校验（相同则抛异常）                     │
│       │               ├─→ 解析阈值配置（Watch → 全局 → 默认值）            │
│       │               ├─→ 解析区域选择（Bounding Box / Visual Selector）  │
│       │               ├─→ [3] compare_images_isolated()  ← 子进程        │
│       │               │       │                                         │
│       │               │       └─→ isolated_opencv.py:_worker_compare()  │
│       │               │               ├─→ 图像解码 → 裁剪 → 灰度 → 模糊    │
│       │               │               ├─→ absdiff() → threshold()       │
│       │               │               └─→ 返回 change_percentage        │
│       │               │                                                 │
│       │               └─→ 判定: changed_detected = (score > min_pct)    │
│       │                                                                 │
│       ├─→ [4] 异常捕获层（worker.py 中）                                 │
│       │       ├─→ checksumFromPreviousCheckWasTheSame → 无变化          │
│       │       ├─→ ProcessorException → 保存错误截图 → 中断流程            │
│       │       ├─→ FilterNotFoundInResponse → 可能发送过滤失败通知        │
│       │       └─→ 其他异常 → 设置 last_error → 中断流程                   │
│       │                                                                 │
│       ├─→ [5] LLM 二次过滤（可选）                                       │
│       │       ├─→ 仅当 changed_detected=True 且配置了 LLM 时执行         │
│       │       └─→ LLM 判定 "不重要" → changed_detected = False          │
│       │                                                                 │
│       ├─→ [6] 保存历史（仅当有变化或首次检测）                            │
│       │       ├─→ save_screenshot()                                     │
│       │       └─→ save_history_blob()                                   │
│       │                                                                 │
│       └─→ [7] 通知触发条件检查                                          │
│               ├─→ changed_detected == True ?                            │
│               ├─→ watch.history_n >= 2 ?                                │
│               ├─→ watch.notification_muted == False ?                   │
│               └─→ 配置了 notification_urls ?                            │
│                   └─→ 全部满足 → send_content_changed_notification()    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 模块职责划分

| 模块 | 职责 | 关键决策点 |
|------|------|-----------|
| `worker.py` | **主调度者** | 异常捕获、LLM 过滤、通知触发条件判断、历史保存 |
| `processor.py` | 图像比对执行 | 阈值解析、区域裁剪、调用子进程、双层阈值判定 |
| `isolated_opencv.py` | 图像算法 | 解码、裁剪、模糊、absdiff、threshold、百分比计算 |
| `notification_service.py` | 通知构建 | 解析通知模板、级联配置（Watch→Tag→Global）、入队 |

---

## 阈值解析与默认值

### 像素级敏感度 (pixel_difference_threshold_sensitivity)

**代码位置**：`processor.py:70-79`

```python
# 优先级链
processor_config.get('pixel_difference_threshold_sensitivity')  # Watch 专属
    ↓ (未设置)
datastore['settings']['application'].get('pixel_difference_threshold_sensitivity', 
    SCREENSHOT_COMPARISON_THRESHOLD_OPTIONS_DEFAULT)  # 全局默认 0.999
    ↓
int(0.999) → 0  ⚠️ 潜在问题！
```

**⚠️ 发现的问题**：
- `SCREENSHOT_COMPARISON_THRESHOLD_OPTIONS_DEFAULT = 0.999` 是浮点数
- 代码中 `int(pixel_difference_threshold_sensitivity)` 会将其转为 `0`
- 这意味着**默认配置下像素敏感度为 0（极高敏感度）**，与 UI 推荐的 80 不符

### 变化百分比门槛 (min_change_percentage)

**代码位置**：`processor.py:82-91`

```python
processor_config.get('min_change_percentage')  # Watch 专属
    ↓ (未设置)
datastore['settings']['application'].get('min_change_percentage', 1)  # 全局默认 1
    ↓
int(value)
    ↓ (异常)
min_change_percentage = 1  # 硬编码回退
```

**⚠️ 不一致**：
- `forms.py:1046` 定义表单默认值 `default=0.1`
- `processor.py:86` 全局回退写死 `1`
- `processor.py:91` 异常回退也写死 `1`

---

## 双层阈值判定机制

### 判定流程

```
子进程返回: change_percentage (0-100%，已通过像素阈值过滤)
                          │
                          ▼
    changed_detected = (change_percentage > min_change_percentage)
                          │
                          ├── True → 进入 LLM 过滤阶段
                          └── False → 无变化，不通知
```

**关键点**：
- **第一层（像素级）**：在子进程内完成，`cv2.threshold(diff, threshold, 255, THRESH_BINARY)`
- **第二层（百分比）**：在 `processor.py:219` 完成，`change_score > min_change_percentage`
- 子进程只返回百分比，**不做最终判定**

---

## 区域选择策略

**代码位置**：`processor.py:62-141`

| 模式 | 触发条件 | 裁剪坐标来源 |
|------|---------|-------------|
| Bounding Box | `processor_config.bounding_box` 存在 | 用户手绘: `x,y,width,height` |
| Visual Selector | `watch.include_filters` 存在且有 `xpath_data` | 元素 bounding box: `left, top, width, height` |
| 全图 | 以上都不满足 | 不裁剪 |

**注意**：区域裁剪发生在**像素比对之前**，变化百分比基于裁剪后的区域计算。

---

## 通知触发前置条件

**代码位置**：`worker.py:530-534`

```python
if watch.history_n >= 2:
    logger.info(f"Change detected in UUID {uuid} - {watch['url']}")
    if not watch.get('notification_muted'):
        await send_content_changed_notification(uuid, notification_q, datastore)
```

**必须同时满足 4 个条件**：

| 条件 | 代码位置 | 说明 |
|------|---------|------|
| `changed_detected == True` | `processor.py:219` | 图像比对判定有变化 |
| `watch.history_n >= 2` | `worker.py:531` | 至少有两次历史快照（首次检测不通知） |
| `not watch.get('notification_muted')` | `worker.py:533` | 用户未静音该 Watch |
| 配置了 `notification_urls` | `notification_service.py:423` | Watch/Tag/Global 任一有通知地址 |

**额外过滤层 - LLM**（`worker.py:458-464`）：
```python
if _llm_result and not _llm_result.get('important', True):
    changed_detected = False  # LLM 认为不重要，取消通知
```

---

## 失败回退协作机制

### 异常层级与协作

```
子进程异常 (isolated_opencv.py)
    │
    └─→ 抛出 RuntimeError / TimeoutError
              │
              ▼
    processor.py:222-229 捕获
    └─→ 封装为 ProcessorException(message, url, screenshot, xpath_data)
              │
              ▼
    worker.py:182-190 捕获
    ├─→ 保存截图: watch.save_screenshot(e.screenshot)
    ├─→ 保存 xpath_data
    ├─→ 设置 last_error
    └─→ process_changedetection_results = False  ← 中断后续流程
```

### 特殊异常：checksumFromPreviousCheckWasTheSame

**代码位置**：`processor.py:50-58` + `worker.py:280-289`

```python
# processor.py - MD5 相同则抛异常
if previous_md5 and current_md5 == previous_md5:
    raise checksumFromPreviousCheckWasTheSame()

# worker.py - 捕获后当作"无变化"处理
except content_fetchers_exceptions.checksumFromPreviousCheckWasTheSame as e:
    process_changedetection_results = False
    changed_detected = False
    datastore.update_watch(uuid=uuid, update_obj={'last_error': False})
    cleanup_error_artifacts(uuid, datastore)
```

**这不是错误**，而是性能优化路径：跳过昂贵的图像比对。

### 首次检测的特殊处理

**代码位置**：`processor.py:146-158`

```python
if len(history_keys) == 0:
    # 首次检测 - 只保存基准线，不比较，不通知
    update_obj = {
        'previous_md5': hashlib.md5(self.screenshot).hexdigest(),
        'last_error': False
    }
    return False, update_obj, self.screenshot  # changed_detected=False
```

---

## 子进程隔离

**代码位置**：`isolated_opencv.py:109-197`

| 特性 | 实现方式 |
|------|---------|
| 进程创建 | `multiprocessing.get_context('spawn')` |
| 通信 | `Pipe()` 双向通信 |
| 线程控制 | `cv2.setNumThreads(1)` 防止子进程内线程爆炸 |
| 超时保护 | `POLL_TIMEOUT_ABSOLUTE` (默认 20 秒) |
| 异步友好 | `asyncio.to_thread()` 包装阻塞调用 |
| 清理流程 | close pipe → join(5s) → terminate(3s) → kill(1s) |

---

## 关键文件索引

| 文件 | 关键行数 | 职责 |
|------|---------|------|
| `worker.py` | 138-175, 280-289, 421-464, 530-534 | 主流程、异常处理、LLM 过滤、通知触发 |
| `processors/image_ssim_diff/processor.py` | 50-58, 70-91, 146-158, 219, 222-229 | MD5 校验、阈值解析、双层判定、异常封装 |
| `processors/image_ssim_diff/image_handler/isolated_opencv.py` | 16-106, 109-197 | 图像处理算法、子进程管理 |
| `notification_service.py` | 17-54, 389-431 | 通知级联配置、通知入队 |
| `processors/image_ssim_diff/__init__.py` | 31-39 | 预设选项、默认值定义（含 0.999 问题） |
| `forms.py` | 1046-1054 | 全局设置表单（min_change_percentage 默认 0.1） |

---

## 总结

### 真正的判定链

```
1. MD5 相同? → 无变化（优化路径）
2. 首次检测? → 只存基准线，不通知
3. 像素差异 > 像素阈值? → 标记为变化像素
4. 变化像素百分比 > min_change_percentage? → changed_detected=True
5. LLM 认为重要? → 可能取消 changed_detected
6. history_n >= 2? → 首次不通知
7. notification_muted? → 静音不通知
8. 有 notification_urls? → 无地址不通知
   └─→ 全部通过 → 发送通知
```

### 发现的潜在问题

1. **`SCREENSHOT_COMPARISON_THRESHOLD_OPTIONS_DEFAULT=0.999`** 转 `int()` 后为 `0`，与 UI 推荐的 80 不符
2. **`min_change_percentage` 默认值不一致**：forms.py 是 0.1，processor.py 硬编码 1
3. **类型转换风险**：浮点阈值被强制转 int，可能丢失精度
