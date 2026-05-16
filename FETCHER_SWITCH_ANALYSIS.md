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

#### 2.2.2 历史快照影响分析

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

| 快照时间 | 是否受影响 | 影响描述 |
|---------|-----------|---------|
| **切换前的所有快照** | ❌ 不受影响 | 已保存的快照内容不变 |
| **切换后的第一个快照** | ✅ 受影响 | 使用 html_requests 抓取，内容可能与之前不同 |
| **切换后的后续快照** | ✅ 受影响 | 全部使用 html_requests |

**可能的问题**: 切换时的第一个快照可能产生**巨大的差异**，因为内容提取方式变化了。

### 2.3 图像差异处理器的特殊情况

**代码位置**: `processors/image_ssim_diff/processor.py`

```python
if not self.fetcher.screenshot:
    raise ProcessorException(
        message="No screenshot available. Ensure the watch is configured to use a real browser.",
        url=watch.get('url')
    )
```

- **切换后果**: 图像处理器会**立即失败**，因为 `html_requests` 不提供 `screenshot` 属性
- **错误信息**: "No screenshot available. Ensure the watch is configured to use a real browser."
- **恢复方式**: 必须重新切换回浏览器 fetcher 才能恢复截图对比功能

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
| **JS 渲染** | ❌ 不支持 | ✅ 完整支持 | `content_fetchers/playwright.py:250-398` |
| **截图功能** | ❌ 不支持 | ✅ 支持 PNG/JPEG | `playwright.py:270-280` |
| **XPath 元素数据** | ❌ 不支持 | ✅ 支持 | `playwright.py:396` |
| **浏览器步骤** | ❌ 不支持 | ✅ 支持 Playwright | `playwright.py:165` |

#### 3.2.2 图像处理器的启用

```python
# processors/image_ssim_diff/processor.py:42-63
def run_changedetection(self, watch, force_reprocess=False):
    # 现在 fetcher.screenshot 可用了
    if self.fetcher.screenshot:
        self.screenshot = self.fetcher.screenshot
        # MD5 快速校验 + OpenCV 像素级对比
        current_md5 = hashlib.md5(self.screenshot).hexdigest()
        # 开始截图差异检测
```

**切换后的变化**:
1. ✅ `fetcher.screenshot` 现在包含截图字节
2. ✅ 图像差异处理器不再抛出异常
3. ✅ 截图文件开始保存到 watch 数据目录
4. ✅ 可以使用视觉选择器和区域对比

### 3.3 历史快照影响

| 快照类型 | 切换前 | 切换后 | 存储位置 |
|---------|-------|-------|---------|
| **文本快照** | html_requests 抓取 | 浏览器渲染后抓取 | `{watch_dir}/history/{timestamp}.txt` |
| **截图快照** | ❌ 不存在 | ✅ 开始保存 | `{watch_dir}/screenshots/{timestamp}.png` |
| **XPath 数据** | ❌ 不存在 | ✅ 开始保存 | `{watch_dir}/xpath/{timestamp}.json` |

**历史快照兼容性**:
- 文本快照可以跨 fetcher 比较，但内容提取方式不同可能导致**伪变更**
- 截图快照只能在切换后开始积累，无法回溯生成之前的截图
- 图像处理器只能比较**切换后的**截图快照

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

### 4.3 场景二：截图依赖处理器对 RSS 的影响

**证据链**:

1. **图像处理器失败状态** (切换到 html_requests 后)
   ```python
   # processors/image_ssim_diff/processor.py
   if not self.fetcher.screenshot:
       raise ProcessorException(message="No screenshot available...")
   ```

2. **错误传播到 Watch 状态**
   ```python
   # worker.py:403-407
   except Exception as e:
       datastore.update_watch(uuid=uuid, update_obj={'last_error': "Exception: " + str(e)})
   ```

3. **RSS 生成时的状态**
   - Watch 有 `last_error` 但 RSS 路由**不检查错误状态**
   - RSS 仍然基于**历史快照**生成（只要快照存在）
   - 切换到浏览器后，新快照包含截图，旧快照没有

4. **RSS 输出的可见影响**
   ```python
   # blueprint/rss/_util.py:130-155
   def populate_feed_entry(fe, watch, content, guid, timestamp, ...):
       # RSS 条目标题不包含截图状态
       fe.title(title=f"{watch_label} - Change @ {timestamp}")
       # 但 RSS 正文中可能引用截图链接（如果可用）
       # 切换前的变更: 无截图链接
       # 切换后的变更: 有截图对比链接
   ```

**结果**:
- RSS 路径不变，但内容完整性变化
- 切换前: RSS 可能显示错误或缺少视觉变更
- 切换后: RSS 包含完整的截图变更信息

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
   - **模式切换时**: 新快照使用新模式，旧快照保持原始格式
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
| **截图变更** | - | 只有浏览器 fetcher 能提供截图，图像处理器的 RSS 输出不同 |
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

#### 5.1.3 差异计算
- ✅ **历史快照不可变**，切换不影响已保存的内容
- ✅ **切换时的第一个快照可能产生伪变更**，需注意
- ✅ **图像处理器强依赖浏览器**，切换到 requests 会立即失败

#### 5.1.4 RSS 输出
- ✅ **路径永远不变**，始终是 `/rss/watch/{uuid}`
- ✅ **内容质量取决于快照生成时的 fetcher**
- ✅ **同一条 RSS feed 可能包含混合质量的变更记录**

### 5.2 架构设计优点

1. **关注点分离**: 队列调度、fetcher 选择、差异计算、RSS 生成分层清晰
2. **向后兼容**: 动态选择机制保证配置变更平滑过渡
3. **灵活性**: 支持全局、单 Watch、自定义浏览器多级配置
4. **优雅降级**: 能力不匹配时自动回退或抛出明确错误

### 5.3 潜在风险与建议

| 风险场景 | 建议 |
|---------|-----|
| 切换 fetcher 时的巨大伪变更 | 在切换后第一次抓取时标记为"基准快照"，不触发通知 |
| 图像处理器因切换失败 | 在 UI 中明确提示处理器与 fetcher 的兼容性要求 |
| RSS 内容质量不一致 | 在 RSS 条目中增加元数据，标识使用的 fetcher 类型 |
| 自定义浏览器配置失效 | 增加配置有效性检查和降级策略 |

---

*报告生成时间: 2026-05-16*  
*代码分析基于: changedetection.io 源码*
