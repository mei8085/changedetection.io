# changedetection.io 只读分享展示逻辑分析

## 一、两套独立的"分享"机制

changedetection.io 存在两种功能不同、实现路径各异的分享机制，需严格区分。

### 1.1 配置导出/导入分享（`form_share_put_watch`）

这是一种**配置复制**机制，不是只读查看。

- 路由：`/share-url/<uuid>`，定义于 [blueprint/ui/\_\_init\_\_.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/__init__.py#L369-L415)
- 点击后把 watch 配置（剔除 history、notification 等敏感字段）POST 到 `https://changedetection.io/share/share`
- 返回的 `share_key` 拼接为 `https://changedetection.io/share/<share_key>`，存入 Flask session
- 其他用户在导入页面粘贴此链接后，[store/\_\_init\_\_.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/store/__init__.py#L684-L728) 检测到 URL 前缀 `https://changedetection.io/share/`，从远程服务器拉取 JSON 配置，创建新 watch
- 白名单字段只允许：`body`, `browser_steps`, `css_filter`, `extract_text`, `headers`, `ignore_text`, `include_filters`, `method`, `paused`, `processor`, `subtractive_selectors`, `tag`, `tags`, `text_should_not_be_present`, `title`, `trigger_text`, `url`, `use_page_title_in_list`, `webdriver_js_execute_code`

### 1.2 匿名只读 Diff 访问（`shared_diff_access`）

这是核心的"只读分享"功能，允许未登录用户查看监控项的历史变更页面。

- 全局开关，位于 Settings → "Allow anonymous access to watch history page when password is enabled"
- 表单字段定义于 [forms.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/forms.py#L1079)
- 默认值 `False`，定义于 [model/App.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/model/App.py#L71)
- **仅在密码保护启用时才有意义**——如果没有设密码，所有页面本来就不需要登录

---

## 二、只读页面隐藏了哪些可编辑入口

当 `shared_diff_access=True` 且密码保护启用、用户**未登录**时，系统在多个层面隐藏或阻断了可编辑入口。

### 2.1 路由层：白名单机制精确放行

[auth_decorator.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py#L10-L14) 定义了严格的白名单：

```python
SHARED_DIFF_READ_ONLY_ENDPOINTS = frozenset({
    'ui.ui_diff.diff_history_page',
    'ui.ui_diff.processor_asset',
    'ui.ui_diff.download_patch',
})
```

[login_optionally_required](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py#L16-L43) 装饰器在每次请求时判断：

```python
if request.endpoint in SHARED_DIFF_READ_ONLY_ENDPOINTS and datastore.data['settings']['application'].get('shared_diff_access'):
    return func(*args, **kwargs)  # 放行
```

**只有这 3 个端点被放行，其余所有端点都需要登录认证。**

#### 被阻断的可编辑/管理端点（未登录时重定向到登录页）

| 端点 | 功能 | 为何必须保护 |
|------|------|-------------|
| `ui.ui_edit.edit_page` | 编辑监控项 | 可修改 URL、过滤器、通知等所有配置 |
| `ui.ui_diff.diff_history_page_extract_GET/POST` | 数据提取（正则→CSV） | 可执行任意正则并写入磁盘（GHSA-vwgh-2hvh-4xm5 安全修复） |
| `ui.ui_preview.preview_page` | 单快照预览 | 不在白名单中，需登录 |
| `ui.form_share_put_watch` | 创建配置分享链接 | 可导出配置 |
| `ui.form_delete` | 删除监控项 | 破坏性操作 |
| `ui.form_clone` | 克隆监控项 | 可复制配置 |
| `ui.form_watch_checknow` | 立即重新检查 | 可触发网络请求 |
| `ui.clear_watch_history` | 清除历史 | 破坏性操作 |
| `ui.mark_all_viewed` | 标记全部已读 | 修改状态 |
| `ui.form_watch_list_checkbox_operations` | 批量操作（删除/暂停/静音等） | 破坏性/修改性操作 |
| `settings.settings_page` | 全局设置 | 可修改密码、fetch backend 等 |
| `imports.import_page` | 导入监控 | 可添加新监控项 |
| `tags.tags_overview_page` | 标签管理 | 可修改分组 |
| `watchlist.index` | 监控列表主页 | 不在白名单，需登录 |

### 2.2 导航栏（menu.html）：隐藏所有管理入口

[menu.html](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/templates/menu.html#L5-L38) 的核心条件：

```jinja2
{% if current_user.is_authenticated or not has_password %}
    {# 显示：GROUPS / SETTINGS / IMPORT / Pause / Mute #}
{% else %}
    {# 未登录+有密码：只显示 "Website Change Detection and Notification." #}
{% endif %}
```

未登录访问者看到的导航栏：
- **隐藏**：GROUPS、SETTINGS、IMPORT、Pause 按钮、Mute 按钮、搜索按钮
- **隐藏**：EDIT 链接（仅在 diff 页面上下文 `current_diff_url` 存在时认证用户可见）
- **显示**：LOG OUT（如果已登录）、LLM 模式切换、主题切换、语言切换、GitHub 链接
- **替代显示**：仅一条 "Website Change Detection and Notification." 文字

### 2.3 base.html 模板：前端交互降级

[base.html](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/templates/base.html) 中的关键逻辑：

- **Logo 链接**：未认证用户点击 logo 跳转到 `https://changedetection.io` 官网而非本站首页（第 59-65 行）
- **搜索模态框**：只在 `current_user.is_authenticated or not has_password` 时渲染（第 299 行）
- **JavaScript 变量**：`is_authenticated` 设为 `false`（第 39 行），前端 JS 据此禁用交互功能

### 2.4 Diff 页面（diff.html）：只读内容保留，编辑操作阻断

[diff.html](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/templates/diff.html) 中：

**访问者仍可使用的功能：**
- 版本选择器（From/To 下拉框）和 diff 偏好设置
- 键盘导航（Previous/Next 按钮）
- 文本 diff 内容查看
- 下载差异补丁（`download_patch` 在白名单中）
- 处理器资产加载（`processor_asset` 在白名单中）
- 截图查看（通过 `static_content` 路由，受 `shared_diff_access` 控制）

**被阻断的功能：**
- **Extract Data 标签页**：虽然在模板中渲染了链接（第 109 行），但 `diff_history_page_extract_GET/POST` 不在白名单中，点击后被重定向到登录页
- **"Goto single snapshot" 链接**（第 154 行）：指向 `ui.ui_preview.preview_page`，不在白名单中
- **"Highlight text to share or add to ignore lists" 提示**（第 159 行）：虽然文字可见，但实际交互（添加到 ignore list）需要认证的 `highlight_submit_ignore_url` 端点

**提示信息**（第 134-137 行）：
```jinja2
{%- if password_enabled_and_share_is_off -%}
    <div class="tip">{{ _('Pro-tip: You can enable <strong>"share access when password is enabled"</strong> from settings.')|safe }}</div>
{%- endif -%}
```
此变量在 [difference.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/text_json_diff/difference.py#L161-L163) 中计算：
```python
password_enabled_and_share_is_off = False
if datastore.data['settings']['application'].get('password') or os.getenv("SALTED_PASS", False):
    password_enabled_and_share_is_off = not datastore.data['settings']['application'].get('shared_diff_access')
```
当密码启用且分享关闭时，给已登录管理员一个提示，建议开启分享。

### 2.5 截图访问控制

[flask_app.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L753-L757) 中对截图资源的访问控制：

```python
if group == 'screenshot':
    if datastore.data['settings']['application']['password'] and not flask_login.current_user.is_authenticated:
        if not datastore.data['settings']['application'].get('shared_diff_access'):
            abort(403)
```

| 场景 | 结果 |
|------|------|
| 无密码保护 | 截图可访问 |
| 有密码 + `shared_diff_access=True` | 截图可访问（匿名） |
| 有密码 + `shared_diff_access=False` | 403 Forbidden |

Favicon 资源更严格——只要有密码且未登录就直接 403，不受 `shared_diff_access` 影响。

---

## 三、关闭分享开关后旧链接如何立即失效

### 3.1 核心机制：实时读取全局开关，无缓存/无 token

`shared_diff_access` 是存储在 datastore 中的一个布尔值，**每次 HTTP 请求时实时读取**，不存在任何中间缓存层或独立 token 机制。

这意味着：
- 关闭开关 → 值从 `True` 变为 `False` → **下一条请求立即被拒绝**
- 无需等待缓存过期、无需撤销 token、无需重启服务

### 3.2 请求处理全链路分析

以匿名用户访问 `/diff/<uuid>` 为例，完整链路如下：

```
请求进入
  │
  ▼
flask_app.py 的 before_request 钩子（第 544 行）
  ├─ 检查 request.endpoint 是否在 SHARED_DIFF_READ_ONLY_ENDPOINTS
  ├─ 实时读取 datastore.data['settings']['application'].get('shared_diff_access')
  ├─ 如果两者都为 True → 放行（return None）
  └─ 否则 → 调用 login_manager.unauthorized()，重定向到登录页
  │
  ▼
login_optionally_required 装饰器（auth_decorator.py 第 33 行）
  ├─ 同样检查 endpoint + shared_diff_access
  ├─ 放行或拦截
  └─ 未放行 → current_app.login_manager.unauthorized()
```

两层检查机制确保即使某一层被绕过，另一层仍会拦截。

### 3.3 各资源类型的立即失效行为

| 资源类型 | 关闭开关后的行为 | 生效时间 |
|----------|-----------------|---------|
| Diff 页面 (`/diff/<uuid>`) | 302 重定向到 `/login` | 立即 |
| 处理器资产 (`/diff/<uuid>/processor-asset/*`) | 302 重定向到 `/login` | 立即 |
| 下载补丁 (`/diff/<uuid>/download-patch`) | 302 重定向到 `/login` | 立即 |
| 截图 (`/static/screenshot/<uuid>`) | 403 Forbidden | 立即 |
| 数据提取 (`/diff/<uuid>/extract`) | 302 重定向到 `/login`（本来就需认证） | 立即 |
| 编辑/管理页面 | 302 重定向到 `/login`（本来就需认证） | 立即 |

### 3.4 为何能做到"立即"——没有独立分享链接的设计

注意 changedetection.io 的 `shared_diff_access` 机制**不是**为每个监控项生成独立的分享链接（如 `/share/abc123`），而是一个**全局开关**，控制是否允许匿名访问 diff 页面。

这意味着：
- 不存在"旧链接"的概念——匿名用户访问的是标准的 `/diff/<uuid>` 路径
- 关闭开关后，不是"撤销某条链接"，而是"关闭整个匿名访问通道"
- 所有曾经能访问的 diff 页面，在同一瞬间全部变为需要登录
- 不存在"部分分享"——要么全部监控项的 diff 页面都对匿名开放，要么全部关闭

### 3.5 与配置分享链接的对比

| 特性 | `shared_diff_access` | `form_share_put_watch` |
|------|---------------------|----------------------|
| 作用 | 允许匿名查看 diff 页面 | 导出配置供他人导入 |
| 链接形式 | 标准 `/diff/<uuid>` | `https://changedetection.io/share/<key>` |
| 粒度 | 全局开关 | 单个监控项 |
| 失效方式 | 关闭开关立即生效 | 远程服务器侧的 share_key 生命周期（本地无法控制） |
| 访问内容 | 变更历史、diff、截图 | 仅监控配置（不含历史和通知） |

---

## 四、关键代码文件索引

| 文件 | 职责 |
|------|------|
| [auth_decorator.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py) | 白名单定义 + `login_optionally_required` 装饰器 |
| [flask_app.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L544) | `before_request` 钩子中的第二层分享访问检查 |
| [flask_app.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L753-L757) | 截图资源访问控制 |
| [blueprint/ui/\_\_init\_\_.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/__init__.py#L369-L415) | `form_share_put_watch` 配置分享路由 |
| [blueprint/ui/diff.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/diff.py) | Diff 页面路由、extract 端点、download_patch、processor_asset |
| [processors/text_json_diff/difference.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/text_json_diff/difference.py#L161-L163) | `password_enabled_and_share_is_off` 变量计算 |
| [processors/extract.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/processors/extract.py#L47-L49) | Extract 页面中的 `password_enabled_and_share_is_off` 计算 |
| [templates/base.html](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/templates/base.html) | 全局模板：导航栏、搜索框、is_authenticated JS 变量 |
| [templates/menu.html](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/templates/menu.html) | 导航菜单：按认证状态条件渲染 |
| [blueprint/ui/templates/diff.html](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/blueprint/ui/templates/diff.html) | Diff 页面模板：版本选择、Extract Data 标签、截图 |
| [model/App.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/model/App.py#L71) | `shared_diff_access` 默认值定义 |
| [forms.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/forms.py#L1079) | `shared_diff_access` 表单字段 |
| [store/\_\_init\_\_.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/store/__init__.py#L684-L728) | 配置分享链接导入逻辑 |
| [tests/test_share_watch.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/test_share_watch.py) | 配置分享功能测试 |
| [tests/test_access_control.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/test_access_control.py) | 访问控制测试（含 GHSA-vwgh-2hvh-4xm5 安全修复验证） |
