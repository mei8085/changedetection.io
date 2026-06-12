# changedetection.io 快照磁盘组织与预览版本检索路径分析

## 1. 概述

changedetection.io 将每个监控目标（Watch）的快照数据以 **文件系统目录树** 形式持久化，同时维护一份轻量级的 **历史索引文件** 来记录「时间戳 ↔ 快照文件名」的映射。预览与 Diff 页面则通过 **时间戳** 查阅索引、定位磁盘文件，再由 Processor 插件体系决定渲染方式。

---

## 2. 磁盘目录结构

每个 Watch 拥有一个独立的数据目录，路径模式为：

```
{datastore_path}/{watch_uuid}/
```

其中 `datastore_path` 默认为 `/datastore`（可在启动时覆盖），`watch_uuid` 是 Watch 创建时生成的 UUID v4。

### 2.1 完整目录布局

```
/datastore/
├── changedetection.json          # 全局设置（应用级配置，不含单 Watch 配置）
├── proxies.json                  # 代理配置
├── headers.txt                   # 全局自定义请求头
│
├── {watch-uuid-1}/               # Watch 数据目录
│   ├── watch.json                # 单个 Watch 的完整配置（URL、过滤器、通知等）
│   ├── tag.json                  # 若此目录是 Tag，则为 Tag 配置
│   ├── history.txt               # 文本快照历史索引（默认 processor）
│   ├── history-image_ssim_diff.txt  # 图像处理器专用历史索引
│   ├── {snapshot_id}.txt         # 纯文本快照（未压缩）
│   ├── {snapshot_id}.txt.br      # 纯文本快照（brotli 压缩）
│   ├── {snapshot_id}.jpeg        # 二进制快照（截图，由 image_ssim_diff 产生）
│   ├── {timestamp}.html.br       # 原始 HTML 快照（brotli 压缩，仅保留最近 2 份）
│   ├── last-screenshot.png       # 最近一次正常截图
│   ├── last-error-screenshot.png # 最近一次错误截图
│   ├── last-fetched.br           # 过滤前的原始文本（brotli 压缩）
│   ├── last-checksum.txt         # 上次原始内容校验和（processor 直接读写，用于快速跳过）
│   ├── last-error.txt            # 最近一次错误文本
│   ├── change-summary-{from}-to-{to}-{hash}.txt  # LLM 变更摘要缓存
│   ├── report-{uuid}.csv         # 历史正则抽取报表
│   ├── favicon.ico               # 站点图标
│   ├── elements.deflate          # XPath/视觉选择器数据（zlib 压缩）
│   ├── elements-error.deflate    # XPath/视觉选择器错误数据
│   ├── visual_comparison_data.json  # 图像对比历史数据（processor 直接读）
│   ├── step_before-*.jpeg        # 浏览器步骤截图
│   ├── cropped_image_template.png  # 模板匹配裁剪图（processor 直接读写，可选）
│   ├── text_json_diff.json       # 处理器专属配置（processor 直接读写，可选）
│   └── image_ssim_diff.json      # 处理器专属配置（processor 直接读写，可选）
│
├── {watch-uuid-2}/
│   └── ...
```

### 2.2 容易混淆点之一：设置信息 vs 单个监控项的存储位置

这是最容易混淆的点。全局设置和单个 Watch 配置存储在 **完全不同的文件** 中：

| 类别 | 存储位置 | 写入逻辑 |
|------|----------|----------|
| **全局设置** | `{datastore_path}/changedetection.json` | 由 [FileSavingDataStore._save_settings()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/store/file_saving_datastore.py#L472-L481) → [save_json_atomic()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/store/file_saving_datastore.py#L36-L176) 原子写入 |
| **单个 Watch 配置** | `{datastore_path}/{watch_uuid}/watch.json` | 由 [EntityPersistenceMixin._save_to_disk()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/persistence.py#L52-L84) → [save_watch_atomic()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/store/file_saving_datastore.py#L200-L208) 原子写入 |
| **单个 Tag 配置** | `{datastore_path}/{tag_uuid}/tag.json` | 同样由 [save_entity_atomic()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/store/file_saving_datastore.py#L178-L198) 写入 |

**加载顺序**（[store/__init__.py:169-188](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/store/__init__.py#L169-L188)）：

```
1. 加载 changedetection.json → 获取全局设置（应用配置、密码、令牌等）
2. 扫描 {datastore_path}/*/watch.json → 加载所有 Watch
3. 扫描 {datastore_path}/*/tag.json → 加载所有 Tag
4. 合并 Tag → 覆盖 changedetection.json 中的 legacy tags
```

**关键点：** `changedetection.json` 中 **不包含** 任何 Watch 的配置数据。它只存全局级别的设置（如 `settings.application`、`settings.headers`、`settings.requests`）。所有 Watch/Tag 都分散在各自 UUID 目录下的 `watch.json`/`tag.json` 中。

### 2.3 关键文件说明

| 文件 | 作用 |
|------|------|
| `watch.json` | Watch 运行时配置，由 [EntityPersistenceMixin._save_to_disk()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/persistence.py#L52-L84) 原子写入 |
| `history.txt` | 文本快照索引，格式为 `{timestamp},{filename}\n`，每行一条记录 |
| `history-{processor}.txt` | 非默认处理器（如 `image_ssim_diff`）使用的独立索引文件 |
| `last-screenshot.png` | 始终保存最新截图，**无版本概念**，仅保留一份 |
| `{timestamp}.html.br` | 原始 HTML 缓存，仅保留最近 2 份（[Watch.py:1246-1257](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1246-L1257)） |

---

## 3. 快照写入流程

### 3.1 完整写入链路（各层职责）

写入链路跨越 **Worker 层 → Processor 层 → Watch 层**，各层职责边界不同：

```
Worker 层 (worker.py)
  ├─ 调用 processor.perform_site_check() → 触发抓取
  ├─ 调用 processor.run_changedetection() → 变更检测
  │
  ├─ 检测到 changed_detected=True 后：
  │   ├─ watch.save_screenshot()         # Watch 层：保存截图到 last-screenshot.png
  │   ├─ watch.save_history_blob()       # Watch 层：保存快照文本/二进制
  │   ├─ watch.save_last_fetched_html()  # Watch 层：保存原始 HTML 缓存
  │   └─ （注意：save_last_text_fetched_before_filters 不在此调用！）
```

### 3.2 容易混淆点之二：过滤前文本（last-fetched.br）究竟由哪一层写入

**答案：由 Processor 层写入，不是 Worker 层，也不是 Watch 层主动写入。**

写入发生在 `text_json_diff` 处理器的 `_apply_diff_filtering()` 方法中（[processor.py:662-681](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/text_json_diff/processor.py#L662-L681)）：

```python
def _apply_diff_filtering(self, watch, stripped_text, text_before_filter):
    """Apply user's diff filtering preferences (show only added/removed/replaced lines)."""
    from changedetectionio import diff

    rendered_diff = diff.render_diff(
        previous_version_file_contents=watch.get_last_fetched_text_before_filters(),
        newest_version_file_contents=stripped_text,
        include_equal=False,
        include_added=watch.get('filter_text_added', True),
        include_removed=watch.get('filter_text_removed', True),
        include_replaced=watch.get('filter_text_replaced', True),
        include_change_type_prefix=False
    )

    # 关键：在这里写入 last-fetched.br
    watch.save_last_text_fetched_before_filters(text_before_filter.encode('utf-8'))

    if not rendered_diff and stripped_text:
        return None

    return rendered_diff
```

**调用时机：** 仅当 Watch 设置了 `filter_text_added` / `filter_text_removed` / `filter_text_replaced`（即 `has_special_diff_filter_options_set()` 返回 True），且已有历史快照时，才会在 `run_changedetection()` 流程中调用此方法（[processor.py:532-533](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/text_json_diff/processor.py#L532-L533)）：

```python
# === DIFF FILTERING ===
if watch.has_special_diff_filter_options_set() and len(watch.history.keys()):
    stripped_text = self._apply_diff_filtering(watch, stripped_text, text_content_before_ignored_filter)
```

**写入的内容是什么？** `text_before_filter` 是经过 CSS/XPath/JSON 过滤、文本提取、空白裁剪后，但 **尚未经过 ignore 过滤** 的文本内容。它是下一次 diff 计算的"上一版本"基准。

### 3.3 容易混淆点之三：原始抓取内容 vs 历史快照是不是同一套文件

**答案：完全不是同一套文件。** 抓取后实际上会产生 **四类独立的文件**，各自有不同的用途、保留策略和写入层：

| 类别 | 文件 | 保留策略 | 用途 | 写入层 | 写入方法 |
|------|------|----------|------|--------|----------|
| **历史快照** | `{snapshot_id}.txt.br` / `.txt` / `.jpeg` | 多份，受 `history_snapshot_max_length` 限制 | 版本间 Diff、历史回溯 | Worker 层 | [save_history_blob()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L653-L729) |
| **原始 HTML 缓存** | `{timestamp}.html.br` | 仅 **最近 2 份**，超出自动删除 | API `?html=true` 返回原始页面 | Worker 层 | [save_last_fetched_html()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1227-L1232) |
| **过滤前原始文本** | `last-fetched.br` | 仅 **最新 1 份**，覆盖写 | 调试过滤器、diff 计算基准 | Processor 层 | [save_last_text_fetched_before_filters()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1222-L1225) |
| **最新截图** | `last-screenshot.png` | 仅 **最新 1 份**，覆盖写 | UI "当前截图" 标签页 | Worker 层 | [save_screenshot()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1192-L1204) |

**读取时的回退逻辑：**

[get_last_fetched_text_before_filters()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1207-L1220) 在 `last-fetched.br` 不存在时，会回退读取最新的历史快照：

```python
def get_last_fetched_text_before_filters(self):
    filepath = os.path.join(self.data_dir, 'last-fetched.br')
    if not os.path.isfile(filepath) or os.path.getsize(filepath) == 0:
        # 如果没有过滤前的原始内容，就回退到最新快照
        dates = list(self.history.keys())
        if len(dates):
            return self.get_history_snapshot(timestamp=dates[-1])
        return ''
    # 否则解压返回 last-fetched.br
    with open(filepath, 'rb') as f:
        return brotli.decompress(f.read()).decode('utf-8')
```

**原始 HTML 的保留策略：**

[_prune_last_fetched_html_snapshots()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1246-L1257) 每次保存新 HTML 后，会遍历历史时间戳，**只保留最近 2 份** `{timestamp}.html.br` 文件，其余删除。

### 3.4 `save_history_blob()` 写入逻辑

方法定义于 [Watch.py:653-729](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L653-L729)，按内容类型分流：

```
save_history_blob(contents, timestamp, snapshot_id)
│
├─ contents 是 bytes（二进制）
│   ├─ 用 puremagic 检测文件类型 → 得到扩展名 ext
│   ├─ 文件名 = "{snapshot_id}.{ext}"
│   ├─ 原子写入到 {data_dir}/{snapshot_id}.{ext}
│   └─ 例: "abc123.jpeg"
│
├─ contents 是 str（文本）且 > BROTLI_COMPRESS_SIZE_THRESHOLD（默认 20KB）
│   ├─ brotli 压缩
│   ├─ 文件名 = "{snapshot_id}.txt.br"
│   ├─ 原子写入（失败则回退为 .txt）
│   └─ 例: "abc123.txt.br"
│
└─ contents 是 str（文本）且 <= 阈值
    ├─ 文件名 = "{snapshot_id}.txt"
    ├─ 原子写入 UTF-8
    └─ 例: "abc123.txt"
```

**写入完成后，更新索引文件：**

```python
# 追加一行到 history.txt
index_line = f"{timestamp},{snapshot_fname}\n"
with open(index_fname, 'a', encoding='utf-8') as f:
    f.write(index_line)
```

**历史裁剪：** 若 `history_snapshot_max_length` 被设置（Watch 级 > 全局级），超出上限的旧快照文件会被删除，索引文件被重写。

### 3.5 `history.txt` 索引文件格式

```
1700000000,abc123.txt.br
1700000060,def456.txt.br
1700000120,ghi789.jpeg
```

每行格式为 `{timestamp},{filename}`，`timestamp` 是 Unix 秒级时间戳，`filename` 是快照文件的 **纯文件名**（不含路径）。

### 3.6 截图存储

截图通过 [Watch.save_screenshot()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1192-L1204) 保存：

```python
def save_screenshot(self, screenshot: bytes, as_error=False):
    if as_error:
        target_path = os.path.join(self.data_dir, "last-error-screenshot.png")
    else:
        target_path = os.path.join(self.data_dir, "last-screenshot.png")
    with open(target_path, 'wb') as f:
        f.write(screenshot)
```

**关键点：** `last-screenshot.png` 始终是最新截图的覆盖写，**没有版本历史**。它仅用于 UI 中"当前截图"标签页展示，不参与版本间 diff。

---

## 4. 索引加载：`history` 属性

[Watch.history](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L441-L491) 是一个 `@property`，每次访问时从磁盘重新读取 `history.txt`：

```python
@property
def history(self):
    tmp_history = {}
    fname = os.path.join(self.data_dir, self.history_index_filename)
    if os.path.isfile(fname):
        with open(fname, "r", encoding='utf-8') as f:
            for i in f.readlines():
                if ',' in i:
                    k, v = i.strip().split(',', 2)
                    # 安全校验：确保路径在 data_dir 内
                    snapshot_fname = os.path.basename(v.strip())
                    resolved_path = os.path.realpath(os.path.join(self.data_dir, snapshot_fname))
                    if not resolved_path.startswith(safe_data_dir + os.sep):
                        continue
                    if not os.path.exists(resolved_path):
                        continue
                    tmp_history[k] = resolved_path
    return tmp_history
```

返回值为 `dict`，结构：`{timestamp_str: absolute_file_path}`。

**索引文件名与处理器绑定：**

```python
@property
def history_index_filename(self):
    processor = self.get('processor')
    if not processor or self.get('processor') == 'text_json_diff':
        return 'history.txt'
    else:
        return f'history-{processor}.txt'
```

即同一个 Watch 可以有多份历史索引，每个处理器一份。切换处理器时，preview/diff 页面自动查阅对应的索引文件。

---

## 5. 快照读取：`get_history_snapshot()`

[Watch.get_history_snapshot()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L553-L605) 是按版本检索快照的核心方法：

```
get_history_snapshot(timestamp=None, filepath=None)
│
├─ 未提供 filepath → 从 history dict 中按 timestamp 查找文件路径
│   filepath = self.history[timestamp]
│
├─ 安全校验：确保 filepath 在 data_dir 内
│
├─ 判断文件类型
│   ├─ 二进制扩展名 (.png/.jpg/.jpeg/.gif/.webp/.pdf/.bin/.jfif)
│   │   → 直接读取 bytes 返回
│   │
│   └─ 文本文件
│       ├─ 优先查找 .br 压缩版本 → brotli 解压后返回 str
│       └─ 回退到未压缩版本 → 读取返回 str
│
└─ 返回: str（文本快照）或 bytes（二进制快照）
```

**版本定位的线索就是 `timestamp`**：调用方通过 `timestamp` 参数传入 Unix 时间戳，`get_history_snapshot()` 查阅 `self.history` 字典得到文件绝对路径，再从磁盘读取。

---

## 6. 预览资源检索路径

### 6.1 容易混淆点之四：图片预览资源默认取哪个版本

**答案：默认取最新版本（`versions[-1]`）。**

在 `image_ssim_diff` 处理器的两个入口中，版本选择逻辑完全一致：

**① Preview 页面 `render()` 方法**（[preview.py:94-98](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L94-L98)）：
```python
preferred_version = request.args.get('version')
timestamp = versions[-1]  # 默认最新
if preferred_version and preferred_version in versions:
    timestamp = preferred_version
```

**② Preview Asset `get_asset()` 方法**（[preview.py:38-42](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L38-L42)）：
```python
preferred_version = request.args.get('version')
timestamp = versions[-1]  # 默认最新
if preferred_version and preferred_version in versions:
    timestamp = preferred_version
```

**③ Diff Asset `get_asset()` 方法**（[difference.py:54-61](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L54-L61)）：
```python
from_version = request.args.get('from_version', versions[-2] if len(versions) >= 2 else versions[0])
to_version = request.args.get('to_version', versions[-1])  # to_version 默认最新

# 无效版本自动回退到默认
if from_version not in versions:
    from_version = versions[-2] if len(versions) >= 2 else versions[0]
if to_version not in versions:
    to_version = versions[-1]
```

**④ Diff 页面 `render()` 方法**（[difference.py:332-333](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L332-L333)）：
```python
from_version = request.args.get('from_version', versions[-2] if len(versions) >= 2 else versions[0])
to_version = request.args.get('to_version', versions[-1])
```

### 6.2 容易混淆点之五：diff 接口还支持哪些版本关键字

所有版本关键字汇总如下：

| 关键字 | 适用入口 | 代码位置 | 含义 |
|--------|----------|----------|------|
| `'latest'` | API 单快照<br>API Diff | [Watch.py:272-273](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L272-L273)<br>[Watch.py:346-348](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L346-L348) | 最新快照<br>`to_timestamp='latest'` → 最新 |
| `'previous'` | API Diff | [Watch.py:351-354](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L351-L354) | `from_timestamp='previous'` → 倒数第二新 |
| `'first'` | Preview/Diff 路由 | [preview.py:31-32](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/preview.py#L31-L32)<br>[diff.py:114-115](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/diff.py#L114-L115) | UUID 快捷方式，跳转到第一个 Watch |

**API Diff 关键字处理**（[Watch.py:346-354](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L346-L354)）：
```python
# Handle 'latest' keyword for to_timestamp
if to_timestamp == 'latest':
    to_timestamp = history_keys[-1]

# Handle 'previous' keyword for from_timestamp (second-most-recent)
if from_timestamp == 'previous':
    if len(history_keys) < 2:
        abort(404, message=f"Not enough history entries. Need at least 2 snapshots for 'previous'")
    from_timestamp = history_keys[-2]
```

### 6.3 预览页面路由

**URL:** `/preview/<uuid>`

定义于 [preview.py:14-125](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/preview.py#L14-L125)。

```
用户访问 /preview/{uuid}
│
├─ 获取 watch 的 processor 类型（默认 text_json_diff）
│
├─ 尝试加载 processor 的 preview 子模块
│   processors/{processor_name}/preview.py::render()
│
├─ 若存在 → 委托给 processor 的 render() 渲染
│
└─ 若不存在 → 使用默认文本预览逻辑
    ├─ 从 URL 查询参数获取 version（时间戳）
    ├─ versions = list(watch.history.keys())  # 所有时间戳
    ├─ timestamp = versions[-1]               # 默认最新版本
    ├─ 如果 ?version=xxx 存在且有效 → 使用指定版本
    ├─ content = watch.get_history_snapshot(timestamp=timestamp)
    └─ 渲染 preview.html 模板
```

### 6.4 Diff 页面路由

**URL:** `/diff/<uuid>`

定义于 [diff.py:97-157](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/diff.py#L97-L157)。

```
用户访问 /diff/{uuid}
│
├─ 获取 watch 的 processor 类型
│
├─ 尝试加载 processor 的 difference 子模块
│   processors/{processor_name}/difference.py::render()
│
├─ 若存在 → 委托给 processor 的 render() 渲染
│
└─ 若不存在 → 回退到 text_json_diff 的 difference.py
    ├─ 从 URL 查询参数获取 from_version / to_version
    ├─ 默认：from = 上次查看位置对应版本, to = 最新版本
    ├─ from_text = watch.get_history_snapshot(timestamp=from_version)
    ├─ to_text   = watch.get_history_snapshot(timestamp=to_version)
    └─ 执行文本 diff → 渲染 diff.html 模板
```

### 6.5 不同入口取版本时的默认规则和关键字分支汇总

不同入口取版本时，**默认规则和关键字分支差异很大**，这是最容易踩坑的地方。以下是各入口的详细规则对比：

| 入口 | 代码位置 | 版本参数 | 默认值规则 | 特殊关键字 |
|------|----------|----------|------------|------------|
| **Preview 页面（文本）** | [preview.py:74-80](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/preview.py#L74-L80) | `version` | `versions[-1]`（最新） | 无 |
| **Preview 页面（图片）** | [image_ssim_diff/preview.py:94-98](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L94-L98) | `version` | `versions[-1]`（最新） | 无 |
| **Preview Asset（图片）** | [image_ssim_diff/preview.py:38-42](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L38-L42) | `version` | `versions[-1]`（最新） | 无 |
| **Diff 页面 (text)** | [text_json_diff/difference.py:138-143](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/text_json_diff/difference.py#L138-L143) | `from_version`, `to_version` | `from` = `get_from_version_based_on_last_viewed`<br>`to` = `dates[-1]`（最新） | 无 |
| **Diff 页面 (image)** | [image_ssim_diff/difference.py:332-333](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L332-L333) | `from_version`, `to_version` | `from` = `versions[-2]`<br>`to` = `versions[-1]` | 无 |
| **Diff Asset (image)** | [image_ssim_diff/difference.py:54-61](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L54-L61) | `from_version`, `to_version` | `from` = `versions[-2]`<br>`to` = `versions[-1]` | 无效版本自动回退 |
| **API 单快照** | [api/Watch.py:272-273](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L272-L273) | `timestamp` | 必须显式指定 | `'latest'` → 最新 |
| **API Diff** | [api/Watch.py:346-354](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L346-L354) | `from_timestamp`, `to_timestamp` | 必须显式指定 | `to='latest'` → 最新<br>`from='previous'` → 倒数第二新 |
| **UUID 快捷入口** | [preview.py:31-32](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/preview.py#L31-L32) | `uuid` | - | `'first'` → 第一个 Watch |

#### 6.5.1 Diff 页面的 `last_viewed` 智能计算（最关键的隐藏规则）

这是最隐蔽、最容易混淆的逻辑。当用户首次进入 Diff 页面且未指定 `from_version` 时，系统不会简单地取 `versions[-2]`，而是通过 [get_from_version_based_on_last_viewed](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L527-L551) 进行智能计算：

```python
@property
def get_from_version_based_on_last_viewed(self):
    keys = list(self.history.keys())
    if not keys:
        return None
    if len(keys) == 1:
        return keys[0]

    last_viewed = int(self.get('last_viewed'))
    sorted_keys = sorted(keys, key=lambda x: int(x))
    sorted_keys.reverse()  # 从新到旧排序

    # 规则1: 上次查看时间 >= 最新快照时间 → 返回倒数第二新
    if last_viewed >= int(sorted_keys[0]):
        return sorted_keys[1]
    
    # 规则2: 上次查看时间在两个快照之间 → 返回较旧的那个
    for newer, older in list(zip(sorted_keys[0:], sorted_keys[1:])):
        if last_viewed < int(newer) and last_viewed >= int(older):
            return older

    # 规则3: 上次查看时间 < 最早快照时间 → 返回最早的
    return sorted_keys[-1]
```

**`last_viewed` 字段的更新时机：**
- 当用户打开 Diff 页面时，[datastore.set_last_viewed()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/text_json_diff/difference.py#L165) 被调用，写入当前时间戳到 `watch['last_viewed']`
- `last_viewed` 保存在 `watch.json` 中，重启后仍然有效

**一个典型场景：**
```
1. 00:00 用户打开 Diff 页面 → set_last_viewed(00:00)
2. 00:05 检测到变更，生成快照1
3. 00:10 检测到变更，生成快照2
4. 00:15 用户再次打开 Diff 页面（未指定 from_version）
   → last_viewed=00:00，介于 快照0(00:00前) 和 快照1(00:05) 之间
   → from_version = 快照0（上次查看时的版本）
   → to_version   = 快照2（最新）
   → 用户看到：快照0 → 快照2 的完整差异
```

### 6.6 Processor Asset 路由（二进制资源流式返回）

**URL:** `/preview/<uuid>/processor-asset/<asset_name>` 或 `/diff/<uuid>/processor-asset/<asset_name>`

定义于 [preview.py:127-187](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/preview.py#L127-L187) 和 [diff.py:477-539](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/diff.py#L477-L539)。

此路由解决了一个重要问题：**大型二进制资源（如截图）不应以 base64 嵌入 HTML 模板**，而是作为独立 HTTP 响应流式返回。

```
/diff/{uuid}/processor-asset/before?from_version=xxx&to_version=yyy
/diff/{uuid}/processor-asset/after?from_version=xxx&to_version=yyy
/diff/{uuid}/processor-asset/rendered_diff?from_version=xxx&to_version=yyy
/preview/{uuid}/processor-asset/screenshot?version=xxx
│
├─ 获取 processor 的 difference/preview 子模块
├─ 调用 processor.get_asset(asset_name, watch, datastore, request)
│
├─ image_ssim_diff/difference.py::get_asset() 逻辑:
│   ├─ 'before'  → watch.get_history_snapshot(timestamp=from_version) → 返回 bytes
│   ├─ 'after'   → watch.get_history_snapshot(timestamp=to_version) → 返回 bytes
│   └─ 'rendered_diff' → 分别读取 before/after → 生成差异可视化图
│
├─ image_ssim_diff/preview.py::get_asset() 逻辑:
│   └─ 'screenshot' → watch.get_history_snapshot(timestamp=version) → 返回 bytes
│
└─ 返回 (binary_data, content_type, cache_control) → Flask Response
```

### 6.7 截图静态路由

**URL:** `/static/screenshot/{uuid}`

定义于 [flask_app.py:740-772](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/flask_app.py#L740-L772)。

```
/static/screenshot/{uuid}
│
├─ 权限校验（如果设置了密码且未登录 → 403）
├─ 文件名固定为 "last-screenshot.png" 或 "last-error-screenshot.png"
├─ 从 {datastore_path}/{uuid}/ 目录下读取
└─ 返回 image/png（无缓存）
```

**注意：** 此路由返回的截图 **没有版本概念**，始终是 `last-screenshot.png`，仅展示最新状态。

---

## 7. Watch 层 vs Processor 层的存储职责边界（修正版）

> **⚠️ 修正说明**：之前的结论"Processor 层不直接读写磁盘文件"不准确。顺着代码追踪发现，Processor 层（尤其是基类）有多处直接 `open()` / `os.remove()` 的磁盘操作。本节重新梳理准确的职责边界。

### 7.1 两层职责总览

| 层面 | 核心职责 | 是否直接读写磁盘 | 典型方法/属性 |
|------|----------|------------------|---------------|
| **Watch 层**<br>`model/Watch.py` | **核心数据存储与检索**<br>配置持久化<br>历史快照读写<br>索引管理<br>属性计算<br>安全校验 | 是（主存储层，所有核心数据） | `save_history_blob()`<br>`get_history_snapshot()`<br>`save_last_fetched_html()`<br>`save_screenshot()`<br>`save_last_text_fetched_before_filters()`<br>`history` 属性<br>`get_from_version_based_on_last_viewed`<br>`newest_history_key` |
| **Processor 层**<br>`processors/*/` | **内容处理与渲染**<br>内容抓取<br>过滤/提取/转换<br>变更检测（checksum）<br>UI 渲染<br>资源提供 | 是（辅助存储层，仅处理器专属优化文件） | `perform_site_check()`<br>`run_changedetection()`<br>`update_last_raw_content_checksum()`<br>`read_extra_watch_config()`<br>`update_extra_watch_config()`<br>`render()`<br>`get_asset()` |

### 7.2 Watch 层统一持久化的核心数据

以下数据**始终由 Watch 层负责写入**，Processor 层即使要使用，也通过调用 Watch 层方法获取或写入：

| 数据类别 | 具体文件 | 写入方法 | 说明 |
|----------|----------|----------|------|
| Watch 配置 | `watch.json` | `_save_to_disk()` | 由 `EntityPersistenceMixin` 统一管理 |
| 历史快照索引 | `history.txt` / `history-{processor}.txt` | `save_history_blob()` 内部追加 | 每个处理器有独立索引 |
| 历史快照文件 | `{snapshot_id}.txt.br` / `.txt` / `.jpeg` | `save_history_blob()` | 带版本、有索引、受裁剪限制 |
| 原始 HTML 缓存 | `{timestamp}.html.br` | `save_last_fetched_html()` | 仅保留最近 2 份 |
| 过滤前原始文本 | `last-fetched.br` | `save_last_text_fetched_before_filters()` | 由 Processor 层**调用时机**，但写入方法是 Watch 层的 |
| 最新截图 | `last-screenshot.png` / `last-error-screenshot.png` | `save_screenshot()` | 无版本，覆盖写 |
| XPath/视觉选择器数据 | `elements.deflate` / `elements-error.deflate` | `save_xpath_data()` | zlib 压缩存储 |
| LLM 变更摘要缓存 | `change-summary-{from}-to-{to}-{hash}.txt` | `save_llm_diff_summary()` | 按版本对缓存 |
| 历史正则抽取报表 | `report-{uuid}.csv` | 由 `store.py` 写入 | 报表导出功能 |
| 错误文本 | `last-error.txt` | 由 `store.py` / `worker.py` 写入 | 最近一次错误信息 |
| 站点图标 | `favicon.ico` | 由 `worker.py` 写入 | 网站 favicon |

### 7.3 Processor 层直接读写磁盘的完整例外清单

以下是 Processor 层**直接 open()/os.remove() 磁盘文件**的所有情况，按重要性排序：

#### ① `last-checksum.txt` — checksum 快速跳过机制（基类）

**代码位置**：[processors/base.py:46-98](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/base.py#L46-L98)

**操作**：读 + 写

```python
# base.py __init__ 时即读取
def read_last_raw_content_checksum(self):
    checksum_file = os.path.join(self.datastore.data_path, self.watch['uuid'], 'last-checksum.txt')
    with open(checksum_file, 'r', encoding='utf-8') as f:
        return f.read().strip()

# 处理完成后写入
def update_last_raw_content_checksum(self, checksum):
    checksum_file = os.path.join(self.datastore.data_path, self.watch['uuid'], 'last-checksum.txt')
    with open(checksum_file, 'w', encoding='utf-8') as f:
        f.write(checksum)
```

**作用**：保存原始 HTML 的 MD5 校验和。下次抓取如果内容相同，直接跳过过滤/提取步骤，大幅提升性能。

---

#### ② `{processor_name}.json` — 处理器专属配置（基类）

**代码位置**：[processors/base.py:275-350](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/base.py#L275-L350)

**操作**：读 + 写（支持 merge）

```python
def read_extra_watch_config(self, filename):
    """Read processor-specific config JSON from watch data dir."""
    config_file = os.path.join(self.datastore.data_path, self.watch['uuid'], filename)
    with open(config_file, 'r', encoding='utf-8') as f:
        return json.load(f)

def update_extra_watch_config(self, filename, config, merge=False):
    """Write processor-specific config JSON to watch data dir."""
    config_file = os.path.join(self.datastore.data_path, self.watch['uuid'], filename)
    with open(config_file, 'w', encoding='utf-8') as f:
        json.dump(merged_config, f, indent=2)
```

**实际使用的文件**：
- `text_json_diff.json` — 文本处理器专属配置（如 ignore text、CSS/JSON 选择器等）
- `image_ssim_diff.json` — 图像处理器专属配置（如对比区域、阈值等）

---

#### ③ `visual_comparison_data.json` — 图像对比历史数据（image_ssim_diff）

**代码位置**：[processors/image_ssim_diff/difference.py:417-423](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L417-L423)

**操作**：只读

**作用**：Diff 页面的"变化趋势图"数据，记录每次检测的相似度分数历史。

---

#### ④ `cropped_image_template.png` — 模板匹配裁剪图（image_ssim_diff）

**代码位置**：[processors/image_ssim_diff/edit_hook.py:85-96](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/edit_hook.py#L85-L96)

**操作**：读 + 写 + 删除（`os.remove()`）

**作用**：布局变化时追踪内容的模板匹配图（需 `ENABLE_TEMPLATE_TRACKING=True`）。

---

#### ⑤ 直接读取 `{timestamp}.html.br` — 过滤器预览（text_json_diff）

**代码位置**：[processors/text_json_diff/__init__.py:78-80](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/text_json_diff/__init__.py#L78-L80)

**操作**：只读

**作用**：过滤器预览功能（`prepare_filter_preview()`）直接读取原始 HTML 缓存文件，用于实时测试过滤器效果。

---

#### ⑥ 直接读取 `elements.deflate` — XPath 坐标数据（image_ssim_diff）

**代码位置**：[processors/image_ssim_diff/difference.py:227-232](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L227-L232)

**操作**：只读（`gzip.open()`）

**作用**：Diff 页面绘制裁剪区域时，直接读取 XPath 元素坐标数据。

### 7.4 修正版调用关系图

```
Worker 层 (worker.py)
  ↓ 调用
Processor 层 (processors/text_json_diff/processor.py)
  ├─ perform_site_check()  → 抓取内容
  ├─ run_changedetection() → 过滤、提取、计算 checksum、检测变更
  │   ├─ ✅ 直接读 last-checksum.txt（基类，快速跳过）
  │   ├─ ✅ 直接写 last-checksum.txt（基类，更新校验和）
  │   ├─ ✅ 直接读/写 {processor_name}.json（基类，专属配置）
  │   └─ 调用 watch.save_last_text_fetched_before_filters() → 写入 last-fetched.br
  │      （写入方法是 Watch 层的，但调用时机由 Processor 控制）
  └─ 返回 (changed_detected, update_obj, contents)
  ↓ 返回结果
Worker 层
  ├─ watch.save_screenshot() → Watch 层：存截图
  ├─ watch.save_history_blob() → Watch 层：存快照 + 更新索引
  └─ watch.save_last_fetched_html() → Watch 层：存原始 HTML

UI 层 (blueprint/ui/preview.py, diff.py)
  ↓ 委托
Processor 层 (processors/*/preview.py, difference.py)
  ├─ render() → 渲染 HTML
  │   ├─ ✅ 直接读 visual_comparison_data.json（图像对比趋势）
  │   ├─ ✅ 直接读 elements.deflate（XPath 坐标）
  │   └─ ✅ 直接读 {timestamp}.html.br（过滤器预览）
  └─ get_asset() → 提供二进制资源
      └─ watch.get_history_snapshot() → Watch 层：读快照
```

> ✅ = Processor 层直接 I/O 的节点

### 7.5 修正后的职责边界四原则

1. **Watch 层是"主存储层"**：所有核心业务数据（配置、快照、索引、截图、HTML 缓存、过滤前文本、XPath 数据、LLM 摘要）都通过 Watch 层封装方法读写，保证路径校验、原子写入等安全性。

2. **Processor 层是"辅助存储层"**：直接读写的文件都局限于**处理器专属的优化数据和配置**——`last-checksum.txt`（性能优化）、`{processor_name}.json`（独立配置）、以及图像处理器专用的辅助文件（趋势数据、模板图）。

3. **Processor 层预览渲染时直接读 Watch 层文件**：为了避免额外的封装开销，Processor 在渲染预览、生成差异图时，会直接读取 Watch 层已写入的文件（`.html.br`、`elements.deflate`），但**不会修改这些文件**。

4. **Watch 层不处理业务逻辑**：过滤器、diff 计算、checksum 跳过判断、UI 渲染等业务逻辑始终由 Processor 层完成。数据存取和业务处理的分层仍然清晰，只是 Processor 层有少量直接 I/O 的性能优化例外。

---

## 8. 版本检索核心线索总结

整条版本检索链路的核心线索是 **Unix 时间戳**（`timestamp`），它同时充当：

1. **history.txt 索引的 key** — 格式为 `{timestamp},{filename}`
2. **URL 查询参数** — `?version=1700000120`、`?from_version=xxx&to_version=yyy`
3. **`watch.history` dict 的 key** — `{timestamp: absolute_filepath}`
4. **快照文件名的一部分** — 对于 HTML 缓存是 `{timestamp}.html.br`

完整的版本检索流程：

```
URL 查询参数 ?version=1700000120
        │
        ▼
  路由处理函数 (preview.py / diff.py)
        │
        ▼
  versions = list(watch.history.keys())
        │  ← 读取 history.txt，构建 {timestamp: filepath} 字典
        ▼
  验证 version 在 versions 中
        │
        ▼
  watch.get_history_snapshot(timestamp="1700000120")
        │
        ▼
  filepath = self.history["1700000120"]  → 如 /datastore/{uuid}/abc123.txt.br
        │
        ▼
  根据扩展名选择读取方式:
    .br → brotli 解压
    .txt → 直接读取
    .jpeg/.png → 返回 bytes
        │
        ▼
  返回快照内容 (str 或 bytes)
```

---

## 9. 五个容易混淆点的澄清总结

| 问题 | 答案 | 关键代码 |
|------|------|----------|
| **设置信息和单个监控项分别落到哪里？** | 全局设置 → `changedetection.json`<br>Watch 配置 → `{uuid}/watch.json`<br>Tag 配置 → `{uuid}/tag.json`<br>三者完全独立 | [file_saving_datastore.py:200-208](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/store/file_saving_datastore.py#L200-L208) |
| **过滤前文本（last-fetched.br）由哪一层写入？** | **Processor 层**，在 `text_json_diff/processor.py` 的 `_apply_diff_filtering()` 中写入<br>仅当设置了 diff 过滤选项且已有历史时触发 | [processor.py:676](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/text_json_diff/processor.py#L676) |
| **原始抓取内容与历史快照是不是同一套文件？** | 不是。有四类独立文件：<br>① 历史快照（多份）<br>② 原始 HTML（仅最近 2 份）<br>③ 过滤前原始文本（仅最新 1 份）<br>④ 最新截图（仅 1 份） | [Watch.py:1207-1225](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1207-L1225) |
| **图片预览资源默认取哪个版本？** | **最新版本（`versions[-1]`）**<br>`image_ssim_diff` 的 `render()` 和 `get_asset()` 都用此默认 | [preview.py:40](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L40)<br>[preview.py:96](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L96) |
| **diff 接口支持哪些版本关键字？** | `'latest'` → 最新快照<br>`'previous'` → 倒数第二新<br>`'first'` → 第一个 Watch（UUID 快捷方式） | [Watch.py:272-273](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L272-L273)<br>[Watch.py:351-354](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L351-L354) |

---

## 10. Processor 插件体系对快照的影响

不同处理器类型会产生不同的快照格式和索引：

| 处理器 | 索引文件 | 快照格式 | 预览渲染 |
|--------|----------|----------|----------|
| `text_json_diff` | `history.txt` | `{snapshot_id}.txt.br` / `.txt` | 文本高亮 |
| `image_ssim_diff` | `history-image_ssim_diff.txt` | `{snapshot_id}.jpeg` / `.png` | 图片对比（滑动条） |

切换处理器时，`history_index_filename` 属性自动返回对应索引文件名，preview/diff 页面自动查阅对应索引。

---

## 11. 关键安全机制

1. **路径穿越防护（Watch 层）**：`history` 属性读取索引时，使用 `os.path.basename()` 剥离路径，`os.path.realpath()` 解析符号链接，确保快照文件在 `data_dir` 内。
2. **原子写入（Watch 层）**：所有快照写入使用 `_write_atomic()`（临时文件 + `os.replace()`），避免半写状态。
3. **API 快照返回**：即使请求 `?html=true`，也以 `text/plain; charset=utf-8` 返回（防止 XSS），并设置 `X-Content-Type-Options: nosniff`。
4. **Processor 层直接读写的安全注意**：Processor 层直接读写磁盘时（如 `last-checksum.txt`、`{processor_name}.json`），**未做 Watch 层那样严格的路径穿越校验**，但：
   - 文件名由代码硬编码（如 `'last-checksum.txt'`），不接受用户输入作为文件名
   - 目录路径通过 `watch.data_dir` 获取，由 Watch 层保证在合法范围内
   - `update_extra_watch_config()` 的 `filename` 参数由上层调用方（UI 保存逻辑、`save_processor_config()`）传入固定值
