# 监控项分享与访问控制机制分析报告

## 一、分享链接生成机制

### 1.1 分享入口与流程

分享功能通过路由 `/share-url/<uuid>` 触发，对应的处理函数是 `form_share_put_watch` [blueprint/ui/__init__.py:369-415](changedetectionio/blueprint/ui/__init__.py:369-415)。

**核心流程：**

1.  **数据提取与清理**：
    - 对 Watch 对象进行深拷贝，避免修改原始数据
    - 删除 `history` 字段（不分享历史快照）
    - 删除所有 `notification_` 开头的字段（保护通知配置隐私）
    - 删除 `uuid`、`last_checked`、`last_changed` 等内部状态字段

2.  **全局配置合并**：
    - 将全局 `ignore_text` 追加到 Watch 的 `ignore_text` 中
    - 将全局 `subtractive_selectors` 追加到 Watch 的 `subtractive_selectors` 中

3.  **上传到中央分享服务器**：
    - 通过 POST 请求发送到 `https://changedetection.io/share/share`
    - 请求头携带 `App-Guid` 用于标识来源实例
    - 服务器返回 `share_key`，拼接成最终分享链接：`https://changedetection.io/share/{share_key}`

4.  **会话存储**：
    - 分享链接存储在 Flask Session 中，用于在页面上展示给用户

### 1.2 分享链接的导入

当用户导入以 `https://changedetection.io/share/` 开头的 URL 时，系统会自动从分享服务器获取元数据 [store/__init__.py:683-728](changedetectionio/store/__init__.py:683-728)。

**导入字段白名单**（仅允许导入以下字段）：
```
body, browser_steps, css_filter, extract_text, headers, ignore_text,
include_filters, method, paused, previous_md5, processor,
subtractive_selectors, tag, tags, text_should_not_be_present,
title, trigger_text, url, use_page_title_in_list, webdriver_js_execute_code
```

**特殊处理**：
- `css_filter` 字段会被重命名为 `include_filters` 并转换为列表格式
- 请求超时限制为 5 秒，防止阻塞
- 携带 `App-Guid` 请求头以获取 JSON 格式响应

---

## 二、匿名访客与已登录用户的访问差异

### 2.1 认证装饰器机制

系统使用 `login_optionally_required` 装饰器进行访问控制 [auth_decorator.py:16-43](changedetectionio/auth_decorator.py:16-43)。

**豁免规则（按优先级）：**

1.  **只读 Diff 页面豁免**：当 `shared_diff_access` 设置启用时，以下端点允许匿名访问：
    ```python
    SHARED_DIFF_READ_ONLY_ENDPOINTS = frozenset({
        'ui.ui_diff.diff_history_page',      # Diff 查看页面
        'ui.ui_diff.processor_asset',         # 处理器资源（如截图）
        'ui.ui_diff.download_patch',          # Patch 文件下载
    })
    ```

2.  **方法豁免**：`OPTIONS` 等 HTTP 方法自动豁免
3.  **登录禁用豁免**：当全局禁用登录时全部豁免
4.  **已登录用户豁免**：已认证用户可访问所有页面

### 2.2 匿名用户权限边界

**允许匿名访问的内容（当 shared_diff_access=True 时）：**

| 资源 | 路径 | 说明 |
|------|------|------|
| Diff 页面 | `/diff/<uuid>` | 查看历史变更对比 |
| Patch 下载 | `/diff/<uuid>/download-patch` | 下载统一 diff 格式文件 |
| 处理器资源 | `/diff/<uuid>/processor-asset/<asset>` | 如图片对比的前后截图 |
| 截图资源 | `/static/screenshot/<uuid>` | 监控页面截图 |
| 静态资源 | `/static/js/*`, `/static/styles/*` | JS、CSS 等前端资源 |

**禁止匿名访问的内容（即使 shared_diff_access=True）：**

| 资源 | 路径 | 安全原因 |
|------|------|----------|
| 数据提取 | `/diff/<uuid>/extract` (GET/POST) | 防止攻击者运行任意正则表达式提取历史数据并写入 CSV [test_access_control.py:55-64](changedetectionio/tests/test_access_control.py:55-64) |
| LLM 摘要 | `/diff/<uuid>/llm-summary` | 防止消耗 Token 预算 |
| 收藏图标 | `/static/favicon/<uuid>` | 防止间接信息泄露 |
| 视觉选择器数据 | `/static/visual_selector_data/<uuid>` | 包含页面结构敏感信息 |
| API 接口 | `/api/v1/*` | API 使用独立的 Token 认证机制 |
| 监控列表 | `/` | 监控项概览 |
| 设置页面 | `/settings` | 系统配置 |

### 2.3 登录流程中的特殊处理

在 `flask_app.py` 的 `login_manager.request_loader` 中也有类似的豁免逻辑 [flask_app.py:530-560](changedetectionio/flask_app.py:530-560)，额外豁免的端点包括：
- RSS Feed (`/rss/*`) - 使用独立的 Token 参数认证
- Socket.IO 通道 - 使用独立的认证机制
- 登录页面本身
- 语言切换页面

---

## 三、分组（Tag/Group）数据模型与实现机制

### 3.1 分组的实体形态

**核心模型定义**：

分组（Tag）是一个完整的领域模型，继承自 `watch_base` 基类，可复用 Watch 的所有字段 [model/Tag.py:26-42](changedetectionio/model/Tag.py:26-42)：

```python
class model(EntityPersistenceMixin, watch_base):
    """Tag domain model - groups watches and can override their settings."""
```

**实体字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `uuid` | str | 唯一标识符 |
| `title` | str | 显示名称 |
| `overrides_watch` | bool | 是否启用覆盖模式，优先级高于 Watch 设置 |
| `url_match_pattern` | str | URL 自动匹配模式，支持通配符 |
| `tag_colour` | str | 标签颜色 |
| `llm_intent` | str | AI 变更意图提示词 |
| `llm_change_summary` | str | AI 变更摘要模板 |
| `notification_*` | 多字段 | 完整的通知配置（与 Watch 相同） |
| `processor_config_*` | dict | 处理器配置（如补货监控设置） |
| ... | ... | 所有 `watch_base` 中的字段 |

**持久化机制**：
- 存储在 `datastore.data['settings']['application']['tags']` 字典中
- 每个 Tag 有独立的目录和 `tag.json` 文件 [model/Tags.py:9-29](changedetectionio/model/Tags.py:9-29)
- 删除 Tag 时通过 `TagsDict.__delitem__` 自动清理文件系统

### 3.2 分组与 Watch 的关联机制

**关联方式**：
- Watch 通过 `tags` 字段（列表）引用 Tag UUID
- 支持手动分配和自动匹配两种方式

**自动匹配机制**：
- Tag 可配置 `url_match_pattern`，支持通配符（`*`, `?`, `[ ]`）或子串匹配
- `get_all_tags_for_watch()` 方法返回 Watch 的所有关联 Tag，包括：
  1. 手动分配的 Tag
  2. URL 模式自动匹配的 Tag [store/__init__.py:980-994](changedetectionio/store/__init__.py:980-994)

```python
# 手动分配的 Tag
result = dictfilt(self.__data['settings']['application']['tags'], watch.get('tags', []))
# URL 自动匹配的 Tag
for tag_uuid, tag in self.__data['settings']['application']['tags'].items():
    if tag_uuid not in result and tag.matches_url(watch_url):
        result[tag_uuid] = tag
```

### 3.3 分组与通知配置的耦合点

**三级级联覆盖机制**：

通知配置采用 Watch → Tag → Global 的三级优先级链，在 `_check_cascading_vars()` 中实现 [notification_service.py:17-54](changedetectionio/notification_service.py:17-54)：

```python
def _check_cascading_vars(datastore, var_name, watch):
    # Level 1: Watch 级别（最高优先级）
    v = watch.get(var_name)
    if v and not watch.get('notification_muted'):
        return v

    # Level 2: Tag 级别（遍历所有关联 Tag）
    tags = datastore.get_all_tags_for_watch(uuid=watch.get('uuid'))
    if tags:
        for tag_uuid, tag in tags.items():
            v = tag.get(var_name)
            if v and not tag.get('notification_muted'):
                return v  # 第一个匹配的 Tag 生效

    # Level 3: Global 级别（最低优先级）
    if datastore.data['settings']['application'].get(var_name):
        return datastore.data['settings']['application'].get(var_name)

    return None
```

**可被 Tag 覆盖的通知字段**：
- `notification_urls` - 通知目标 URL
- `notification_title` - 通知标题模板
- `notification_body` - 通知正文模板
- `notification_format` - 通知格式（text/html/markdown）
- `notification_muted` - 通知静音状态
- `notification_screenshot` - 是否包含截图
- `notification_alert_count` - 通知告警计数

**静音状态的级联**：
- 任意一级设置 `notification_muted=True` 会阻止该级别及其下级的通知
- Watch 静音不影响 Tag，Tag 静音不影响 Global，但会跳过本级

### 3.4 分享时的分组处理

**分享导出时**：
- Watch 的 `tags` 字段（UUID 列表）会包含在分享数据中
- 但 Tag 实体本身（包括其通知配置、覆盖规则）**不会**被分享
- 接收方导入后，若不存在对应 UUID 的 Tag，则标签关联失效

**分享导入时**：
- `tags` 字段在导入白名单中 [store/__init__.py:710](changedetectionio/store/__init__.py:710)
- 但仅导入 UUID 引用，不导入 Tag 实体
- `overrides_watch`、`notification_*` 等 Tag 级字段不在导入白名单中

### 3.5 匿名与登录用户在分组功能上的权限差异

**所有分组管理端点都受 `@login_optionally_required` 保护** [blueprint/tags/__init__.py:14-264](changedetectionio/blueprint/tags/__init__.py:14-264)：

| 功能 | 路由 | 登录用户 | 匿名用户 |
|------|------|---------|---------|
| 查看分组列表 | `/tags/list` | ✅ 可访问 | ❌ 重定向到登录 |
| 创建分组 | `/tags/add` | ✅ 可创建 | ❌ 重定向到登录 |
| 编辑分组 | `/tags/edit/<uuid>` | ✅ 可编辑 | ❌ 重定向到登录 |
| 静音分组 | `/tags/mute/<uuid>` | ✅ 可操作 | ❌ 重定向到登录 |
| 删除分组 | `/tags/delete/<uuid>` | ✅ 可删除 | ❌ 重定向到登录 |
| 取消关联 | `/tags/unlink/<uuid>` | ✅ 可操作 | ❌ 重定向到登录 |

**间接权限边界**：
- 匿名用户无法通过 `/tags/*` 路由直接管理分组
- 但匿名用户通过 Diff 页面可**间接看到** Tag 级通知配置的效果（如通知标题、正文内容）
- 匿名用户无法知道具体使用了哪个 Tag 或其 UUID

---

## 四、失效、撤销与历史 Diff 暴露面处理

### 4.1 分享链接的有效性边界

**已实现的约束**：

| 约束 | 实现位置 | 说明 |
|------|---------|------|
| 历史快照不分享 | [blueprint/ui/__init__.py:382-383](changedetectionio/blueprint/ui/__init__.py:382-383) | `del watch['history']` 明确删除 |
| 通知配置不分享 | [blueprint/ui/__init__.py:386-388](changedetectionio/blueprint/ui/__init__.py:386-388) | 遍历删除所有 `notification_` 前缀字段 |
| 导入字段白名单 | [store/__init__.py:696-717](changedetectionio/store/__init__.py:696-717) | 仅允许 21 个安全字段 |
| 请求超时保护 | [store/__init__.py:688-692](changedetectionio/store/__init__.py:688-692) | 导入时 5 秒超时限制 |

**现状限制（未实现的功能）**：

| 功能 | 现状 | 代码证据 |
|------|------|---------|
| 本地撤销机制 | ❌ 未实现 | 整个代码库无 `revoke_share`、`delete_share` 等相关逻辑 |
| 过期时间 | ❌ 未实现 | 分享数据中无 `expires_at`、`ttl` 等字段 |
| 分享链接列表 | ❌ 未实现 | 分享链接仅存储在 Flask Session 中，刷新即丢失 |
| 修改联动 | ❌ 未实现 | 原 Watch 修改后不会通知中央服务器更新 |
| 访问日志 | ❌ 未实现 | 无代码记录谁访问了分享链接 |
| 访问密码 | ❌ 未实现 | 分享链接无需密码即可导入 |

**风险提示**：一旦分享，数据就上传到了第三方服务器 `changedetection.io`，本地无法控制其后续访问、修改或删除。

### 4.2 历史 Diff 暴露面控制

#### 4.2.1 分享时的历史数据隔离

- **历史快照不分享**：`history` 字段在分享时被明确删除 [blueprint/ui/__init__.py:382-383](changedetectionio/blueprint/ui/__init__.py:382-383)
- 分享的仅包含监控配置（URL、过滤器、Headers 等），不包含任何历史快照数据
- **注意**：`previous_md5` 字段在导入白名单中，但这只是校验和，不包含实际内容

#### 4.2.2 匿名访问时的历史数据控制

当 `shared_diff_access=True` 时，匿名用户可通过 Diff 页面访问历史数据，但有以下限制：

1.  **仅能访问已有快照**：匿名用户无法触发新的检查
2.  **只读权限**：仅能查看，不能修改监控配置
3.  **Diff 页面的特殊处理**：
    - 在 `difference.py` 中通过 `password_enabled_and_share_is_off` 变量控制 UI 元素显示 [processors/text_json_diff/difference.py:161-163](changedetectionio/processors/text_json_diff/difference.py:161-163)
    - 当密码启用且分享关闭时，会隐藏敏感操作按钮

4.  **截图访问控制** [flask_app.py:753-757](changedetectionio/flask_app.py:753-757)：
    ```python
    if group == 'screenshot':
        if datastore.data['settings']['application']['password'] and not flask_login.current_user.is_authenticated:
            if not datastore.data['settings']['application'].get('shared_diff_access'):
                abort(403)
    ```

#### 4.2.3 安全修复记录（GHSA-vwgh-2hvh-4xm5）

**问题**：早期版本中使用前缀匹配豁免端点，导致 `/diff/<uuid>/extract` 等危险端点被错误豁免。

**修复**：改为精确的端点名称匹配 [auth_decorator.py:6-14](changedetectionio/auth_decorator.py:6-14)：
```python
# 修复前：前缀匹配（不安全）
if 'diff_history_page' in request.endpoint: ...

# 修复后：精确集合匹配
SHARED_DIFF_READ_ONLY_ENDPOINTS = frozenset({
    'ui.ui_diff.diff_history_page',
    'ui.ui_diff.processor_asset',
    'ui.ui_diff.download_patch',
})
```

**测试验证** [test_access_control.py:55-64](changedetectionio/tests/test_access_control.py:55-64)：
```python
# Extract GET/POST 必须重定向到登录
res = c.get(url_for("ui.ui_diff.diff_history_page_extract_GET", uuid="first"))
assert res.status_code == 302

res = c.post(url_for("ui.ui_diff.diff_history_page_extract_POST", uuid="first"), data={...})
assert res.status_code == 302
```

---

## 五、架构总结

### 5.1 两种分享模式对比

| 特性 | 配置链接分享 | 匿名 Diff 访问 |
|------|-------------|---------------|
| 启用方式 | 点击"分享"按钮 | 设置 `shared_diff_access=True` |
| 数据流向 | 上传到 changedetection.io | 直接访问本实例 |
| 分享内容 | 监控配置（无历史） | 历史 Diff + 快照 |
| 访问控制 | 拥有链接即可导入 | 知道 UUID 即可查看 |
| 撤销能力 | 依赖中央服务器 | 关闭 `shared_diff_access` 即可 |
| 适用场景 | 分享监控模板给他人 | 团队内共享变更结果 |

### 5.2 分组（Tag）架构特性

| 特性 | 说明 |
|------|------|
| 继承关系 | Tag 继承自 watch_base，与 Watch 共享所有字段 |
| 覆盖机制 | Watch → Tag → Global 三级级联，`overrides_watch` 控制优先级 |
| 自动匹配 | 支持 URL 模式自动分配 Tag 到 Watch |
| 持久化 | 每个 Tag 独立存储，删除时自动清理文件 |
| 分享范围 | 仅分享 Watch 中的 Tag UUID 引用，不分享 Tag 实体 |

### 5.3 安全设计原则

1.  **最小权限原则**：匿名用户仅能访问只读 Diff 相关端点
2.  **隐私保护**：通知配置等敏感信息在分享时完全剥离
3.  **白名单机制**：导入时仅允许指定字段，防止恶意字段注入
4.  **纵深防御**：
    - 路由层：`login_optionally_required` 装饰器
    - 静态资源层：`static_content` 路由的额外检查
    - 视图层：根据 `password_enabled_and_share_is_off` 控制 UI

### 5.4 潜在改进点

1.  **分享链接管理**：增加本地分享链接列表，支持查看和撤销
2.  **过期机制**：支持设置分享链接的过期时间
3.  **端到端加密**：分享数据在上传前加密，仅分享方可解密
4.  **访问日志**：记录匿名用户对 Diff 页面的访问
5.  **标签映射**：跨实例分享时支持标签名称到 UUID 的映射
6.  **Tag 分享**：支持单独分享 Tag 配置（不含敏感信息）

---

## 六、关键代码索引

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 分享链接生成 | `blueprint/ui/__init__.py` | 369-415 |
| 分享链接导入 | `store/__init__.py` | 683-728 |
| 认证装饰器 | `auth_decorator.py` | 1-43 |
| Diff 路由定义 | `blueprint/ui/diff.py` | 97-157 |
| 匿名访问豁免列表 | `auth_decorator.py` | 10-14 |
| 登录请求加载器 | `flask_app.py` | 530-560 |
| 静态资源访问控制 | `flask_app.py` | 740-818 |
| Diff 页面权限变量 | `processors/text_json_diff/difference.py` | 161-163 |
| 导入字段白名单 | `store/__init__.py` | 696-717 |
| 通知字段删除逻辑 | `blueprint/ui/__init__.py` | 386-388 |
| Tag 模型定义 | `model/Tag.py` | 26-68 |
| Tag 集合类 | `model/Tags.py` | 9-39 |
| 通知级联覆盖 | `notification_service.py` | 17-54 |
| 获取 Watch 的所有 Tag | `store/__init__.py` | 980-994 |
| Tag 管理路由 | `blueprint/tags/__init__.py` | 14-264 |
| 访问控制测试 | `tests/test_access_control.py` | 55-75 |
