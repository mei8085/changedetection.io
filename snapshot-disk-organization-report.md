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
│   ├── last-checksum.txt         # 上次校验和（用于快速比对跳过）
│   ├── favicon.ico               # 站点图标
│   ├── elements.deflate          # XPath/视觉选择器数据（zlib 压缩）
│   ├── step_before-*.jpeg        # 浏览器步骤截图
│   ├── text_json_diff.json       # 处理器专属配置（可选）
│   └── image_ssim_diff.json      # 处理器专属配置（可选）
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

### 3.1 入口：Worker 检测到变更

完整写入链路从 [worker.py](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/worker.py#L526-L528) 开始：

```
Worker 检测到变更
  → watch.save_screenshot()         # 保存截图到 last-screenshot.png
  → watch.save_history_blob()       # 保存快照文本/二进制
  → watch.save_last_fetched_html()  # 保存原始 HTML 缓存
  → watch.save_last_text_fetched_before_filters()  # 保存过滤前的原始文本
```

### 3.2 容易混淆点之二：原始抓取内容 vs 历史快照是不是同一套文件

**答案：完全不是同一套文件。** 抓取后实际上会产生 **三类独立的文件**，各自有不同的用途和保留策略：

| 类别 | 文件 | 保留策略 | 用途 | 写入方法 |
|------|------|----------|------|----------|
| **历史快照** | `{snapshot_id}.txt.br` / `.txt` / `.jpeg` | 保留多份，受 `history_snapshot_max_length` 限制 | 用于版本间 Diff、历史回溯 | [save_history_blob()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L653-L729) |
| **原始 HTML 缓存** | `{timestamp}.html.br` | 仅保留最近 **2 份**，超出自动删除 | 用于 API `?html=true` 请求返回原始页面 | [save_last_fetched_html()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1227-L1232) |
| **过滤前原始文本** | `last-fetched.br` | 仅保留 **最新 1 份**，覆盖写 | 用于调试过滤器、查看"过滤前"内容 | [save_last_text_fetched_before_filters()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1222-L1225) |
| **最新截图** | `last-screenshot.png` | 仅保留 **最新 1 份**，覆盖写 | 用于 UI "当前截图"标签页 | [save_screenshot()](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1192-L1204) |

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

### 3.3 `save_history_blob()` 写入逻辑

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

### 3.4 `history.txt` 索引文件格式

```
1700000000,abc123.txt.br
1700000060,def456.txt.br
1700000120,ghi789.jpeg
```

每行格式为 `{timestamp},{filename}`，`timestamp` 是 Unix 秒级时间戳，`filename` 是快照文件的 **纯文件名**（不含路径）。

### 3.5 截图存储

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

### 6.1 预览页面路由

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

### 6.2 Diff 页面路由

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

### 6.3 容易混淆点之三：不同入口取版本时的默认规则和关键字分支

不同入口取版本时，**默认规则和关键字分支差异很大**，这是最容易踩坑的地方。以下是各入口的详细规则对比：

| 入口 | 代码位置 | 版本参数 | 默认值规则 | 特殊关键字 |
|------|----------|----------|------------|------------|
| **Preview 页面** | [preview.py:74-80](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/blueprint/ui/preview.py#L74-L80) | `version` | `versions[-1]`（最新） | 无特殊关键字 |
| **Diff 页面 (text_json_diff)** | [difference.py:138-143](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/text_json_diff/difference.py#L138-L143) | `from_version`, `to_version` | `from` = `get_from_version_based_on_last_viewed`<br>`to` = `dates[-1]`（最新） | 无特殊关键字 |
| **Diff Asset (image_ssim_diff)** | [difference.py:54-61](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L54-L61) | `from_version`, `to_version` | `from` = `versions[-2]`<br>`to` = `versions[-1]` | 无效版本自动回退到默认 |
| **API 单快照** | [Watch.py:272-273](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L272-L273) | `timestamp` | 必须显式指定 | `timestamp='latest'` → 映射到最新 |
| **Preview Asset (image)** | [preview.py:?] | `version` | 需显式指定 | 无效版本返回 None |

#### 6.3.1 Diff 页面的 `last_viewed` 智能计算（最关键的隐藏规则）

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

#### 6.3.2 API 入口的 `latest` 关键字

API 接口 `/api/v1/watch/<uuid>/history/<timestamp>` 支持特殊关键字 `timestamp='latest'`，在 [Watch.py:272-273](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/api/Watch.py#L272-L273) 中处理：

```python
if timestamp == 'latest':
    timestamp = list(watch.history.keys())[-1]
```

#### 6.3.3 Diff Asset 路由的版本回退

在 [image_ssim_diff/difference.py:54-61](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L54-L61) 中，如果请求的 `from_version` 或 `to_version` 不存在，会自动回退到默认值：

```python
from_version = request.args.get('from_version', versions[-2] if len(versions) >= 2 else versions[0])
to_version = request.args.get('to_version', versions[-1])

if from_version not in versions:
    from_version = versions[-2] if len(versions) >= 2 else versions[0]
if to_version not in versions:
    to_version = versions[-1]
```

#### 6.3.4 `uuid='first'` 快捷入口

Preview 和 Diff 路由都支持 `uuid='first'` 作为快捷方式，自动跳转到第一个 Watch：

```python
if uuid == 'first':
    uuid = list(datastore.data['watching'].keys()).pop()
```

### 6.4 Processor Asset 路由（二进制资源流式返回）

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

### 6.5 截图静态路由

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

## 7. 版本检索核心线索总结

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

## 8. 三个容易混淆点的澄清总结

| 问题 | 答案 | 关键代码 |
|------|------|----------|
| **设置信息和单个监控项分别落到哪里？** | 全局设置 → `changedetection.json`<br>Watch 配置 → `{uuid}/watch.json`<br>Tag 配置 → `{uuid}/tag.json`<br>三者是完全独立的文件 | [file_saving_datastore.py:200-208](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/store/file_saving_datastore.py#L200-L208) |
| **原始抓取内容与历史快照是不是同一套文件？** | 不是。有三类独立文件：<br>① 历史快照（多份）<br>② 原始 HTML（仅最近 2 份）<br>③ 过滤前原始文本（仅最新 1 份） | [Watch.py:1207-1225](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L1207-L1225) |
| **不同入口取版本时有什么默认规则？** | Diff 页面有 `last_viewed` 智能计算<br>API 支持 `'latest'` 关键字<br>Diff Asset 有版本回退逻辑<br>Preview 默认取最新 | [Watch.py:527-551](file:///d:/fz/0601-1/solo-dogfeeding/code/35-changedetection.io/changedetectionio/model/Watch.py#L527-L551) |

---

## 9. Processor 插件体系对快照的影响

不同处理器类型会产生不同的快照格式和索引：

| 处理器 | 索引文件 | 快照格式 | 预览渲染 |
|--------|----------|----------|----------|
| `text_json_diff` | `history.txt` | `{snapshot_id}.txt.br` / `.txt` | 文本高亮 |
| `image_ssim_diff` | `history-image_ssim_diff.txt` | `{snapshot_id}.jpeg` / `.png` | 图片对比（滑动条） |

切换处理器时，`history_index_filename` 属性自动返回对应索引文件名，preview/diff 页面自动查阅对应索引。

---

## 10. 关键安全机制

1. **路径穿越防护**：`history` 属性读取索引时，使用 `os.path.basename()` 剥离路径，`os.path.realpath()` 解析符号链接，确保快照文件在 `data_dir` 内。
2. **原子写入**：所有快照写入使用 `_write_atomic()`（临时文件 + `os.replace()`），避免半写状态。
3. **API 快照返回**：即使请求 `?html=true`，也以 `text/plain; charset=utf-8` 返回（防止 XSS），并设置 `X-Content-Type-Options: nosniff`。
