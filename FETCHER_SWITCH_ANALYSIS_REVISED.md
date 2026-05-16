# Fetcher 切换机制与系统影响分析报告

## 目录
1. [fetch_backend 决策链完整分析](#1-fetch_backend-决策链完整分析)
2. [切换到 html_requests 时的系统响应](#2-切换到-html_requests-时的系统响应)
3. [切换到 html_webdriver 时的系统响应](#3-切换到-html_webdriver-时的系统响应)
4. [RSS 输出路径不变但内容变化的证据链](#4-rss-输出路径不变但内容变化的证据链)
5. [总结与关键结论](#5-总结与关键结论)

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

### 1.3 决策流程图

```
开始
  ↓
获取 watch.fetch_backend (默认 'system')
  ↓
┌─ 值为 'system' 或空? ──┐
│        是              │        否
│        ↓               │        ↓
│  使用全局设置的值      │  使用当前值
└────────────────────────┘
           ↓
┌─ 以 'extra_browser_' 开头? ─┐
│            是                │        否
│            ↓                 │        ↓
│  设置为 html_webdriver       │  继续
│  记录自定义浏览器 URL        │
└─────────────────────────────┘
           ↓
┌─ 是 PDF 文件? ────┐
│        是         │       否
│        ↓          │       ↓
│  强制 html_requests     继续
└────────────────────┘
           ↓
┌─ 有浏览器步骤且是 webdriver? ─┐
│            是                   │       否
│            ↓                    │       ↓
│  强制使用 playwright fetcher    │    继续
└─────────────────────────────────┘
           ↓
┌─ fetcher 存在吗? ───┐
│        是           │       否
│        ↓            │       ↓
│  使用该 fetcher     │  回退 html_requests
└─────────────────────┘
           ↓
         结束
```

---

## 2. 切换到 html_requests 时的系统响应

### 2.1 队列处理器响应

**队列机制**: `queue_handlers.py:RecheckPriorityQueue`

#### 2.1.1 读取新配置的时机

| 阶段 | 读取位置 | 读取内容 | 代码位置 |
|-----|---------|---------|---------|
| **入队时** | 队列不读取配置 | 仅存储 UUID 和优先级 | `queue_handlers.py:65-100` |
| **出队时** | worker 开始处理 | 仍不读取 fetcher 配置 | `worker.py:300-350` |
| **抓取前** | `call_browser()` 方法 | 读取最新 fetch_backend 配置 | `processors/base.py:133` |

**关键发现**: 队列是**完全无状态**的，配置读取仅在**实际抓取前一刻**发生。

```python
# worker.py: 调度循环
while True:
    item = await queue.async_get()  # 仅获取 UUID
    uuid = item['uuid']
    watch = datastore.data['watching'][uuid]
    # 此时才会根据最新配置创建处理器
    processor = processor_class(datastore=datastore, watch_uuid=uuid)
    # 处理器内部的 call_browser() 才真正读取 fetcher
```

#### 2.1.2 已入队任务的行为

- **切换前已入队的任务**: 当这些任务被取出时，会读取**最新的配置**，使用切换后的 `html_requests`
- **切换后新入队的任务**: 同样在抓取时读取新配置，使用 `html_requests`
- **无延迟生效**: 配置更改后，**下一次抓取就会立即生效**

### 2.2 差异计算模块响应

**文本处理器**: `processors/text_json_diff/processor.py`

#### 2.2.1 受影响的阶段

| 处理阶段 | 影响描述 | 代码位置 |
|---------|---------|---------|
| **内容获取** | fetcher.content 返回纯 HTTP 响应，无 JS 渲染 | `processor.py:463` |
| **内容预处理** | 失去浏览器渲染后的 DOM，可能缺少动态内容 | `processor.py:466-488` |
| **过滤器应用** | CSS/XPath 过滤器可能找不到动态生成的元素 | `processor.py:498-506` |
| **MD5 校验** | 内容差异导致校验和变化，可能触发误报变更 | `processor.py:585` |

#### 2.2.2 图像差异处理器可用性判断（修正版）

**重要修正**：`html_requests` 并非完全不支持截图，取决于响应的 Content-Type。

**代码证据**: `content_fetchers/requests.py:203-207`

```python
# If the content is an image, set it as screenshot for SSIM/visual comparison
content_type = r.headers.get('content-type', '').lower()
if 'image/' in content_type:
    self.screenshot = r.content
    logger.debug(f"Image content detected ({content_type}), set as screenshot for comparison")
```

**图像处理器判断逻辑**: `processors/image_ssim_diff/processor.py:42-46`

```python
if not self.fetcher.screenshot:
    raise ProcessorException(
        message="No screenshot available. Ensure the watch is configured to use a real browser.",
        url=watch.get('url')
    )
```

**两类场景的可用性判断**:

| 场景类型 | Content-Type 条件 | requests 截图可用性 | 是否需要浏览器 fetcher |
|---------|------------------|-------------------|----------------------|
| **普通网页抓取** | `text/html`, `application/json`, `text/plain` 等 | ❌ 不可用 | ✅ 需要 html_webdriver |
| **直接图像响应** | `image/png`, `image/jpeg`, `image/gif`, `image/webp` 等 | ✅ 可用 | ❌ 不需要，html_requests 即可 |

**可用性判断完整分支**:

```
图像处理器可用性判断
  ↓
┌─ 是否有 self.fetcher.screenshot? ─┐
│        ↓                          │
│    ┌─ 有值? ────────────────┐    │
│    │    ↓                   │    │
│    │  ┌─ 来源? ──────────┐  │    │
│    │  │  requests        │  │    │
│    │  │  → image/* 响应  │  │    │
│    │  └──────────────────┘  │    │
│    │    ↓                   │    │
│    │  ┌─ 来源? ──────────┐  │    │
│    │  │  webdriver       │  │    │
│    │  │  → 浏览器渲染截图 │  │    │
│    │  └──────────────────┘  │    │
│    │    ↓                   │    │
│    │  ✅ 可用                │    │
│    └─────────────────────────┘    │
│        ↓                          │
│    ❌ 不可用                       │
└───────────────────────────────────┘
```

#### 2.2.3 历史快照影响分析

**快照存储机制**: `model/Watch.py:553-605`

```python
def get_history_snapshot(self, timestamp=None, filepath=None):
    # 从磁盘读取已保存的快照文件
    filepath = self.history[timestamp]
    # 返回压缩或非压缩的文件内容
```

**历史快照不受影响的原因**:
1. **快照是不可变的**: 一旦保存到磁盘，就不会被修改 (`save_history_blob()` 只写新文件)
2. **内容比较基于磁盘快照**: `diff.render_diff()` 直接对比两个时间点的磁盘快照
3. **切换前的快照保持原样**: 使用浏览器抓取的快照继续保留浏览器渲染后的内容

**受影响的快照范围**:

| 快照类型 | 切换前 (html_webdriver) | 切换后 (html_requests) | 备注 |
|---------|-----------------------|----------------------|------|
| **普通网页文本快照** | ✅ 完整渲染内容 | ⚠️ 原始 HTML 文本 | 可能产生伪变更 |
| **图片 URL 文本快照** | ✅ 图片 MD5 | ✅ 图片 MD5 | 内容一致 |
| **普通网页截图** | ✅ PNG/JPEG 截图 | ❌ 无截图 | 图像处理器失败 |
| **图片 URL 截图** | ✅ 原始图片数据 | ✅ 原始图片数据 | **两者都可用** |

**可能的问题**:
- 普通网页切换时的第一个快照可能产生**巨大的差异**
- 图片 URL 类型的 Watch 可以在两种 fetcher 间无缝切换
- 如果原先是浏览器 fetcher 且有截图历史，切换到 requests 后**无法生成新截图对比**（除非是图片 URL）

### 2.3 图像差异处理器的特殊情况（修正版）

```python
# processors/image_ssim_diff/processor.py:42-63
def run_changedetection(self, watch, force_reprocess=False):
    # 检查截图可用性
    if not self.fetcher.screenshot:
        # 只有在非图片 URL 且使用 requests 时才会失败
        raise ProcessorException(
            message="No screenshot available. Ensure the watch is configured to use a real browser.",
            url=watch.get('url')
        )
    # 如果有截图（无论是浏览器渲染还是直接图片响应），继续处理
    self.screenshot = self.fetcher.screenshot
    # MD5 快速校验 + OpenCV 像素级对比
    current_md5 = hashlib.md5(self.screenshot).hexdigest()
    # 开始截图差异检测
```

**切换后的实际行为**:

| Watch 类型 | 切换前状态 | 切换后状态 | 是否失败 |
|-----------|-----------|-----------|---------|
| **普通网页 (text/html)** | ✅ 有截图 | ❌ 无截图 | ✅ 失败，抛出异常 |
| **图片 URL (image/png)** | ✅ 有截图 | ✅ 有截图 | ❌ 正常工作 |
| **PDF 文件** | ⚠️ 无截图 | ⚠️ 无截图 | - |

**代码验证**: `content_fetchers/requests.py:203-207` 明确表明，当检测到 `image/` Content-Type 时，requests fetcher 会自动设置 `self.screenshot`。

---

## 3. 切换到 html_webdriver 时的系统响应

### 3.1 队列处理器响应

与切换到 `html_requests` 相同的队列响应机制：

1. **配置读取时机**: 仍然在 `call_browser()` 执行时读取最新配置
2. **已入队任务**: 出队处理时使用新配置，切换到浏览器
3. **即时生效**: 切换后的第一个抓取就使用浏览器

### 3.2 差异计算模块响应

#### 3.2.1 文本处理器的新能力

| 能力 | 切换前 (html_requests) | 切换后 (html_webdriver) | 代码位置 |
|-----|-----------------------|-------------------------|---------|
| **JS 渲染** | ⚠️ 仅静态 HTML | ✅ 完整支持 | `content_fetchers/playwright.py:250-398` |
| **截图功能** | ⚠️ 仅图片 URL 支持 | ✅ 所有页面支持 | `playwright.py:270-280` |
| **XPath 元素数据** | ❌ 不支持 | ✅ 支持 | `playwright.py:396` |
| **浏览器步骤** | ❌ 不支持 | ✅ 支持 Playwright | `playwright.py:165` |

#### 3.2.2 图像处理器的启用

```python
# processors/image_ssim_diff/processor.py:42-63
def run_changedetection(self, watch, force_reprocess=False):
    # 现在 fetcher.screenshot 对所有页面都可用了
    if self.fetcher.screenshot:
        self.screenshot = self.fetcher.screenshot
        # MD5 快速校验 + OpenCV 像素级对比
        current_md5 = hashlib.md5(self.screenshot).hexdigest()
        # 开始截图差异检测
```

**切换后的变化**:
1. ✅ 所有页面类型都支持截图（不仅是图片 URL）
2. ✅ 图像差异处理器对所有 Watch 都能工作
3. ✅ 支持视觉选择器和区域对比功能
4. ✅ 支持浏览器步骤后的截图对比

### 3.3 历史快照影响

| 快照类型 | 切换前 (html_requests) | 切换后 (html_webdriver) | 存储位置 |
|---------|----------------------|------------------------|---------|
| **普通网页文本** | ⚠️ 原始 HTML 文本 | ✅ 完整渲染内容 | `{watch_dir}/history/{timestamp}.txt` |
| **图片 URL 文本** | ✅ 图片 MD5 | ✅ 图片 MD5 | `{watch_dir}/history/{timestamp}.txt` |
| **普通网页截图** | ❌ 不存在 | ✅ 开始保存 | `{watch_dir}/screenshots/{timestamp}.png` |
| **图片 URL 截图** | ✅ 原始图片数据 | ✅ 原始图片数据（格式可能不同） | `{watch_dir}/screenshots/{timestamp}.png` |
| **XPath 数据** | ❌ 不存在 | ✅ 开始保存 | `{watch_dir}/xpath/{timestamp}.json` |

**历史快照兼容性**:
- 文本快照可以跨 fetcher 比较，但内容提取方式不同可能导致**伪变更**
- 截图快照对于图片 URL 类型可以继续对比，但普通网页类型只能从切换后开始积累
- 图像处理器只能比较**都有截图**的快照对

---

## 4. RSS 输出路径不变但内容变化的证据链

### 4.1 RSS 路由路径的固定性

**代码位置**: `blueprint/rss/single_watch.py:12-115`

```python
@rss_blueprint.route("/watch/<uuid_str:uuid>", methods=['GET'])
def rss_single_watch(uuid):
    # 路由路径固定: /rss/watch/{uuid}
    # 无论使用什么 fetcher，路径完全相同
    # 返回 RSS XML 格式的变更历史
```

**关键证据**:
- ✅ 路由定义与 fetcher 类型**完全无关**
- ✅ RSS 输出路径永远是 `/rss/watch/{uuid}`
- ✅ 路径不包含任何 fetcher 标识或参数

### 4.2 场景一：动态页面 JS 渲染导致内容差异

**证据链**:

1. **抓取阶段差异**
   ```python
   # html_requests: 仅获取原始 HTML (无 JS 执行)
   # content_fetchers/requests.py: 直接 HTTP 请求
   
   # html_webdriver: 完整浏览器渲染
   # content_fetchers/playwright.py:250-398
   async def run(self, ...):
       # 等待页面 JS 执行完成
       await page.wait_for_timeout(extra_wait * 1000)
       # 获取渲染后的完整 DOM
       self.content = await self.page.content()
   ```

2. **快照保存阶段**
   ```python
   # worker.py:526-528
   watch.save_history_blob(contents=contents,  # contents 来自 fetcher
                           timestamp=int(fetch_start_time),
                           snapshot_id=update_obj.get('previous_md5', 'none'))
   ```

3. **RSS 生成阶段**
   ```python
   # blueprint/rss/single_watch.py:81-109
   # 直接从历史快照读取内容，不关心来源
   for i in range(num_diffs - 1, -1, -1):
       timestamp_to = dates[date_index_to]
       timestamp_from = dates[date_index_from]
       # 读取快照并生成差异（快照内容取决于抓取时使用的 fetcher）
       res = render_notification(...)
   ```

**结果**:
- 同一路径 `/rss/watch/{uuid}`
- 切换前的 RSS 条目包含**原始 HTML 文本**
- 切换后的 RSS 条目包含**JS 渲染后的完整内容**
- 同一 RSS feed 中可能出现两种不同质量的变更记录

### 4.3 场景二：截图依赖处理器对 RSS 的影响（修正版）

**证据链**:

1. **图像处理器失败条件**
   ```python
   # processors/image_ssim_diff/processor.py:42-46
   if not self.fetcher.screenshot:
       # 仅在以下情况抛出:
       # 1. 使用 requests fetcher
       # 2. 响应的 Content-Type 不是 image/*
       # 3. 因此 self.screenshot 为 None
       raise ProcessorException(message="No screenshot available...")
   ```

2. **成功条件（图片 URL 场景）**
   ```python
   # content_fetchers/requests.py:203-207
   content_type = r.headers.get('content-type', '').lower()
   if 'image/' in content_type:
       # 即使使用 requests fetcher，这里也会设置 screenshot
       self.screenshot = r.content
       # 图像处理器不会失败
   ```

3. **Watch 状态传播**
   ```python
   # worker.py:403-407
   except Exception as e:
       # 只有非图片 URL 使用 requests 时才会记录错误
       datastore.update_watch(uuid=uuid, update_obj={'last_error': "Exception: " + str(e)})
   ```

4. **RSS 生成时的状态差异**

   | Watch 类型 | 使用 requests fetcher | 使用 webdriver fetcher | RSS 差异 |
   |-----------|----------------------|----------------------|----------|
   | **普通网页** | ⚠️ 可能有 last_error，无截图变更 | ✅ 正常，有截图变更 | 截图差异的视觉呈现不同 |
   | **图片 URL** | ✅ 正常，有截图变更 | ✅ 正常，有截图变更 | 截图格式可能略有差异 |

5. **RSS 输出的可见影响**
   ```python
   # blueprint/rss/_util.py:130-155
   def populate_feed_entry(fe, watch, content, guid, timestamp, ...):
       # RSS 条目标题不包含截图状态
       fe.title(title=f"{watch_label} - Change @ {timestamp}")
       # 但 RSS 正文中可能引用截图链接（如果可用）
       # 普通网页 + requests: 无截图链接，可能显示错误
       # 普通网页 + webdriver: 有截图对比链接
       # 图片 URL + requests/webdriver: 都有截图对比链接
   ```

**修正后的结论**:
- RSS 路径不变，但内容完整性和可用性取决于 **Watch 类型**
- 普通网页类型：切换到 requests 可能导致视觉对比缺失
- 图片 URL 类型：两种 fetcher 都能正常工作，RSS 输出质量一致

### 4.4 场景三：rss_reader_mode 预处理模式的影响

**证据链**:

1. **RSS Reader Mode 定义**
   ```python
   # processors/text_json_diff/processor.py:466-472
   if stream_content_type.is_rss:
       content = content_processor.preprocess_rss(content)
       if self.datastore.data["settings"]["application"].get("rss_reader_mode"):
           # 将 RSS <item> 格式化为 HTML，可应用 CSS/XPath 过滤器
           stream_content_type.is_rss = False
           stream_content_type.is_html = True
           self.fetcher.content = content
   ```

2. **两种模式的输出差异**

   | rss_reader_mode | 处理方式 | 快照内容 | RSS 输出特征 |
   |-----------------|---------|---------|-------------|
   | **False (默认)** | 原始 RSS CDATA 转文本 | 纯文本内容 | 保留 XML 结构 |
   | **True** | 格式化为 HTML 列表 | 结构化 HTML | 可应用 CSS 过滤器 |

3. **切换模式对 RSS 输出的影响**
   - **全局设置生效**: `rss_reader_mode` 是全局设置，影响所有 Watch
   - **模式切换时**: 新快照使用新模式，旧快照保留原始格式
   - **RSS feed 中的混合内容**: 同一条 RSS feed 可能包含两种格式的变更记录

**代码验证**: `tests/test_rss_reader_mode.py:123, 149, 176`
```python
# 测试用例验证了不同模式下快照内容的差异
snapshot_contents = watch.get_history_snapshot(timestamp=dates[0])
# 断言: 内容格式符合预期的 rss_reader_mode 设置
```

### 4.5 RSS 内容变化的总结

| 变化维度 | 路径不变性证据 | 内容变化证据 |
|---------|--------------|-------------|
| **路由定义** | `@rss_blueprint.route("/watch/<uuid_str:uuid>")` 硬编码，无 fetcher 参数 | - |
| **快照来源** | - | 快照内容由**抓取时使用的 fetcher** 决定，后续生成 RSS 时直接读取 |
| **动态 JS 内容** | - | 浏览器 fetcher 抓取的快照有完整渲染内容 |
| **截图变更** | - | 普通网页需要浏览器 fetcher；**图片 URL 两种 fetcher 都支持** |
| **rss_reader_mode** | - | 全局设置影响预处理，但路径完全相同 |

---

## 5. 总结与关键结论

### 5.1 核心发现

#### 5.1.1 Fetcher 决策
- ✅ **6 层决策链**确保正确的 fetcher 选择
- ✅ **PDF 和浏览器步骤有最高优先级**，会覆盖用户配置
- ✅ **决策发生在抓取前一刻**，保证配置即时生效

#### 5.1.2 队列响应
- ✅ **队列完全无状态**，不存储任何 fetcher 配置
- ✅ **配置读取延迟到抓取前**，已入队任务也能使用新配置
- ✅ **零延迟生效**，切换后第一个抓取就使用新 fetcher

#### 5.1.3 差异计算（修正版）
- ✅ **历史快照不可变**，切换不影响已保存的内容
- ✅ **切换时的第一个快照可能产生伪变更**，需注意
- ✅ **图像处理器可用性条件修正**：
  - ❌ **旧错误结论**: "切换到 html_requests 会立即失败"
  - ✅ **正确结论**: "切换到 html_requests 时，**普通网页**会失败，但**图片 URL 类型仍可正常工作**"
- ✅ **关键判断条件**: `content_type` 是否包含 `image/` 前缀

#### 5.1.4 RSS 输出
- ✅ **路径永远不变**，始终是 `/rss/watch/{uuid}`
- ✅ **内容质量取决于快照生成时的 fetcher**
- ✅ **同一条 RSS feed 可能包含混合质量的变更记录**
- ✅ **图片 URL 类型的 Watch 在两种 fetcher 下 RSS 质量一致**

### 5.2 截图可用性判断完整矩阵

| Fetcher 类型 | Watch 类型 | Content-Type | Screenshot 可用性 | 图像处理器状态 |
|-------------|-----------|-------------|------------------|---------------|
| html_webdriver | 普通网页 | text/html | ✅ 浏览器渲染截图 | 正常工作 |
| html_webdriver | 图片 URL | image/* | ✅ 浏览器截图 | 正常工作 |
| html_requests | 普通网页 | text/html | ❌ 无 | 抛出异常 |
| html_requests | 图片 URL | image/png/jpeg 等 | ✅ 原始响应内容 | 正常工作 |
| html_requests | PDF 文件 | application/pdf | ❌ 无 | 不适用（文本处理） |

### 5.3 架构设计优点

1. **关注点分离**: 队列调度、fetcher 选择、差异计算、RSS 生成分层清晰
2. **向后兼容**: 动态选择机制保证配置变更平滑过渡
3. **灵活性**: 支持全局、单 Watch、自定义浏览器多级配置
4. **智能降级**: 对于图片 URL，requests fetcher 自动提供截图能力，无需浏览器开销

### 5.4 潜在风险与建议

| 风险场景 | 建议 |
|---------|-----|
| 切换 fetcher 时的巨大伪变更 | 在切换后第一次抓取时标记为"基准快照"，不触发通知 |
| 普通网页使用 requests + 图像处理器失败 | UI 中增加提示："图像处理器建议配合浏览器 fetcher 使用，或直接监听图片 URL" |
| RSS 内容质量不一致 | 在 RSS 条目中增加元数据，标识使用的 fetcher 类型和快照来源类型 |
| 自定义浏览器配置失效 | 增加配置有效性检查和降级策略 |
| 图片 URL 在切换时的格式差异 | 统一截图存储格式，确保跨 fetcher 可比性 |

---

## 附录：关键代码位置速查表

| 功能 | 文件路径 | 行号 |
|-----|---------|-----|
| requests fetcher 截图设置 | `content_fetchers/requests.py` | 203-207 |
| 图像处理器截图检查 | `processors/image_ssim_diff/processor.py` | 42-46 |
| 主 fetcher 决策逻辑 | `processors/base.py` | 117-192 |
| Playwright 截图捕获 | `content_fetchers/playwright.py` | 410 |
| RSS 路由定义 | `blueprint/rss/single_watch.py` | 12 |
| 快照保存函数 | `model/Watch.py` | 553-605 |

*报告生成时间: 2026-05-16*  
*报告版本: v2.0 (修正截图可用性判断)*  
*代码分析基于: changedetection.io 源码*
