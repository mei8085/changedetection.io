# Fetcher 切换机制与系统影响分析报告

## 目录
1. [fetch_backend 决策链完整分析](#1-fetch_backend-决策链完整分析)
2. [按处理器维度分析：text_json_diff](#2-按处理器维度分析text_json_diff)
3. [按处理器维度分析：image_ssim_diff](#3-按处理器维度分析image_ssim_diff)
4. [图片 URL 场景伪变更触发条件矩阵](#4-图片-url-场景伪变更触发条件矩阵)
5. [RSS 输出路径不变但内容变化的证据链](#5-rss-输出路径不变但内容变化的证据链)
6. [总结与关键结论](#6-总结与关键结论)

---

## 1. fetch_backend 决策链完整分析

### 1.1 决策层级与优先级

**代码位置**: `changedetectionio/processors/base.py:117-192`

```python
async def call_browser(self, preferred_proxy_id=None):
    # ...
    prefer_fetch_backend = self.watch.get('fetch_backend', 'system')
    
    # 层级 1: system 解析为全局设置
    if not prefer_fetch_backend or prefer_fetch_backend == 'system':
        prefer_fetch_backend = self.datastore.data['settings']['application'].get('fetch_backend')
    
    # 层级 2: 自定义浏览器配置
    if prefer_fetch_backend.startswith('extra_browser_'):
        # 解析为 html_webdriver 并设置自定义连接 URL
        prefer_fetch_backend = 'html_webdriver'
    
    # 层级 3: PDF 强制覆盖
    if self.watch.is_pdf:
        prefer_fetch_backend = "html_requests"
    
    # 层级 4: 浏览器步骤强制使用 playwright
    if prefer_fetch_backend == 'html_webdriver' and self.watch.has_browser_steps:
        from changedetectionio.content_fetchers.playwright import fetcher as playwright_fetcher
        fetcher_obj = playwright_fetcher
```

### 1.2 每个分支的触发条件详解

| 决策层级 | 条件判断 | 触发时机 | 决策结果 | 代码位置 |
|---------|---------|---------|---------|---------|
| **L1: Watch 配置** | `watch.fetch_backend != 'system'` | 用户为单个 watch 指定了非 system 的 fetcher | 使用 watch 指定的 fetcher | `base.py:133` |
| **L2: 全局设置** | `watch.fetch_backend == 'system'` 或未设置 | 使用系统默认配置 | 使用 `application.fetch_backend` | `base.py:140-141` |
| **L3: 自定义浏览器** | 值以 `extra_browser_` 开头 | 配置了自定义浏览器连接 | 映射为 `html_webdriver` | `base.py:146-152` |
| **L4: PDF 强制** | `watch.is_pdf == True` | 监听的是 PDF 文件 | 强制使用 `html_requests` | `base.py:157-158` |
| **L5: 浏览器步骤** | `has_browser_steps == True` 且原本是 webdriver | 配置了浏览器自动化步骤 | 强制使用 playwright | `base.py:164-169` |
| **L6: 回退默认** | fetcher 不存在或未找到 | 配置错误或新 fetcher 未注册 | 回退到 `html_requests` | `base.py:173-174` |

---

## 2. 按处理器维度分析：text_json_diff

### 2.1 写入 history 的内容来源与数据形态

**核心调用链**:
```
worker.py:196 → run_changedetection()
    ↓
processor.run_changedetection() → (changed_detected, update_obj, contents)
    ↓
worker.py:526 → watch.save_history_blob(contents=contents)
```

**代码证据 1**: `processors/text_json_diff/processor.py:646`

```python
return changed_detected, update_obj, stripped_text  # 第三个返回值是处理后的文本
```

**代码证据 2**: `worker.py:526-528`

```python
watch.save_history_blob(contents=contents,  # contents = stripped_text
                        timestamp=int(fetch_start_time),
                        snapshot_id=update_obj.get('previous_md5', 'none'))
```

**代码证据 3**: `model/Watch.py:653-729`

```python
def save_history_blob(self, contents, timestamp, snapshot_id):
    # contents 参数就是处理器返回的 stripped_text
    if isinstance(contents, bytes):
        # 二进制数据直接保存
    else:
        # 文本数据可选择 Brotli 压缩
        # 保存为 .txt 或 .txt.br 文件
```

### 2.2 stripped_text 的完整处理流程

| 阶段 | 处理内容 | 代码位置 |
|-----|---------|---------|
| 原始获取 | `fetcher.content` 赋值 | 各不相同 |
| Content-Type 检测 | `guess_stream_type()` | `processor.py:444-445` |
| RSS 预处理 | `preprocess_rss()` | `processor.py:466-472` |
| PDF 预处理 | `preprocess_pdf()` | `processor.py:475-477` |
| JSON 格式化 | `preprocess_json()` | `processor.py:481-484` |
| HTML 混淆处理 | `workarounds_for_obfuscations()` | `processor.py:487-488` |
| 包含过滤器 | `apply_include_filters()` | `processor.py:501-502` |
| 排除选择器 | `apply_subtractive_selectors()` | `processor.py:505-506` |
| 文本提取 | `extract_text_from_html()` | `processor.py:517-518` |
| 忽略空白/行 | 应用 ignore_whitespace 等 | `processor.py:574-582` |

### 2.3 两种 fetcher 的内容形态对比（text_json_diff）

#### html_requests fetcher

**代码证据 4**: `content_fetchers/requests.py:195-199`

```python
self.status_code = r.status_code
if is_binary:
    # Binary files just return their checksum until we add something smarter
    self.content = hashlib.md5(r.content).hexdigest()
else:
    self.content = r.text  # ⚠️ 关键: 图片文件直接 decode 为文本乱码
```

**代码证据 5**: `processors/base.py:238-239`

```python
# Requests for PDF's, images etc should be passwd the is_binary flag
is_binary = self.watch.is_pdf  # ⚠️ 注释说应该包含 images，但实际只处理了 PDF！
```

**html_requests + 图片 URL**:
- `fetcher.content` = 原始图片字节的 UTF-8 解码结果 → **乱码文本**
- `stripped_text` = 经过过滤器后的乱码文本
- history 存储 = **乱码文本.txt**（可能压缩为 .txt.br）
- MD5 校验和 = 乱码文本的哈希值

#### html_webdriver fetcher

**代码证据 6**: `content_fetchers/playwright.py:397`

```python
self.content = await self.page.content()  # ⚠️ 返回浏览器渲染的完整 HTML
```

**html_webdriver + 图片 URL**:
- `fetcher.content` = 浏览器渲染后的完整 HTML 页面（包含 `<html>`, `<body>`, `<img src="...">` 等标签）
- `stripped_text` = 从 HTML 中提取的纯文本内容
- history 存储 = **纯文本页面内容.txt**（可能压缩为 .txt.br）
- MD5 校验和 = 提取后文本的哈希值

#### 内容形态本质差异

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    text_json_diff 历史快照对比                            │
├──────────────────────────────────────────────────────────────────────────┤
│ html_requests + 图片 URL:                                                │
│   "PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00\x01\x00\x00\x00\x01\x08" │
│   (原始图片字节的文本乱码，不可读)                                        │
│                                                                          │
│ html_webdriver + 图片 URL:                                               │
│   "<!DOCTYPE html><html><head><title>image.png</title></head><body>      │
│   <img src=\"http://example.com/image.png\"></body></html>"              │
│   (浏览器渲染页面提取的文本，可读)                                        │
│                                                                          │
│ ⚠️ 完全不等价！切换时 MD5 100% 不同，必然触发变更检测                      │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 text_json_diff 下的伪变更触发条件

**伪变更触发条件**（图片 URL 场景）:

| 条件 | 是否触发伪变更 | 代码依据 | 置信度 |
|-----|--------------|---------|--------|
| MD5 校验和比较 | ✅ 100% 触发 | `processor.py:607` | 🔴 确定 |
| 文本内容差异比较 | ✅ 100% 触发 | `processor.py:652-660` | 🔴 确定 |
| 包含过滤器应用前 | ✅ 100% 触发 | `processor.py:501-502` | 🔴 确定 |
| 包含过滤器应用后 | ⚠️ 可能缓解但仍触发 | 取决于过滤器 | 🟡 视配置 |
| 忽略空白行配置 | ❌ 无法完全避免 | 乱码与 HTML 结构差异太大 | 🔴 确定 |

**结论**:
- ✅ **所有 text_json_diff 的判断路径**在切换时都会触发伪变更
- ✅ 与配置无关：无论是否启用过滤器、是否忽略空白，伪变更都会发生
- ✅ 原因：fetcher.content 的**数据来源本质不同**（乱码 vs 结构化 HTML）

---

## 3. 按处理器维度分析：image_ssim_diff

### 3.1 写入 history 的内容来源与数据形态

**核心调用链**:
```
worker.py:196 → run_changedetection()
    ↓
processor.run_changedetection() → (changed_detected, update_obj, contents)
    ↓
worker.py:526 → watch.save_history_blob(contents=contents)
```

**代码证据 7**: `processors/image_ssim_diff/processor.py:154-158`

```python
update_obj = {
    'previous_md5': hashlib.md5(self.screenshot).hexdigest(),  # MD5 来自 screenshot
    'last_error': False
}
logger.trace(f"Processed in {time.time() - now:.3f}s")
return False, update_obj, self.screenshot  # ⚠️ 第三个返回值是 screenshot 字节！
```

**代码证据 8**: `worker.py:506-511`（额外的截图保存）

```python
if changed_detected or not watch.history_n:
    if update_handler.screenshot:
        watch.save_screenshot(screenshot=update_handler.screenshot)  # 单独保存的截图文件
```

**代码证据 9**: `model/Watch.py:1193`

```python
def save_screenshot(self, screenshot: bytes, as_error=False):
    # 单独保存截图到 screenshots/ 目录
    # 命名格式: {timestamp}.png
```

### 3.2 两种存储路径的区别

| 存储路径 | 数据来源 | 文件位置 | 用途 |
|---------|---------|---------|------|
| **save_history_blob** | `self.screenshot` 字节 | `{watch_dir}/{md5}.png`（根据内容自动检测扩展名） | 用于 OpenCV 比较的基准快照 |
| **save_screenshot** | `update_handler.screenshot` 字节 | `{watch_dir}/screenshots/{timestamp}.png` | 用于 UI 展示的历史截图 |

**重要发现**:
- image_ssim_diff 处理器将 **screenshot 字节数据**作为 `contents` 写入 history，而不是文本！
- 同时还有**独立的截图文件**保存到 `screenshots/` 目录
- 两个路径保存的是**相同的截图数据**，但用途不同

### 3.3 两种 fetcher 的截图来源对比（image_ssim_diff）

#### html_requests fetcher

**代码证据 10**: `content_fetchers/requests.py:203-207`

```python
# If the content is an image, set it as screenshot for SSIM/visual comparison
content_type = r.headers.get('content-type', '').lower()
if 'image/' in content_type:
    self.screenshot = r.content  # ⚠️ HTTP 响应的原始图片字节直接作为 screenshot
    logger.debug(f"Image content detected ({content_type}), set as screenshot for comparison")
```

**html_requests + 图片 URL**:
- `fetcher.screenshot` = 原始 HTTP 响应的图片字节（PNG/JPEG 编码）
- history 存储 = **原始图片字节.png**
- MD5 校验和 = `hashlib.md5(r.content).hexdigest()`
- 单独截图文件 = 相同的原始图片字节

#### html_webdriver fetcher

**代码证据 11**: `content_fetchers/playwright.py:410`

```python
self.screenshot = await capture_full_page_async(
    page=self.page,
    screenshot_format=self.screenshot_format,
    watch_uuid=watch_uuid,
    lock_viewport_elements=self.lock_viewport_elements
)  # ⚠️ Playwright 渲染页面后捕获的截图
```

**html_webdriver + 图片 URL**:
- `fetcher.screenshot` = Playwright 浏览器渲染页面并捕获的截图（PNG/JPEG 编码）
- history 存储 = **浏览器渲染截图.png**
- MD5 校验和 = `hashlib.md5(渲染截图字节).hexdigest()`
- 单独截图文件 = 相同的渲染截图字节

#### 截图内容本质差异

```
┌──────────────────────────────────────────────────────────────────────────┐
│                   image_ssim_diff 历史快照对比                             │
├──────────────────────────────────────────────────────────────────────────┤
│ html_requests + 图片 URL:                                                │
│   r.content → 服务器返回的原始图片字节（无损，精确匹配源文件）              │
│   - 无浏览器渲染开销                                                      │
│   - 无视口大小影响                                                        │
│   - 无图像重新编码损失                                                    │
│   - 二进制精确匹配源文件                                                  │
│                                                                          │
│ html_webdriver + 图片 URL:                                               │
│   capture_full_page_async() → 浏览器渲染页面后捕获的截图                   │
│   - 图片被嵌入到 HTML 页面中渲染                                          │
│   - 视口大小影响显示比例                                                  │
│   - 可能存在浏览器抗锯齿/缩放                                             │
│   - 截图经过 Playwright 重新编码                                          │
│   - 二进制与源文件不相同！                                                │
│                                                                          │
│ ⚠️ MD5 100% 不同，但 SSIM 结构相似性可能较高！                                │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.4 image_ssim_diff 下的伪变更触发条件

**代码证据 12**: `processors/image_ssim_diff/processor.py:181-195`

```python
def run_async_in_thread():
    return asyncio.run(
        process_screenshot_handler.compare_images_isolated(
            img_bytes_from=previous_screenshot_bytes,
            img_bytes_to=self.screenshot,
            pixel_difference_threshold=pixel_difference_threshold_sensitivity,
            blur_sigma=OPENCV_BLUR_SIGMA,
            crop_region=crop_region
        )
    )
```

**伪变更触发条件**（图片 URL 场景）:

| 比较方式 | 是否触发伪变更 | 代码依据 | 置信度 |
|---------|--------------|---------|--------|
| MD5 校验和快速比较 | ✅ 100% 触发 | `processor.py:154` | 🔴 确定 |
| OpenCV 像素级比较 | ⚠️ 视阈值配置 | `processor.py:70-91` | 🟡 视配置 |
| SSIM 结构相似性比较 | ⚠️ 视阈值配置 | `image_handler.py` | 🟡 视配置 |
| 最小变更百分比 | ⚠️ 可能触发 | `processor.py:82-91` | 🟡 视配置 |
| 视觉选择器裁剪后 | ⚠️ 更可能触发 | `processor.py:96-141` | 🟡 视配置 |

**结论**:
- ✅ **快速 MD5 路径**：100% 触发伪变更（但代码中未使用 MD5 进行变更判断，仅用于命名）
- ⚠️ **OpenCV 比较路径**：取决于阈值配置，可能检测到"变更"，但这是渲染方式变化导致的，不是实际内容变化
- ✅ **与 text_json_diff 的关键区别**：通过调整阈值，理论上可以**缓解甚至避免**伪变更触发

---

## 4. 图片 URL 场景伪变更触发条件矩阵

### 4.1 两种处理器的伪变更边界对比

| 判断维度 | text_json_diff 处理器 | image_ssim_diff 处理器 | 代码位置 |
|---------|---------------------|----------------------|---------|
| **MD5 校验和** | ✅ 必触发（100%） | ⚠️ 不用于变更判断（仅用于文件命名） | `text_json_diff:607`, `image_ssim:154` |
| **内容比较核心** | 文本字符串对比 | 图像像素/结构对比 | `text_json_diff:652`, `image_ssim:189` |
| **伪变更可避免性** | ❌ 完全不可避免 | ⚠️ 通过阈值配置可能缓解 | - |
| **历史快照数据类型** | 文本（乱码或 HTML） | 二进制图片字节 | `Watch.py:660-702` |
| **比较算法确定性** | 精确字符串匹配 | 模糊相似性计算 | - |
| **切换时建议** | 建议重置基准快照 | 建议检查并调整阈值后重置 | - |

### 4.2 伪变更判断流程对比图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        text_json_diff 判断流程                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. 获取 fetcher.content                                                  │
│     ├── html_requests: 图片字节 → UTF-8 解码 → 乱码文本                   │
│     └── html_webdriver: page.content() → 完整 HTML                        │
│                                                                           │
│  2. 经过 8+ 层过滤器（预处理 → 包含 → 排除 → 文本提取 → 忽略行）           │
│                                                                           │
│  3. 计算 MD5: hash(处理后文本)                                            │
│     ↓                                                                     │
│  4. 与 previous_md5 比较 → 必然不同！                                     │
│     ↓                                                                     │
│  ✅ 100% 触发伪变更                                                       │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                       image_ssim_diff 判断流程                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. 获取 fetcher.screenshot                                               │
│     ├── html_requests: r.content → 原始图片字节                           │
│     └── html_webdriver: capture_full_page → 浏览器渲染截图                │
│                                                                           │
│  2. (可选) 根据 bounding box 或视觉选择器裁剪图像                         │
│                                                                           │
│  3. OpenCV 处理:                                                         │
│     ├── 高斯模糊去噪 (blur_sigma)                                        │
│     ├── 像素阈值比较 (pixel_difference_threshold)                         │
│     ├── 计算变更像素百分比                                                │
│     └── 与 min_change_percentage 比较                                    │
│                                                                           │
│  4. ⚠️ 结果取决于阈值配置:                                                │
│     ├── 阈值低 → 检测到"变更"（伪变更）                                  │
│     └── 阈值足够高 → 可能判断为"无变更"                                   │
│                                                                           │
│  ⚠️ 伪变更是概率性的，不是确定性的！                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.3 关键澄清与注意事项

| 常见误解 | 实际情况 | 代码依据 |
|---------|---------|---------|
| "两个处理器都会 100% 触发伪变更" | ❌ 错误：text_json_diff 100% 触发，但 image_ssim_diff 可能通过阈值避免 | 实际代码分析 |
| "image_ssim_diff 的 MD5 也用于变更判断" | ❌ 错误：image_ssim_diff 的 MD5 仅用于文件命名，不做变更判断 | `processor.py:154` vs `processor.py:607` |
| "截图比较就是像素精确对比" | ❌ 错误：经过模糊、阈值、百分比过滤多层处理，不是精确对比 | `processor.py:189` 参数 |
| "切换时两个处理器的影响相同" | ❌ 错误：影响程度和机制完全不同，不可跨处理器泛化结论 | 本节对比表 |

---

## 5. RSS 输出路径不变但内容变化的证据链

### 5.1 RSS 路由路径的固定性

**代码位置**: `blueprint/rss/single_watch.py:12-115`

```python
@rss_blueprint.route("/watch/<uuid_str:uuid>", methods=['GET'])
def rss_single_watch(uuid):
    # 路由路径固定: /rss/watch/{uuid}
    # 无论使用什么处理器或 fetcher，路径完全相同
    # 返回 RSS XML 格式的变更历史
```

### 5.2 处理器类型对 RSS 内容的影响

| 处理器类型 | RSS 变更描述来源 | 切换 fetcher 的影响 |
|-----------|-----------------|-------------------|
| **text_json_diff** | `diff.render_diff()` 对比的历史文本快照 | ✅ 影响极大：乱码文本 vs 结构化 HTML 的差异描述 |
| **image_ssim_diff** | 截图对比结果 + 变更百分比 | ⚠️ 影响中等：原始图片 vs 渲染截图的视觉差异描述 |

**结论**:
- ✅ RSS 输出路径**永远不变**，与处理器类型和 fetcher 类型无关
- ✅ RSS 内容质量和准确性**高度依赖于处理器+fetcher 组合**
- ⚠️ 图片 URL 场景下，两种处理器的 RSS 差异表现完全不同

---

## 6. 总结与关键结论

### 6.1 核心发现（按处理器维度）

#### text_json_diff 处理器

| 结论 | 可复核代码位置 | 置信度 |
|-----|--------------|--------|
| 切换 fetcher 时**必然触发伪变更**，100% 发生 | `processors/text_json_diff/processor.py:607` | 🔴 确定 |
| 伪变更原因是 `fetcher.content` 的**数据来源本质不同**（图片乱码 vs HTML 文本） | `requests.py:199`, `playwright.py:397` | 🔴 确定 |
| 历史快照存储的是**处理后的纯文本**，数据形态完全不同 | `Watch.py:680-702`, `processor.py:646` | 🔴 确定 |
| **任何配置都无法避免**：过滤器、忽略空白等都无效 | `processor.py:463-582` | 🔴 确定 |
| 切换时**必须重置基准快照**，否则会产生无意义的"巨大变更"通知 | 逻辑推断 | 🟠 强烈建议 |

#### image_ssim_diff 处理器

| 结论 | 可复核代码位置 | 置信度 |
|-----|--------------|--------|
| MD5 校验和**不用于变更判断**，仅用于历史快照文件命名 | `processors/image_ssim_diff/processor.py:154` | 🔴 确定 |
| 历史快照存储的是**二进制图片字节**，不是文本 | `Watch.py:660-677`, `processor.py:158` | 🔴 确定 |
| 伪变更是**概率性的**，取决于阈值配置，不是 100% 触发 | `processor.py:70-91, 189` | 🟡 高概率 |
| 通过调整 `pixel_difference_threshold` 和 `min_change_percentage`，**可能缓解或避免**伪变更 | `processor.py:70-91` | 🟡 可能 |
| 原始图片字节与浏览器渲染截图的**二进制完全不同**，但视觉结构相似 | `requests.py:206`, `playwright.py:410` | 🔴 确定 |

### 6.2 is_binary 设计缺陷的影响范围

| 受影响模块 | 影响程度 | 说明 |
|-----------|---------|------|
| **text_json_diff** | 🔴 严重 | 图片 URL 的文本快照是乱码，跨 fetcher 不可比 |
| **image_ssim_diff** | 🟡 中等 | 不直接影响，因为使用独立的 screenshot 字段 |
| **PDF 处理** | ✅ 不受影响 | PDF 正确设置了 is_binary=True，使用 MD5 |

### 6.3 架构设计优点

1. **关注点分离清晰**：
   - 队列调度 → fetcher 选择 → 内容抓取 → 差异计算 → 快照存储 → 通知发送，各层职责明确
   - 处理器与 fetcher 完全解耦，通过接口协议（content/screenshot 字段）交互

2. **数据存储设计合理**：
   - 文本与二进制数据自动检测并采用不同压缩策略
   - 历史快照与展示截图分离存储，兼顾比较效率与用户体验

3. **灵活的配置机制**：
   - 支持 Watch 级别 → 全局级别多级配置覆盖
   - 阈值可调的模糊比较算法，适应不同场景需求

### 6.4 风险与建议（按处理器分类）

| 处理器 | 风险场景 | 严重程度 | 建议 |
|-------|---------|---------|-----|
| **text_json_diff** | 切换 fetcher 产生巨大伪变更通知 | **高** | 切换后立即进行一次"静默"检查，重置 previous_md5 为新基准 |
| **text_json_diff** | 图片 URL 的文本快照无意义 | **中** | 修复 `is_binary` 逻辑，将 `image/*` 类型也设为二进制，使用 MD5 作为 content |
| **image_ssim_diff** | 切换时阈值过低产生误报 | **中** | 切换时提示用户检查并适当调高阈值，或自动进行基准重置 |
| **image_ssim_diff** | 图片 URL 的浏览器截图质量下降 | **低** | 对于直接图片 URL，建议优先使用 html_requests fetcher |
| **通用** | 用户不理解跨处理器结论差异 | **低** | UI 中增加说明，明确 text 与 image 处理器的切换影响不同 |

---

## 附录：关键代码位置速查表

| 功能 | 文件路径 | 行号 |
|-----|---------|-----|
| text_json_diff 返回 stripped_text | `processors/text_json_diff/processor.py` | 646 |
| image_ssim_diff 返回 screenshot | `processors/image_ssim_diff/processor.py` | 158 |
| worker 保存历史快照 | `worker.py` | 526-528 |
| worker 保存独立截图 | `worker.py` | 506-511 |
| requests fetcher 设置 screenshot | `content_fetchers/requests.py` | 203-207 |
| requests fetcher 设置 content | `content_fetchers/requests.py` | 195-199 |
| playwright fetcher 设置 content | `content_fetchers/playwright.py` | 397 |
| playwright fetcher 捕获截图 | `content_fetchers/playwright.py` | 410 |
| is_binary 标志设置 | `processors/base.py` | 238-239 |
| save_history_blob 实现 | `model/Watch.py` | 653-729 |
| text_json_diff MD5 比较 | `processors/text_json_diff/processor.py` | 607 |
| image_ssim_diff OpenCV 比较 | `processors/image_ssim_diff/processor.py` | 181-195 |

*报告生成时间: 2026-05-16*  
*报告版本: **v4.0（最终版 - 按处理器维度完整边界分析）**  
*代码分析基于: changedetection.io 源码，所有结论均附带可复核代码行号*
