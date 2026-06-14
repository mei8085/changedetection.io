# 端点与插件资源审计

本文分析 changedetection.io 中三条"尾线"安全边界：三个运维端点（gc-cleanup/worker-health/queue-status）的装饰器正确性与信息泄露面，以及 plugin 资源组的实际文件供给范围。

---

## 一、三个运维端点的装饰器顺序审计

### 1.1 端点位置与装饰器排列

三个端点都定义在 [flask_app.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L902-L983) 内部，统一使用 `@app.route` + `@login_optionally_required` 组合。

| 端点 | 路由 | 装饰器顺序 |
|------|------|-----------|
| `gc_cleanup` | `/gc-cleanup` | [第 903-904 行：`@app.route` 在外，`@login_optionally_required` 在内 |
| `worker_health` | `/worker-health` | [第 913-914 行：`@app.route` 在外，`@login_optionally_required` 在内 |
| `queue_status` | `/queue-status` | [第 940-941 行：`@app.route` 在外，`@login_optionally_required` 在内 |

**代码样例（gc_cleanup）：**

```python
@app.route('/gc-cleanup', methods=['GET'])
@login_optionally_required
def gc_cleanup():
    ...
```

### 1.2 顺序是否正确？

✅ **正确**。三个端点的装饰器排列都是 `@app.route`（最外层 / 最上面）、`@login_optionally_required`（内层 / 下面）。

为什么这是正确顺序：
- Python 装饰器求值顺序是**从下往上**应用的
- `@login_optionally_required` 先作用于原始函数 `gc_cleanup`，返回包装后的函数
- 然后 `@app.route` 接收到这个**已包装**的函数并注册到路由
- 因此路由表中存储的是带认证保护的函数

如果写反的后果（GHSA-jmrh-xmgh-x9j4）：
- `@login_optionally_required` 在最上面 → 最外层
- `@app.route` 先注册了**原始裸函数**到路由表
- `@login_optionally_required` 的包装结果被丢弃
- 认证完全失效，匿名可直接访问

### 1.3 AST 静态检查能否覆盖这三个端点？

✅ **能覆盖**。

[test_auth_decorator_order.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/unit/test_auth_decorator_order.py) 的检测逻辑：

```python
def _is_route_decorator(node: ast.expr) -> bool:
    return (
        isinstance(node, ast.Call)
        and isinstance(node.func, ast.Attribute)
        and node.func.attr == "route"
    )

def _is_auth_decorator(node: ast.expr) -> bool:
    return isinstance(node, ast.Name) and node.id == "login_optionally_required"
```

扫描范围是 `changedetectionio/` 目录下所有 `.py` 文件，包括 `flask_app.py`。

检测逻辑只检查"auth 装饰器索引 < route 装饰器索引"即违规。三个端点中 route 在第 0 位（最外层最上面），auth 在第 1 位（内层），所以 `auth_idx > route_idx`，不触发违规。

**检测盲区（已知绕过方式）：
- 别名导入 `from auth_decorator import login_optionally_required as auth_req` → 检测不到
- 变量赋值 `auth = login_optionally_required` → 检测不到
- 但三个端点目前都是直接用原名，无别名，都能检测到

---

## 二、无密码部署下的内部运行画像泄露

### 2.1 认证行为差异：无密码 = 全部公开

`login_optionally_required` 的核心逻辑（[auth_decorator.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py#L30-L42)）：

```python
has_password_enabled = datastore.data['settings']['application'].get('password') or os.getenv("SALTED_PASS", False)
...
if has_password_enabled and not current_user.is_authenticated:
    return current_app.login_manager.unauthorized()
return func(*args, **kwargs)
```

当 `has_password_enabled = False` 时，函数直接返回 `func(*args, **kwargs)` —— 所有端点**全部放行**。

`before_request` 的 `check_authentication` 也有同样逻辑：有密码 + 未登录才拦截。

**结论：无密码部署下，整个系统没有任何访问控制。** 匿名用户不仅能访问所有页面和所有 API，包括设置、编辑、删除、导入导出等所有操作。

### 2.2 三个运维端点具体泄露的内部运行画像

#### `/gc-cleanup`（[第 903-910 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L903-L910)）

```python
def gc_cleanup():
    result = memory_cleanup(app)
    return jsonify({"status": "success", "message": "Memory cleanup completed", "result": result})
```

**返回数据：
- `status`: "success"
- `message`: "Memory cleanup completed"
- `result`: "cleaned"（字符串）

**泄露信息**：
- 确认系统存在（指纹识别）
- 可以**触发**内存清理操作（可被滥用来频繁触发 GC 造成性能下降 / 影响正常业务）

**实际影响**：
- 信息泄露量小，但可被用于**拒绝服务**——反复调用触发 GC 清理，影响爬虫 worker 性能
- `memory_cleanup` 内部执行 `gc.collect()`、`malloc_trim(0)`、`re.purge()` 等操作，都是 CPU/内存密集型操作

#### `/worker-health`（[第 913-937 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L913-L937)）

```python
def worker_health():
    expected_workers = int(os.getenv("FETCH_WORKERS", ...))
    status = worker_pool.get_worker_status()
    health_result = worker_pool.check_worker_health(...)
    return jsonify({
        "status": "success",
        "worker_status": status,
        "health_check": health_result,
        "expected_workers": expected_workers
    })
```

**`get_worker_status()** 返回（[worker_pool.py 第 439-446 行）：
```python
{
    'worker_type': 'async',
    'worker_count': <int,
    'running_uuids': [<uuid1', ...],
    'active_threads': <int>,
}
```

**`check_worker_health()** 返回（[worker_pool.py 第 498-522 行）：
```python
{
    'status': 'healthy' | ...,
    'expected_count': <int>,
    'actual_count': <int>,
    'message': 'All N async workers running'
}
```

**泄露信息：**
- Worker 类型（async）
- 预期 worker 数量、实际运行中的 worker 数量
- **正在运行的监控项 UUID 列表（`running_uuids`）——这是敏感信息！
- 活跃线程数
- 系统健康状态

**实际影响：**
- 泄露内部系统规模（worker 数量）
- **泄露正在处理的监控项 UUID
- 可用于探测哪些 UUID 正在被处理（时序分析）
- 可被滥用来频繁检查 worker 状态（对系统影响不大）

#### `/queue-status`（[第 940-982 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L940-L982)）

支持三种查询模式：

**模式 1：查询特定 UUID 的位置（`?uuid=<uuid>`）
```python
position_info = update_q.get_uuid_position(target_uuid)
return jsonify({
    "status": "success",
    "uuid": target_uuid,
    "queue_position": position_info
})
```

返回结构（[queue_handlers.py 第 271-299 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/queue_handlers.py#L271-L299)）：
```python
{
    'position': <int or None>,
    'total_items': <int>,
    'priority': <int or None>,
    'found': <bool>
}
```

**模式 2：队列摘要（`?summary=1`）
```python
summary = update_q.get_queue_summary()
return jsonify({
    "status": "success",
    "queue_summary": summary
})
```

返回结构（[queue_handlers.py 第 336-365 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/queue_handlers.py#L336-L365)）：
```python
{
    'total_items': <int>,
    'priority_breakdown': {<priority>: <count>, ...},
    'immediate_items': <int>,   # priority == 1
    'clone_items': <int>,       # priority == 5
    'scheduled_items': <int>,   # priority > 100
}
```

**模式 3：完整队列列表（默认，分页）
```python
all_queued = update_q.get_all_queued_uuids(limit=..., offset=...)
return jsonify({
    "status": "success",
    "queue_size": update_q.qsize(),
    "queued_data": all_queued
})
```

返回结构（[queue_handlers.py 第 301-334 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/queue_handlers.py#L301-L334)）：
```python
{
    'items': [
        {'uuid': '<uuid>', 'position': <int>, 'priority': <int>},
        ...
    ],
    'total_items': <int>,
    'returned_items': <int>,
    'has_more': <bool>
}
```

**泄露信息：**
- 队列总大小（任务积压量）
- **队列中所有监控项的 UUID 完整列表**（可分页获取，默认 limit=50 或全部）
- 每个 UUID 的优先级和位置
- 优先级分布（立即/克隆/定时任务数量

**实际影响：**
- **严重信息泄露——可获取所有待抓取目标 UUID 清单（相当于监控项清单泄露是最大的泄露！
- 可了解系统负载和任务量
- 可探测特定 UUID 是否在队列中
- 可用于了解监控项的优先级配置

### 2.3 无密码部署下的整体威胁模型

| 端点 | 信息敏感度 | 可被滥用方式 |
|------|-----------|-------------|
| `/gc-cleanup` | 低（只返回 success/cleaned | DoS（频繁触发 GC） |
| `/worker-health` | **中高**（泄露运行中 UUID 列表 + worker 数量） | 信息收集、时序分析 |
| `/queue-status` | **高**（泄露全部队列 UUID 列表 + 优先级） | 批量枚举所有监控项、信息收集 |

注意：这些只是"额外"的运维端点，不是主要信息泄露面相比主页面。在无密码部署下，本来就能访问所有监控项和设置。这些端点提供的是**更结构化、更适合自动化的系统内部运行画像，更方便攻击者批量收集。

---

## 三、plugin 资源组实际供给审计

### 3.1 plugin 组的路由与发现机制

[flask_app.py static_content 中 plugin 分支（[第 823-846 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L823-L846)）：

```python
if group == 'plugin':
    from changedetectionio.pluggy_interface import plugin_manager
    for plugin_name, plugin_obj in plugin_manager.list_name_plugin():
        if hasattr(plugin_obj, 'plugin_static_path'):
            try:
                static_path = plugin_obj.plugin_static_path()
                if static_path and os_check.path.isdir(static_path):
                    plugin_file_path = os_check.path.join(static_path, filename)
                    if os_check.path.isfile(plugin_file_path):
                        response = make_response(send_from_directory(static_path, filename))
                        response.headers['Cache-Control'] = 'max-age=3600, public'
                        return response
            except Exception as e:
                logger.debug(...)
                pass
    abort(404)
```

**工作流程：**
1. 遍历所有已注册的 pluggy 插件
2. 检查插件是否有 `plugin_static_path` 方法
3. 调用该方法获取静态文件目录路径
4. 在该目录下查找请求的文件
5. 找到则返回文件（`send_from_directory）
6. 所有插件都找不到 → 404

### 3.2 哪些插件实现了 `plugin_static_path`？

**结论：当前代码库中 0 个内置插件实现 `plugin_static_path`。**

全面搜索结果：

| 插件类别 | 插件名称 | 是否实现 plugin_static_path |
|-----------|----------|---------------------------|
| conditions 插件 | `levenshtein_plugin` | ❌ 否 |
| conditions 插件 | `wordcount_plugin` | ❌ 否 |
| conditions 默认 | `default_plugin` | ❌ 否 |
| restock 插件 | `llm_restock` | ❌ 否 |
| 内置 fetcher | `builtin_requests` | ❌ 否 |
| 内置 fetcher | `builtin_playwright` | ❌ 否 |
| 内置 fetcher | `builtin_puppeteer` | ❌ 否 |
| 内置 fetcher | `builtin_webdriver_selenium` | ❌ 否 |

`plugin_static_path` 只在 [pluggy_interface.py 第 55 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/pluggy_interface.py#L55-L61) 作为 hookspec 定义存在，是为**第三方外部插件**预留的扩展接口。

### 3.3 插件注册的完整清单

当前运行时实际注册的所有 pluggy 插件：

**从 `load_plugins_from_directories()` 加载（[pluggy_interface.py 第 244-274 行）：
- `levenshtein_plugin`（conditions 插件）
- `wordcount_plugin`（conditions 插件）

**从 `register_builtin_fetchers()` 加载（[pluggy_interface.py 第 296-316 行）：
- `builtin_requests`
- `builtin_playwright`
- `builtin_puppeteer`
- `builtin_webdriver_selenium`

**从 `register_builtin_restock_plugins()` 加载（[第 318-333 行）：
- `llm_restock`

**从 setuptools entrypoints 加载**：
- 取决于安装的第三方包（默认安装无

### 3.4 plugin 组的实际安全边界

| 场景 | 是否有密码 + 未登录 | 有密码 + 已登录 | 无密码 |
|------|---------------------|-----------------|--------|
| **访问 plugin 组** | ✅ 可访问（无认证检查） | ✅ 可访问 | ✅ 可访问 |

**安全性分析：**

1. **当前无实际文件**：目前 0 个内置插件实现 `plugin_static_path`，所以任何请求 `/static/plugin/anything` 都会返回 404
2. **路径安全**：`send_from_directory` 防止路径穿越
3. **扩展风险**：如果未来安装第三方插件后，如果第三方插件的 `plugin_static_path()` 返回了包含敏感信息的目录，就会在密码保护模式下被匿名访问
4. **无认证**：plugin 组完全没有认证检查，对比 screenshot 至少还有个 `shared_diff_access` 判断

### 3.5 与其他静态资源组的对比

| 资源组 | 认证检查 | 缓存策略 | 实际内容 |
|--------|---------|---------|-----------|
| `screenshot` | 有密码+未登录+分享关闭 → 403 | `no-cache` | 监控截图 |
| `favicon` | 有密码+未登录 → 403 | `max-age=300` | 网站 favicon |
| `visual_selector_data` | 有密码+未登录 → 403 | `no-cache` | DOM 元素数据 |
| `plugin` | **无认证检查** | `max-age=3600, public` | 插件静态文件 |
| 默认（js/styles/images） | **无认证检查** | 默认 | CSS/JS/图片 |
| `flags` | **无认证检查**（独立路由 `static_flags`） | — | 国旗 SVG |

### 3.6 `plugin 资源供给的设计意图

从 [pluggy_interface.py 第 179-234 行](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/pluggy_interface.py#L179-L234) 的 `get_html_head_extras` hookspec 的文档注释中提到两种插件提供了两种静态资源提供方式：

1. **内联方式**：直接返回 HTML 字符串（适合少量 CSS/JS
2. **自定义路由方式**：插件自己注册 Flask 路由

而 `plugin_static_path` 是第三种方式——**通过统一的 `/static/plugin/<filename>` 路径提供。

设计假设：
- 插件静态文件（CSS/JS/图片等公共资源
- 不包含敏感信息
- 类似于 `plugin_static_path` 是一个 hookspec 文档中说明 "Return the path to the plugin's static files directory"

---

## 四、综合风险总结

### 4.1 装饰器顺序

| 端点 | 顺序正确 | AST 可检测 | 无密码泄露 |
|------|----------|------------|------------|
| `gc_cleanup` | ✅ 是 | ✅ 是 | 低 |
| `worker_health` | ✅ 是 | ✅ 是 | 中高（运行中 UUID 列表） |
| `queue-status` | ✅ 是 | ✅ 是 | 高（队列中全部 UUID 列表） |

三个端点的装饰器顺序目前都是正确的，AST 静态检查也都能覆盖到。主要风险在无密码部署下的信息泄露面。

### 4.2 plugin 资源组

- **现状：** 0 个内置插件实现 `plugin_static_path`，实际无文件可泄露
- **结构风险：** 第三方插件如果实现后，其静态文件在密码保护模式下也可匿名访问
- **建议：** 对 `plugin` 组增加与 `screenshot` 类似的 `shared_diff_access` 检查，或至少增加密码保护模式下默认拒绝访问

### 4.3 代码文件索引

| 文件 | 分析内容 |
|------|---------|
| [flask_app.py#L902-L983](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L902-L983) | 三个运维端点定义 |
| [auth_decorator.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/auth_decorator.py) | `login_optionally_required` 装饰器 |
| [tests/unit/test_auth_decorator_order.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/tests/unit/test_auth_decorator_order.py) | AST 静态检查实现 |
| [gc_cleanup.py](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/gc_cleanup.py) | 内存清理实现 |
| [worker_pool.py#L439-L524](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/worker_pool.py#L439-L524) | Worker 状态与健康检查 |
| [queue_handlers.py#L271-L365](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/queue_handlers.py#L271-L365) | 队列状态查询接口 |
| [flask_app.py#L823-L846](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/flask_app.py#L823-L846) | plugin 资源组实现 |
| [pluggy_interface.py#L55-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/pluggy_interface.py#L55-L61) | `plugin_static_path` hookspec 定义 |
| [pluggy_interface.py#L244-L277](file:///d:/fz/0601-1/solo-dogfeeding/code/75-changedetection.io/changedetectionio/pluggy_interface.py#L244-L277) | 插件加载逻辑 |
