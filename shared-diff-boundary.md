# shared_diff_access 安全边界代码分析

本文沿着 changedetection.io 中 `shared_diff_access` 的放行链路，从四个维度深入分析其安全边界设计。

---

## 一、白名单上三个 diff 端点对匿名读者 UUID 归属的校验

### 1.1 端点清单与路由层校验

三个白名单端点均通过 `<uuid_str:uuid>` URL 转换器接收 UUID 参数，定义于 [diff.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/diff.py)：

| 端点 | 路由 | 函数 |
|------|------|------|
| diff_history_page | `/diff/<uuid_str:uuid>` | 第 97-157 行 |
| download_patch | `/diff/<uuid_str:uuid>/download-patch` | 第 436-475 行 |
| processor_asset | `/diff/<uuid_str:uuid>/processor-asset/<string:asset_name>` | 第 477-540 行 |

### 1.2 StrictUUIDConverter：路由级格式强校验

UUID 参数首先经过自定义的 [StrictUUIDConverter](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L75-L95) 严格校验：

```python
class StrictUUIDConverter(BaseConverter):
    _ALLOWED_SENTINELS = frozenset({'first'})

    def to_python(self, value: str) -> str:
        if value in self._ALLOWED_SENTINELS:
            return value
        try:
            u = UUID(value)
        except ValueError as e:
            raise ValidationError() from e
        # Reject non-standard formats (braces, URNs, no-hyphens)
        if str(u) != value.lower():
            raise ValidationError()
        return str(u)
```

**校验要点：**
- 仅允许标准 UUID 字符串格式（带连字符、无花括号、非 URN 格式），否则直接 404，请求不会进入视图函数
- 特殊哨兵值 `first` 被放行，后续在视图函数中转换为第一个监控项的实际 UUID
- 转换后统一返回 `str(u)` 的小写标准化形式，避免大小写绕过

### 1.3 各端点内部的 UUID 归属校验

#### diff_history_page（第 114-126 行）

```python
if uuid == 'first':
    uuid = list(datastore.data['watching'].keys()).pop()

try:
    watch = datastore.data['watching'][uuid]
except KeyError:
    flash(gettext("No history found for the specified link, bad link?"), "error")
    return redirect(url_for('watchlist.index'))

dates = list(watch.history.keys())
if not dates or len(dates) < 2:
    flash(gettext("Not enough history (2 snapshots required) to show difference page for this watch."), "error")
    return redirect(url_for('watchlist.index'))
```

**校验链路：**
1. `first` → 取 datastore 中第一个 watch 的 UUID
2. `datastore.data['watching'][uuid]` → KeyError 兜底（UUID 不存在则重定向到监控列表）
3. `watch.history` 长度校验 → 少于 2 条快照拒绝访问

注意：UUID 必须在 `datastore.data['watching']` 中存在，不存在的 UUID 会被拦截。匿名用户无法探测 UUID 是否存在——因为重定向目标是监控列表（本身需要登录），实际行为是被登录拦截。

#### download_patch（第 446-453 行）

```python
try:
    watch = datastore.data['watching'][uuid]
except KeyError:
    return make_response('Watch not found', 404)

dates = list(watch.history.keys())
if len(dates) < 2:
    return make_response('Not enough history', 400)
```

**校验链路：**
1. 同样用字典索引做 KeyError 校验，不存在时返回 404 而非重定向（API 风格响应）
2. 历史快照数 < 2 时返回 400

#### processor_asset（第 500-507 行）

```python
if uuid == 'first':
    uuid = list(datastore.data['watching'].keys()).pop()

try:
    watch = datastore.data['watching'][uuid]
except KeyError:
    flash(gettext("No history found for the specified link, bad link?"), "error")
    return redirect(url_for('watchlist.index'))
```

**校验链路：** 与 `diff_history_page` 完全一致。

### 1.4 UUID 校验的安全结论

| 风险场景 | 防御方式 | 效果 |
|----------|---------|------|
| 非标准 UUID 格式注入 | StrictUUIDConverter 格式校验 | 路由层直接 404 |
| 不存在的 UUID 探测 | 字典索引 KeyError → 重定向/404 | 阻止访问 |
| 路径穿越（如 `../`） | StrictUUIDConverter 拒绝非 UUID 字符 | 路由层直接 404 |
| `first` 哨兵值滥用 | 在视图函数中转换为合法 UUID，再走字典索引 | 与直接传 UUID 等价 |

---

## 二、frozenset 精确比对命名防御的出处（GHSA-vwgh-2hvh-4xm5）

### 2.1 漏洞背景

在修复前，认证检查可能使用了**子字符串匹配**来判断端点是否在白名单中，例如：

```python
# 漏洞代码（推测）
if 'diff_history_page' in request.endpoint and shared_diff_access:
    return func(*args, **kwargs)
```

问题在于：`diff_history_page_extract_GET` 和 `diff_history_page_extract_POST` 这两个端点的名字中也包含 `diff_history_page` 子串——

```
ui.ui_diff.diff_history_page              ← 应该放行
ui.ui_diff.diff_history_page_extract_GET  ← 包含子串，误放行！
ui.ui_diff.diff_history_page_extract_POST ← 包含子串，误放行！
```

而 `extract` 端点可以执行**攻击者提供的任意正则表达式**扫描历史快照并将结果写入磁盘 CSV，是一个高风险操作。

### 2.2 修复代码：精确端点名白名单

[auth_decorator.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py#L6-L14)：

```python
# Endpoints exempt from auth when `shared_diff_access` is enabled.
# Must be exact endpoint names — substring matching (GHSA-vwgh-2hvh-4xm5)
# let the state-changing `/diff/<uuid>/extract` endpoints slip through
# because their names share the `diff_history_page` prefix.
SHARED_DIFF_READ_ONLY_ENDPOINTS = frozenset({
    'ui.ui_diff.diff_history_page',
    'ui.ui_diff.processor_asset',
    'ui.ui_diff.download_patch',
})
```

**设计要点：**

1. **`frozenset`**：不可变集合，运行时无法被意外修改；查找 O(1)，比列表更高效
2. **精确 endpoint 名**：Flask 的 `request.endpoint` 格式为 `blueprint.function`，必须完全匹配
3. **注释明确标注 GHSA 编号**，追溯安全修复上下文

### 2.3 两层检查的防御深度

这套精确比对在**两个独立位置**同时执行，形成纵深防御：

**第一层：[flask_app.py check_authentication](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L526-L560)**（全局 before_request 钩子）

```python
elif request.endpoint in SHARED_DIFF_READ_ONLY_ENDPOINTS and datastore.data['settings']['application'].get('shared_diff_access'):
    return None  # 放行
else:
    return login_manager.unauthorized()
```

**第二层：[auth_decorator.py login_optionally_required](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py#L22-L42)**（路由装饰器）

```python
if request.endpoint in SHARED_DIFF_READ_ONLY_ENDPOINTS and datastore.data['settings']['application'].get('shared_diff_access'):
    return func(*args, **kwargs)
```

即使某一层被绕过（如 before_request 未注册、装饰器顺序错误），另一层仍会拦截。代码库中专门有 [test_auth_decorator_order.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/unit/test_auth_decorator_order.py) 用 AST 静态分析防止装饰器顺序错误（GHSA-jmrh-xmgh-x9j4）。

### 2.4 测试验证

[test_access_control.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/test_access_control.py#L51-L75) 明确验证了此修复：

```python
# GHSA-vwgh-2hvh-4xm5: shared_diff_access only covers the read-only
# diff page — the extract endpoints (which run an attacker-supplied
# regex against history and write a CSV to disk) must still require
# auth even when the share flag is enabled.
res = c.get(url_for("ui.ui_diff.diff_history_page_extract_GET", uuid="first"))
assert res.status_code == 302, "Extract form GET must redirect to login for anonymous users"

res = c.post(url_for("ui.ui_diff.diff_history_page_extract_POST", uuid="first"), ...)
assert res.status_code == 302, "Extract POST must redirect to login for anonymous users"
```

同时验证白名单内的端点确实可访问：

```python
res = c.get(url_for("ui.ui_diff.download_patch", uuid="first"))
assert res.status_code != 302, "download_patch must be reachable for shared diff viewers"

res = c.get(url_for("ui.ui_diff.processor_asset", uuid="first", asset_name="before"))
assert res.status_code != 302, "processor_asset must be reachable for shared diff viewers"
```

---

## 三、screenshot 与 favicon 静态资源的访问规则差异

两者都由 [flask_app.py static_content 路由](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L740-L822) 处理，但访问规则和实现细节有显著差异。

### 3.1 规则对比表

| 维度 | screenshot（`/static/screenshot/<uuid>`） | favicon（`/static/favicon/<uuid>`） |
|------|-------------------------------------------|------------------------------------|
| 代码位置 | 第 753-772 行 | 第 774-792 行 |
| **认证检查** | 有密码 + 未登录 → **再检查 `shared_diff_access`**，为 False 才 403 | 有密码 + 未登录 → **直接 403**，不看 `shared_diff_access` |
| **受分享开关控制** | ✅ 是 | ❌ 否 |
| **路径安全** | 拼接 `datastore_path/filename/screenshot_filename`，不校验 UUID 存在性 | 先 `datastore.data['watching'].get(filename)` 校验 UUID 存在，再从 `watch.data_dir` 读取 |
| **文件名** | 固定 `last-screenshot.png` 或 `last-error-screenshot.png`（由 `?error_screenshot=1` 决定） | 通过 `watch.get_favicon_filename()` 动态查找 `favicon.*` |
| **Cache-Control** | `no-cache, no-store, must-revalidate`（完全禁止缓存） | `max-age=300, must-revalidate`（缓存 5 分钟） |
| **UUID 不存在时** | `FileNotFoundError` → 404 | `watch is None` → 404 |
| **403 触发条件** | 有密码 + 未登录 + `shared_diff_access=False` | 有密码 + 未登录（任何情况） |

### 3.2 关键差异代码

**screenshot 的认证放行逻辑（第 753-757 行）：**

```python
if group == 'screenshot':
    if datastore.data['settings']['application']['password'] and not flask_login.current_user.is_authenticated:
        if not datastore.data['settings']['application'].get('shared_diff_access'):
            abort(403)
```

**favicon 的认证放行逻辑（第 774-777 行）：**

```python
if group == 'favicon':
    if datastore.data['settings']['application']['password'] and not flask_login.current_user.is_authenticated:
        abort(403)
```

截图的访问是 `shared_diff_access` 设计意图的一部分——diff 页面中会嵌入截图展示给匿名用户。而 favicon 被视为更敏感的指纹信息（暴露了被监控的目标网站域名），因此即使开启分享也不允许匿名访问。

### 3.3 其他静态资源组的认证规则

作为对比，其他相关资源组的访问控制：

| 资源组 | 认证规则 |
|--------|---------|
| `screenshot` | 有密码 + 未登录 + `shared_diff_access=False` → 403 |
| `favicon` | 有密码 + 未登录 → 403（不看 shared_diff_access） |
| `visual_selector_data` | 有密码 + 未登录 → 403（不看 shared_diff_access） |
| `js`, `styles`, `images` 等 | 无条件放行（不受密码保护） |

### 3.4 screenshot 的路径安全隐患

注意 screenshot 的文件拼接方式：

```python
response = make_response(send_from_directory(os.path.join(datastore_o.datastore_path, filename), screenshot_filename))
```

- `filename` 是路由参数中的 UUID 字符串，已在路由层被正则 `[^a-z0-9_-]+` 清洗（第 746 行 `re.sub(r'[^a-z0-9_-]+', '', group.lower())` 是对 `group` 的清洗，`filename` 本身未做同等级清洗——但路由参数中 `<string:filename>` 已自动阻止路径穿越字符）
- 没有像 favicon 那样先确认 `datastore.data['watching'].get(filename)` 存在

实际安全性依赖于 Flask/Werkzeug 的 `send_from_directory` 本身会阻止 `..` 穿越，且 UUID 格式由上层逻辑保证（screenshot 链接都是从已存在的 watch UUID 生成）。

---

## 四、不同处理器在 get_asset 函数上的实际返回行为

### 4.1 处理器发现与调用路径

`processor_asset` 端点通过 [get_processor_submodule](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/diff.py#L513-L539) 动态加载各处理器的 `difference.py` 模块：

```python
processor_module = get_processor_submodule(processor_name, 'difference')
if processor_module and hasattr(processor_module, 'get_asset'):
    result = processor_module.get_asset(asset_name=asset_name, watch=watch, ...)
    if result is None:
        abort(404, ...)
    binary_data, content_type, cache_control = result
    ...
else:
    abort(404, description=f"Processor '{processor_name}' does not support assets")
```

**返回契约**：`(binary_data: bytes, content_type: str, cache_control: str | None)` 或 `None`（表示资源不存在）。

### 4.2 现有处理器实现对比

代码库中共有 4 处 `render`/`get_asset` 实现，分布在 3 个处理器模块中：

| 处理器 | difference.get_asset | difference.render | preview.get_asset | preview.render |
|--------|----------------------|-------------------|-------------------|----------------|
| **text_json_diff** | ❌ 未实现 | ✅ [第 104 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/text_json_diff/difference.py#L104) | ❌ 未实现 | ❌ 未实现 |
| **image_ssim_diff** | ✅ [第 27 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L27) | ✅ [第 306 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L306) | ✅ [第 11 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L11) | ✅ [第 72 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L72) |
| **restock_diff** | ❌ 未实现 | ❌ 未实现 | ❌ 未实现 | ❌ 未实现 |

#### 4.2.1 text_json_diff（默认处理器）

`text_json_diff.difference.py` **只实现了 `render()`，没有 `get_asset()`**。因此匿名访问 `/diff/<uuid>/processor-asset/before` 时，会走到 `else` 分支返回：

```
404 Processor 'text_json_diff' does not support assets
```

这是合理的——文本 diff 直接内联在 HTML 中渲染，不需要额外的二进制资源。

#### 4.2.2 image_ssim_diff（截图对比处理器）

这是唯一完整实现了 `get_asset` 的处理器。

**difference.get_asset（[第 27-157 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L27-L157)）：**

| asset_name | 行为 | 返回格式 |
|-----------|------|---------|
| `'before'` | 从 `from_version` 历史快照读取截图，可选叠加蓝色 bounding box | `(img_bytes, mime, 'public, max-age=3600')` |
| `'after'` | 从 `to_version` 历史快照读取截图，可选叠加蓝色 bounding box | `(img_bytes, mime, 'public, max-age=3600')` |
| `'rendered_diff'` | 在隔离子进程中用 OpenCV 生成两张图的差异可视化（红色高亮） | `(jpeg_bytes, 'image/jpeg', 'public, max-age=300')` |
| 其他值 | 返回 `None` → 404 | — |

版本参数：从 URL query string 读取 `from_version` 和 `to_version`，若不存在则默认最近两张快照；若指定的时间戳不在 `watch.history` 中则回退到默认值。

**preview.get_asset（[第 11-69 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L11-L69)）：**

| asset_name | 行为 | 返回格式 |
|-----------|------|---------|
| `'screenshot'` | 从 `version` 参数指定的快照读取图片 | `(screenshot_bytes, mime, 'public, max-age=10')` |
| 其他值 | 返回 `None` → 404 | — |

但需注意：**preview 的 `processor_asset` 路由不在 `SHARED_DIFF_READ_ONLY_ENDPOINTS` 白名单中**，匿名访问会被重定向到登录页。

### 4.3 缓存策略对比

| 来源 | Cache-Control | 说明 |
|------|--------------|------|
| image_ssim_diff `before`/`after` | `public, max-age=3600` | 原始截图，缓存 1 小时 |
| image_ssim_diff `rendered_diff` | `public, max-age=300` | 计算产物，缓存 5 分钟 |
| image_ssim_diff preview screenshot | `public, max-age=10` | 预览截图，极短缓存 |
| favicon | `max-age=300, must-revalidate` | 缓存 5 分钟后必须重验证 |
| screenshot（static_content） | `no-cache, no-store, must-revalidate` | 完全禁止缓存 |
| API 返回的截图 | `max-age=300, must-revalidate` | 缓存 5 分钟 |

匿名通过白名单访问 `processor_asset` 获取的截图使用 `public, max-age=3600`，这意味着 CDN/浏览器可以缓存长达 1 小时——在关闭 `shared_diff_access` 后，已缓存的截图可能仍可被访问到（浏览器缓存侧）。这是当前设计的一个微妙边界：**认证拦截即时生效，但浏览器缓存不受服务端控制**。

### 4.4 get_asset 的潜在安全影响

`get_asset` 的参数中包含 `request`，意味着处理器可以读取 URL query string 中的任意参数（`from_version`, `to_version`, `version` 等）。目前的实现只做了"存在性检查"（值是否在 `watch.history` 中），这是安全的——因为 `watch.get_history_snapshot()` 内部有严格的 data_dir 边界检查（[Watch.py 第 568-572 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/model/Watch.py#L568-L572)）：

```python
if self.data_dir and not strtobool(os.getenv('HISTORY_SNAPSHOT_FILE_ALLOW_OUTSIDE_WATCH_DATADIR', 'False')):
    safe_data_dir = os.path.realpath(self.data_dir)
    resolved = os.path.realpath(filepath)
    if not (resolved.startswith(safe_data_dir + os.sep) or resolved == safe_data_dir):
        raise PermissionError(f"Snapshot path {filepath!r} is outside the watch data directory")
```

即使攻击者通过 query 参数传入恶意值，最终也会被 `get_history_snapshot` 限制在 watch 自己的 data_dir 内。

---

## 五、代码文件索引

| 文件 | 分析内容 |
|------|---------|
| [auth_decorator.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py) | `SHARED_DIFF_READ_ONLY_ENDPOINTS` 白名单定义 + `login_optionally_required` 装饰器 |
| [flask_app.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L75-L95) | `StrictUUIDConverter` UUID 格式强校验 |
| [flask_app.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L526-L560) | `check_authentication` 全局 before_request 钩子（第二层检查） |
| [flask_app.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L740-L822) | `static_content` 路由：screenshot vs favicon 访问规则 |
| [blueprint/ui/diff.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/diff.py) | 三个白名单端点实现 + `processor_asset` 调度 |
| [processors/image_ssim_diff/difference.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L27-L157) | image_ssim_diff `get_asset` 实现 |
| [processors/image_ssim_diff/preview.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/image_ssim_diff/preview.py#L11-L69) | image_ssim_diff preview `get_asset` 实现 |
| [processors/text_json_diff/difference.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/text_json_diff/difference.py) | text_json_diff `render`（无 `get_asset`） |
| [model/Watch.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/model/Watch.py#L568-L572) | `get_history_snapshot` data_dir 边界校验 |
| [tests/test_access_control.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/test_access_control.py) | GHSA-vwgh-2hvh-4xm5 修复验证测试 |
| [tests/unit/test_auth_decorator_order.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/unit/test_auth_decorator_order.py) | 装饰器顺序静态检查（GHSA-jmrh-xmgh-x9j4） |
