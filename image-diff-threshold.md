# 页面截图比对：图像差异阈值计算与通知触发机制

## 概述

Changedetection.io 的视觉截图比对系统采用 **双层阈值判定机制**，结合 **像素级敏感度** 和 **变化百分比门槛**，实现精确而可控的页面变化检测。本文档详细阐述从图像采样、相似度计算到最终判定和失败回退的完整协作流程。

---

## 目录

1. [核心概念与配置层级](#1-核心概念与配置层级)
2. [图像处理流水线](#2-图像处理流水线)
3. [双层阈值判定机制](#3-双层阈值判定机制)
4. [区域选择与裁剪策略](#4-区域选择与裁剪策略)
5. [子进程隔离与内存管理](#5-子进程隔离与内存管理)
6. [通知触发流程](#6-通知触发流程)
7. [失败回退与异常处理](#7-失败回退与异常处理)
8. [配置参数详解](#8-配置参数详解)

---

## 1. 核心概念与配置层级

### 1.1 双层阈值架构

系统采用**双层阈值**设计，分别控制：

| 阈值类型 | 参数名称 | 作用范围 | 默认值 | 可调范围 |
|---------|---------|---------|--------|---------|
| 像素级敏感度 | `pixel_difference_threshold_sensitivity` | 单个像素差异判断 | 0.999 (实际使用 80) | 0-255 |
| 变化百分比门槛 | `min_change_percentage` | 整体图像变化比例 | 1% | 1-100% |

### 1.2 配置优先级链

配置值的解析遵循 **Watch → 全局 → 默认值** 的优先级链：

```
Watch 处理器配置 (image_ssim_diff.json)
    ↓ (未设置)
全局应用配置 (settings.json)
    ↓ (未设置)
硬编码默认值
```

**代码位置**：
- `changedetectionio/processors/image_ssim_diff/processor.py:70-91`
- `changedetectionio/processors/image_ssim_diff/difference.py:341-351`

**配置解析逻辑示例**：
```python
# 1. 先尝试从 Watch 专属配置读取
pixel_difference_threshold_sensitivity = processor_config.get('pixel_difference_threshold_sensitivity')

# 2. 若未设置，回退到全局配置
if not pixel_difference_threshold_sensitivity:
    pixel_difference_threshold_sensitivity = self.datastore.data['settings']['application'].get(
        'pixel_difference_threshold_sensitivity', 
        SCREENSHOT_COMPARISON_THRESHOLD_OPTIONS_DEFAULT
    )

# 3. 类型转换和有效性验证
try:
    pixel_difference_threshold_sensitivity = int(pixel_difference_threshold_sensitivity)
except (ValueError, TypeError):
    logger.warning("Invalid value, using default")
    pixel_difference_threshold_sensitivity = SCREENSHOT_COMPARISON_THRESHOLD_OPTIONS_DEFAULT
```

### 1.3 预设敏感度选项

UI 提供四种预设敏感度级别：

| 值 | 描述 | 使用场景 |
|----|------|---------|
| 200 | 低敏感度（仅检测重大变化） | 监控主要内容块 |
| 80 | 中等敏感度（推荐默认） | 通用场景 |
| 20 | 高敏感度（检测小幅变化） | 精密监控 |
| 0 | 极高敏感度（任何变化） | 调试/测试 |

**代码位置**：`changedetectionio/processors/image_ssim_diff/__init__.py:31-36`

---

## 2. 图像处理流水线

### 2.1 处理流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    图像差异比对流水线                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. MD5 快速校验 (预处理)                                       │
│     ├── 完全相同 → 跳过对比，直接返回                            │
│     └── 不同 → 进入完整处理流程                                  │
│                                                                 │
│  2. 区域裁剪 (可选)                                             │
│     ├── Bounding Box 模式: 用户手绘矩形区域                      │
│     ├── Visual Selector 模式: XPath 元素定位                    │
│     └── 未配置 → 使用完整图像                                    │
│                                                                 │
│  3. 图像解码与预处理                                             │
│     ├── PNG → BGR 色彩空间                                      │
│     ├── 尺寸归一化 (如两图尺寸不同)                              │
│     └── 灰度转换 (BGR → GRAY)                                   │
│                                                                 │
│  4. 高斯模糊 (降噪)                                             │
│     ├── 模糊半径: 2*round(3*sigma) + 1 (必须为奇数)              │
│     ├── 默认 sigma: 3.0 (可通过环境变量覆盖)                     │
│     └── 目的: 消除 JPEG 压缩噪点和微小渲染差异                    │
│                                                                 │
│  5. 像素级差异计算                                              │
│     ├── cv2.absdiff(gray1, gray2)                              │
│     └── 输出: 差异图像 (每个像素值 0-255)                         │
│                                                                 │
│  6. 阈值二值化 (第一层判定)                                     │
│     ├── cv2.threshold(diff, threshold, 255, THRESH_BINARY)      │
│     └── 输出: 二值掩码 (0=无变化, 255=变化)                       │
│                                                                 │
│  7. 变化百分比计算                                              │
│     ├── changed_pixels = count_nonzero(mask)                   │
│     └── change_percentage = (changed / total) * 100             │
│                                                                 │
│  8. 最终判定 (第二层阈值)                                       │
│     └── changed_detected = change_percentage > min_change_pct   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 MD5 快速校验机制

在进行任何图像操作之前，系统首先执行 **MD5 快速校验**：

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:50-58`

```python
current_md5 = hashlib.md5(self.screenshot).hexdigest()
previous_md5 = watch.get('previous_md5')

if previous_md5 and current_md5 == previous_md5:
    # 完全相同，跳过昂贵的图像比对
    raise checksumFromPreviousCheckWasTheSame()
```

**性能优化意义**：
- 避免对完全相同的截图进行像素级比对
- MD5 计算 O(n) 时间复杂度，远低于图像处理
- 减少子进程创建和内存开销

### 2.3 核心算法实现

**代码位置**：`changedetectionio/processors/image_ssim_diff/image_handler/isolated_opencv.py:16-106`

```python
def _worker_compare(conn, img_bytes_from, img_bytes_to, 
                    pixel_difference_threshold, blur_sigma, crop_region):
    # 1. 禁用 OpenCV 多线程 (防止子进程线程爆炸)
    cv2.setNumThreads(1)
    
    # 2. 图像解码
    img_from = cv2.imdecode(np.frombuffer(img_bytes_from, np.uint8), cv2.IMREAD_COLOR)
    img_to = cv2.imdecode(np.frombuffer(img_bytes_to, np.uint8), cv2.IMREAD_COLOR)
    
    # 3. 区域裁剪
    if crop_region:
        left, top, right, bottom = crop_region
        img_from = img_from[top:bottom, left:right]
        img_to = img_to[top:bottom, left:right]
    
    # 4. 尺寸归一化
    if img_from.shape != img_to.shape:
        img_from = cv2.resize(img_from, (img_to.shape[1], img_to.shape[0]))
    
    # 5. 灰度转换
    gray_from = cv2.cvtColor(img_from, cv2.COLOR_BGR2GRAY)
    gray_to = cv2.cvtColor(img_to, cv2.COLOR_BGR2GRAY)
    
    # 6. 高斯模糊 (降噪)
    if blur_sigma > 0:
        ksize = int(2 * round(3 * blur_sigma)) + 1
        if ksize % 2 == 0:
            ksize += 1
        gray_from = cv2.GaussianBlur(gray_from, (ksize, ksize), blur_sigma)
        gray_to = cv2.GaussianBlur(gray_to, (ksize, ksize), blur_sigma)
    
    # 7. 绝对差异计算
    diff = cv2.absdiff(gray_from, gray_to)
    
    # 8. 阈值化 (第一层判定)
    _, thresholded = cv2.threshold(
        diff, 
        int(pixel_difference_threshold), 
        255, 
        cv2.THRESH_BINARY
    )
    
    # 9. 计算变化百分比
    total_pixels = thresholded.size
    changed_pixels = np.count_nonzero(thresholded)
    change_percentage = (changed_pixels / total_pixels) * 100.0
    
    conn.send(float(change_percentage))
```

---

## 3. 双层阈值判定机制

### 3.1 判定逻辑详解

系统使用 **双层阈值** 进行级联判定：

```
┌─────────────────────────────────────────────────────────────┐
│                   双层阈值判定流程                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Layer 1: 像素级敏感度 (pixel_difference_threshold)         │
│  ─────────────────────────────────────────────────          │
│  输入: 差异图像 (每个像素值 0-255)                            │
│  规则: pixel_diff >= threshold → 标记为"变化像素"             │
│  输出: 二值掩码 (变化区域)                                    │
│                                                             │
│                    │                                        │
│                    ▼                                        │
│                                                             │
│  Layer 2: 变化百分比门槛 (min_change_percentage)            │
│  ─────────────────────────────────────────────────          │
│  输入: 变化像素百分比 (0-100%)                                │
│  规则: change_pct > min_threshold → 判定为"有变化"             │
│  输出: changed_detected (True/False)                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 阈值协作示例

假设配置：
- `pixel_difference_threshold_sensitivity = 80`（中等敏感度）
- `min_change_percentage = 1`（变化 1% 触发）

**判定示例**：

| 场景 | 像素差异分布 | 第一层判定 | 变化百分比 | 最终结果 |
|------|-------------|-----------|-----------|---------|
| 完全相同 | 全部 0 | 0 像素变化 | 0% | 无变化 |
| 微小噪点 | 多个像素值 50 | 0 像素变化 | 0% | 无变化 |
| 小幅调整 | 0.5% 像素值 100 | 0.5% 像素变化 | 0.5% | 无变化 |
| 明显变化 | 2% 像素值 150 | 2% 像素变化 | 2% | **触发通知** |

### 3.3 判定代码

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:214-220`

```python
# 子进程只返回变化百分比，由调用者决定是否算"变化"
change_score = result_container[0]
if change_score is None:
    raise RuntimeError("Image comparison subprocess returned no result")

# 第二层阈值判定
changed_detected = change_score > min_change_percentage

# 详细日志
logger.info(
    f"UUID: {watch.get('uuid')} - "
    f"{process_screenshot_handler.IMPLEMENTATION_NAME}: "
    f"{change_score:.2f}% pixels changed, "
    f"pixel_diff_threshold_sensitivity: {pixel_difference_threshold_sensitivity:.0f} "
    f"score={change_score:.2f}%, "
    f"min_change_threshold={min_change_percentage}%"
)
```

---

## 4. 区域选择与裁剪策略

### 4.1 两种区域选择模式

系统支持两种区域选择模式，实现 **局部监控**：

#### 模式 A: Bounding Box (手绘矩形)

用户在 UI 上手动绘制矩形区域，坐标格式：`x,y,width,height`

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:96-108`

```python
if bounding_box:
    try:
        parts = [int(p.strip()) for p in bounding_box.split(',')]
        if len(parts) == 4:
            x, y, width, height = parts
            # 转换为 OpenCV 裁剪坐标: (left, top, right, bottom)
            crop_region = (max(0, x), max(0, y), x + width, y + height)
    except Exception as e:
        logger.warning(f"Failed to parse bounding box: {e}")
```

#### 模式 B: Visual Selector (XPath 元素定位)

通过 CSS 选择器/XPath 定位页面元素，使用元素的 bounding box

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:110-141`

```python
if not crop_region:
    include_filters = watch.get('include_filters', [])
    
    if include_filters and len(include_filters) > 0:
        first_filter = include_filters[0].strip()
        
        if first_filter and self.xpath_data:
            xpath_data_obj = json.loads(self.xpath_data)
            
            for element in xpath_data_obj.get('size_pos', []):
                if element.get('xpath') == first_filter:
                    left = element.get('left', 0)
                    top = element.get('top', 0)
                    width = element.get('width', 0)
                    height = element.get('height', 0)
                    
                    crop_region = (
                        max(0, left), 
                        max(0, top), 
                        left + width, 
                        top + height
                    )
                    break
```

### 4.2 区域选择的影响

| 模式 | 影响 | 适用场景 |
|------|------|---------|
| 无区域限制 | 比对整张截图 | 全页面监控 |
| Bounding Box | 只比对指定矩形区域 | 监控固定位置的内容 |
| Visual Selector | 只比对元素区域 | 监控动态布局中的特定元素 |

**注意**：区域裁剪发生在 **像素比对之前**，变化百分比基于裁剪后的区域计算，而非整张图。

---

## 5. 子进程隔离与内存管理

### 5.1 为什么需要子进程隔离

OpenCV 在多线程/多进程环境中存在内存管理问题，系统采用 **子进程隔离** 策略：

1. **避免内存泄漏**：子进程退出后，操作系统自动回收所有内存
2. **防止线程爆炸**：每个子进程限制 OpenCV 只用 1 个线程
3. **崩溃隔离**：子进程崩溃不影响主进程
4. **异步友好**：不阻塞主事件循环

### 5.2 子进程创建流程

**代码位置**：`changedetectionio/processors/image_ssim_diff/image_handler/isolated_opencv.py:109-197`

```python
async def compare_images_isolated(img_bytes_from, img_bytes_to, 
                                   pixel_difference_threshold, 
                                   blur_sigma, 
                                   crop_region=None):
    # 使用 spawn 方法创建干净进程 (避免 fork 问题)
    ctx = multiprocessing.get_context('spawn')
    parent_conn, child_conn = ctx.Pipe()
    
    p = ctx.Process(
        target=_worker_compare,
        args=(child_conn, img_bytes_from, img_bytes_to, 
              pixel_difference_threshold, blur_sigma, crop_region)
    )
    
    p.start()
    
    # 异步轮询，不阻塞事件循环
    deadline = time.time() + POLL_TIMEOUT_ABSOLUTE
    while time.time() < deadline:
        has_data = await asyncio.to_thread(parent_conn.poll, 0.1)
        if has_data:
            result = await asyncio.to_thread(parent_conn.recv)
            if isinstance(result, dict) and 'error' in result:
                raise RuntimeError(f"Image comparison failed: {result['error']}")
            return result
        await asyncio.sleep(0)  # 让出控制权
    
    # 超时处理
    raise TimeoutError(f"Image comparison subprocess timeout after {POLL_TIMEOUT_ABSOLUTE}s")
```

### 5.3 进程生命周期管理

```
┌──────────────┐     启动     ┌──────────────┐     退出     ┌──────────────┐
│   主进程      │─────────────▶│   子进程      │─────────────▶│  内存回收     │
│  (Flask App) │              │  (OpenCV)    │              │  (OS 自动)   │
└──────────────┘              └──────────────┘              └──────────────┘
       │                             │
       │        Pipe (双向通信)       │
       └─────────────────────────────┘
```

**清理流程**：
1. 关闭 Pipe 连接
2. 等待进程正常退出 (5s 超时)
3. 未退出则发送 SIGTERM (3s 超时)
4. 仍未退出则发送 SIGKILL (强制终止)

---

## 6. 通知触发流程

### 6.1 完整的变化检测与通知链路

```
┌──────────────────────────────────────────────────────────────────────┐
│                      变化检测与通知触发完整流程                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 1: 执行图像比对 (processor.py)                         │   │
│  │  ─────────────────────────────────────────────────           │   │
│  │  - 调用 compare_images_isolated()                            │   │
│  │  - 获取 change_percentage                                   │   │
│  │  - 判定 changed_detected = (change_pct > min_change_pct)     │   │
│  │  - 返回 (changed_detected, update_obj, screenshot_bytes)    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 2: 主流程处理 (queue_handlers.py / base.py)            │   │
│  │  ─────────────────────────────────────────────────           │   │
│  │  - 接收 changed_detected 标志                                │   │
│  │  - 保存历史快照 (如果有变化)                                  │   │
│  │  - 更新 Watch 状态 (last_checked, previous_md5 等)           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 3: 触发通知 (notification_service.py)                  │   │
│  │  ─────────────────────────────────────────────────           │   │
│  │  - 检查是否需要通知 (notification_muted? 已有未读变化?)       │   │
│  │  - 构建通知内容 (包含变化百分比、截图差异图等)                 │   │
│  │  - 发送到配置的通知渠道 (Email, Webhook, Slack 等)            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.2 处理器返回值

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:231-243`

```python
# 构造返回对象
update_obj = {
    'previous_md5': hashlib.md5(self.screenshot).hexdigest(),
    'last_error': False
}

if changed_detected:
    logger.info(f"UUID: {watch.get('uuid')} - Change detected! Score: {change_score:.2f}")
else:
    logger.debug(f"UUID: {watch.get('uuid')} - No significant change. Score: {change_score:.2f}")

# 返回三元组: (是否变化, 更新对象, 截图字节)
return changed_detected, update_obj, self.screenshot
```

### 6.3 日志记录示例

当变化被检测到时，系统会记录详细的调试信息：

```
INFO: UUID: abc123-def456 - OpenCV: 2.35% pixels changed, 
      pixel_diff_threshold_sensitivity: 80 score=2.35%, 
      min_change_threshold=1%
INFO: UUID: abc123-def456 - Change detected using OpenCV! Score: 2.35
```

---

## 7. 失败回退与异常处理

### 7.1 异常层级体系

系统定义了完整的异常处理链：

```
ProcessorException (处理器异常基类)
    │
    ├── checksumFromPreviousCheckWasTheSame (快速校验相同)
    │       └── 这不是错误，而是优化路径
    │
    ├── 图像解码失败
    │       └── ValueError: Failed to decode image
    │
    ├── 子进程超时
    │       └── TimeoutError: Image comparison subprocess timeout
    │
    └── 子进程异常
            └── RuntimeError: Image comparison failed: {error_msg}
```

### 7.2 快速校验相同的处理

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:50-58`

```python
current_md5 = hashlib.md5(self.screenshot).hexdigest()
previous_md5 = watch.get('previous_md5')

if previous_md5 and current_md5 == previous_md5:
    logger.debug(f"UUID: {watch.get('uuid')} - Screenshot MD5 unchanged, skipping comparison")
    # 抛出特殊异常，上层捕获后直接返回"无变化"
    raise checksumFromPreviousCheckWasTheSame()
```

**这不是错误**，而是一个优化路径。上层捕获此异常后：
- 不保存新的历史快照
- 不触发通知
- 只更新基本状态

### 7.3 子进程超时处理

**代码位置**：`changedetectionio/processors/image_ssim_diff/image_handler/isolated_opencv.py:142-160`

```python
deadline = time.time() + POLL_TIMEOUT_ABSOLUTE  # 默认 20 秒
while time.time() < deadline:
    has_data = await asyncio.to_thread(parent_conn.poll, 0.1)
    if has_data:
        result = await asyncio.to_thread(parent_conn.recv)
        return result
    await asyncio.sleep(0)
else:
    # 超时，记录日志并抛出异常
    logger.critical(f"[OpenCV subprocess] Timeout waiting for result after {POLL_TIMEOUT_ABSOLUTE}s")
    raise TimeoutError(f"Image comparison subprocess timeout after {POLL_TIMEOUT_ABSOLUTE}s")
```

### 7.4 处理器异常封装

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:222-229`

```python
except Exception as e:
    logger.error(f"UUID: {watch.get('uuid')} - Failed to compare screenshots: {e}")
    logger.trace(f"UUID: {watch.get('uuid')} - Processed in {time.time() - now:.3f}s")
    
    # 封装为 ProcessorException 向上抛出
    raise ProcessorException(
        message=f"UUID: {watch.get('uuid')} - Screenshot comparison failed: {e}",
        url=watch.get('url')
    )
```

### 7.5 首次检测的特殊处理

当 Watch 没有历史记录时（首次检测），系统只保存基准线：

**代码位置**：`changedetectionio/processors/image_ssim_diff/processor.py:146-158`

```python
history_keys = list(watch.history.keys())
if len(history_keys) == 0:
    # 首次检测 - 只保存基准线，不进行比较
    logger.info(f"UUID: {watch.get('uuid')} - First check - saving baseline screenshot")
    
    update_obj = {
        'previous_md5': hashlib.md5(self.screenshot).hexdigest(),
        'last_error': False
    }
    
    # 返回 changed_detected=False，不触发通知
    return False, update_obj, self.screenshot
```

---

## 8. 配置参数详解

### 8.1 核心阈值参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `pixel_difference_threshold_sensitivity` | Integer (0-255) | 80 (推荐) | 像素级敏感度阈值。值越小越敏感，0 表示任何像素变化都算。 |
| `min_change_percentage` | Integer (1-100) | 1 | 变化百分比门槛。变化像素比例超过此值才触发通知。 |

### 8.2 可通过环境变量覆盖的参数

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `OPENCV_SUBPROCESS_TIMEOUT` | 20 | 子进程超时时间（秒） |
| `OPENCV_BLUR_SIGMA` | 3.0 | 高斯模糊 sigma 值，越大降噪越强 |
| `MAX_DIFF_WIDTH` | 900 | 差异可视化最大宽度（像素） |
| `MAX_DIFF_HEIGHT` | 8000 | 差异可视化最大高度（像素） |

### 8.3 配置存储位置

| 配置类型 | 存储位置 |
|---------|---------|
| Watch 专属配置 | `{data_dir}/{watch_uuid}/image_ssim_diff.json` |
| 全局配置 | `{data_dir}/settings.json` → `application.pixel_difference_threshold_sensitivity` |
| 表单定义 | `changedetectionio/processors/image_ssim_diff/forms.py` |

### 8.4 Watch 级配置示例

`image_ssim_diff.json` 文件内容：

```json
{
    "bounding_box": "100,200,500,300",
    "selection_mode": "draw",
    "pixel_difference_threshold_sensitivity": 80,
    "min_change_percentage": 1,
    "auto_track_region": false
}
```

---

## 附录：关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `processors/image_ssim_diff/processor.py` | 主处理器，阈值解析与判定逻辑 |
| `processors/image_ssim_diff/image_handler/isolated_opencv.py` | OpenCV 图像处理（子进程） |
| `processors/image_ssim_diff/difference.py` | UI 差异渲染与可视化 |
| `processors/image_ssim_diff/forms.py` | 配置表单定义 |
| `processors/image_ssim_diff/__init__.py` | 默认值与常量定义 |
| `processors/base.py` | 处理器基类与接口定义 |
| `content_fetchers/screenshot_handler.py` | 截图获取与处理 |
| `model/Watch.py` | Watch 模型与历史管理 |

---

## 总结

Changedetection.io 的图像差异比对系统通过以下机制实现精确可控的变化检测：

1. **双层阈值判定**：像素级敏感度控制"什么算变化"，百分比门槛控制"变化多大才通知"
2. **MD5 快速校验**：避免对完全相同的图片进行昂贵处理
3. **子进程隔离**：保证内存安全和进程稳定性
4. **区域选择**：支持局部监控，减少干扰
5. **完整异常处理**：超时、解码失败、首次检测等场景均有处理
6. **可配置性**：Watch 级、全局级、环境变量三级配置覆盖

这套设计在 **性能**（快速校验、子进程隔离）和 **精确度**（双层阈值、区域选择）之间取得了良好平衡。
