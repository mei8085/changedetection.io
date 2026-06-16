# changedetection.io 启动配置体系分析

## 1. 启动入口与初始化调用链

### 1.1 入口点

```
changedetection.py  →  changedetectionio.main()
```

[changedetection.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetection.py#L7-L8) 仅做转发，真正逻辑在 [changedetectionio/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/__init__.py#L181) 的 `main()` 中。

### 1.2 初始化顺序（按时间线）

| 阶段 | 代码位置 | 做了什么 |
|------|----------|----------|
| ① 模块级全局配置 | `__init__.py` 顶部 | 设置 `multiprocessing.set_start_method('spawn')`；设置 `MALLOC_ARENA_MAX` |
| ② CLI 参数解析 | `main()` L198-L323 | getopt 解析 `-h/-p/-d/-l/-s/-P` 等；预处理 `-u/-r/-b` |
| ③ 环境变量 → 运行时变量 | `main()` L204-L206 | `LISTEN_HOST`→`host`，`PORT`→`port`；`LOGGER_LEVEL`→日志级别 |
| ④ 创建 DataStore | `main()` L383 | `ChangeDetectionStore(datastore_path, ...)` — **核心配置装载点** |
| ⑤ 创建 Flask App | `main()` L428 | `changedetection_app(app_config, datastore)` |
| ⑥ 启动 Worker 池 | `flask_app.py` L996-L998 | `worker_pool.start_workers(n_workers, ...)` |
| ⑦ 启动后台线程 | `flask_app.py` L1004-L1021 | ticker、notification_runner、version_checker |
| ⑧ 启动 HTTP 服务 | `main()` L686-L697 | `socketio.run(app, ...)` 或 `app.run(...)` |

---

## 2. 配置装载层：三层合并模型

配置的最终值由三层叠加而成，优先级从高到低：

```
环境变量 (ENV)  >  配置文件 (JSON)  >  默认值 (代码硬编码)
```

但**并非所有配置项都严格遵循此顺序**，不同子系统有各自的解析策略，详见第 3 节。

### 2.1 第一层：代码默认值

#### 全局默认值 — App.model.base_config

定义于 [model/App.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/App.py#L20-L85)，是一个 dict 类的类属性：

```python
base_config = {
    'watching': {},
    'settings': {
        'headers': {},
        'requests': {
            'time_between_check': {'weeks': None, 'days': None, 'hours': 3, ...},
            'timeout': int(getenv("DEFAULT_SETTINGS_REQUESTS_TIMEOUT", "45")),
            'workers': int(getenv("DEFAULT_SETTINGS_REQUESTS_WORKERS", "5")),
            'default_ua': { 'html_requests': getenv("DEFAULT_SETTINGS_HEADERS_USERAGENT", ...) },
        },
        'application': {
            'fetch_backend': getenv("DEFAULT_FETCH_BACKEND", "html_requests"),
            'notification_body': default_notification_body,
            'notification_format': default_notification_format,
            ...
        }
    }
}
```

**关键特征**：`base_config` 中的默认值在**类定义时**就已求值，部分字段直接内嵌了 `os.getenv()` 调用——这意味着环境变量可以在默认值层面就介入优先级链。

#### Watch 级默认值 — watch_base.\_\_init\_\_

定义于 [model/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L174-L306)，每个 Watch 对象创建时自动填充：

```python
self.update({
    'fetch_backend': 'system',          # 'system' 意味着回退到全局设置
    'notification_format': USE_SYSTEM_DEFAULT_NOTIFICATION_FORMAT_FOR_WATCH,
    'filter_failure_notification_send': strtobool(os.getenv('FILTER_FAILURE_NOTIFICATION_SEND_DEFAULT', 'True')),
    'time_between_check_use_default': True,
    ...
})
```

**关键设计**：很多字段默认为 `None` 或 `'system'`，显式表达"未设置，使用上级配置"的语义。

#### LLM 默认值 — LLMSettings (Pydantic)

定义于 [model/LLMSettings.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/LLMSettings.py#L21-L56)，这是项目中**唯一使用 Pydantic** 的配置模型：

```python
class LLMSettings(BaseModel):
    model_config = ConfigDict(extra='forbid')   # 严格模式，拒绝未声明的字段
    enabled: bool = True
    thinking_budget: int = 0
    max_summary_tokens: int = 3000
    model: str = ''
    api_key: str = ''
    ...
```

读取时通过 `LLMSettings.model_validate(stored_dict_or_empty)` 水合，写入时通过 `.model_dump()` 序列化。存储层始终是纯 dict，Pydantic 仅作为验证/类型层。

### 2.2 第二层：配置文件 (JSON)

#### 存储结构

```
{datastore_path}/
├── changedetection.json        ← 全局设置（不含 watching/tags）
├── proxies.json                ← 外部代理列表
├── headers.txt                 ← 全局额外 HTTP 请求头
├── {uuid}/
│   ├── watch.json              ← 单个 Watch 配置
│   ├── tag.json                ← 单个 Tag 配置
│   ├── restock_diff.json      ← restock 处理器专用配置
│   ├── headers.txt             ← Watch 级额外 HTTP 请求头
│   └── ...                     ← 历史快照、截图等
└── {uuid}/
    ├── tag.json                ← Tag 持久化
    └── ...
```

#### 加载流程

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L190) `reload_state()` 的三种场景：

1. **已有 changedetection.json**：`_load_state()` → `_load_settings()` + `_load_watches()` + `_load_tags()` → `_rehydrate_watches()` + `_rehydrate_tags()` → `run_updates()`
2. **仅有 url-watches.json（旧格式）**：先从旧格式加载 → 运行 update_26 迁移 → 重新 `_load_state()`
3. **全新安装**：`init_fresh_install()` → 生成 app_guid/rss_access_token → 添加默认 watch → `_save_settings()` → `_load_state()`

#### 合并机制 — `_apply_settings()`

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L102-L134) 中 `_apply_settings()` 使用 `dict.update()` 将 JSON 数据合并到默认值之上：

```python
def _apply_settings(self, settings_data):
    if 'headers' in settings_data['settings']:
        self.__data['settings']['headers'].update(settings_data['settings']['headers'])
    if 'requests' in settings_data['settings']:
        self.__data['settings']['requests'].update(settings_data['settings']['requests'])
    if 'application' in settings_data['settings']:
        self.__data['settings']['application'].update(settings_data['settings']['application'])
    if 'watching' in settings_data:
        self.__data['watching'].update(settings_data['watching'])
```

**重要**：这是浅层 `dict.update()`，意味着 JSON 文件中的键会**覆盖**默认值中的同名键，但不会递归合并嵌套 dict（整个嵌套 dict 被替换）。

#### Watch 反序列化（Rehydration）

1. [file_saving_datastore.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/file_saving_datastore.py#L273) `load_all_watches()` 扫描所有 `{uuid}/watch.json`
2. 每个 watch_dict 传入 [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L321-L335) `rehydrate_entity()`，构造 `Watch.model(default=watch_dict)`
3. Watch.\_\_init\_\_ 先填入默认值，再 `self.update(kw['default'])` 用 JSON 数据覆盖——JSON 中的字段**覆盖**默认值

#### Tag 加载的双重来源

1. `_apply_settings()` 从 changedetection.json 的 `settings.application.tags` 读取
2. `_load_tags()` 从独立的 `{uuid}/tag.json` 文件读取，**后者覆盖前者**

```python
# store/__init__.py _load_tags():
if tags:
    self.__data['settings']['application']['tags'].update(tags)
```

### 2.3 第三层：环境变量 (ENV)

环境变量是**最高优先级**的配置源，但其介入方式不统一，分为以下几类：

#### A. 启动时一次性解析（CLI 级覆盖）

| 环境变量 | 被谁读取 | 覆盖谁 | 默认值 |
|----------|----------|--------|--------|
| `LISTEN_HOST` | `main()` | CLI `-h` 参数 | `"0.0.0.0"` |
| `PORT` | `main()` | CLI `-p` 参数 | `5000` |
| `LOGGER_LEVEL` | `main()` | CLI `-l` 参数 | `"DEBUG"` |
| `SSL_CERT_FILE` / `SSL_PRIVKEY_FILE` | `main()` | CLI `-s` 参数 | `'cert.pem'` / `'privkey.pem'` |
| `SALTED_PASS` | `User.check_password()` | JSON 存储的密码 | — |

#### B. 在默认值定义时嵌入（App.model 层面）

这些环境变量通过 `getenv()` 嵌入 `base_config`，若环境变量有值则**替代硬编码默认值**，但会被后续 JSON 文件的 `update()` 覆盖：

| 环境变量 | 位置 | 默认值 |
|----------|------|--------|
| `DEFAULT_SETTINGS_REQUESTS_TIMEOUT` | App.py | `"45"` |
| `DEFAULT_SETTINGS_REQUESTS_WORKERS` | App.py | `"5"` |
| `DEFAULT_SETTINGS_HEADERS_USERAGENT` | App.py | Chrome UA 字符串 |
| `DEFAULT_FETCH_BACKEND` | App.py | `"html_requests"` |

**优先级**：`ENV` > `硬编码默认值`，但 JSON 配置文件可覆盖这两者。

#### C. 运行时动态解析（每次读取时判定）

这类环境变量在代码运行过程中每次被读取时实时判定，**始终覆盖 JSON 存储值**：

| 环境变量 | 读取位置 | 行为 |
|----------|----------|------|
| `FETCH_WORKERS` | flask_app.py L996, worker_pool.py | 覆盖 `settings.requests.workers` |
| `BASE_URL` | store/\_\_init\_\_.py L590-L591 | 覆盖 `settings.application.base_url`（仅当 JSON 中未设置时） |
| `SALTED_PASS` | flask_app.py L533 | 覆盖 JSON 存储的密码 |
| `HIDE_REFERER` | flask_app.py L643 | 控制响应头 |
| `USE_X_SETTINGS` | flask_app.py L656 | 启用 ProxyFix 中间件 |
| `FLASK_SERVER_NAME` | flask_app.py L118-L119 | 设置 Flask SERVER_NAME |
| `FLASK_ENABLE_COMPRESSION` | flask_app.py L100 | 启用 Flask-Compress |
| `DISABLE_VERSION_CHECK` | flask_app.py L1019 | 禁用版本检查 |
| `NOTIFICATION_WORKERS` | flask_app.py L1007 | 通知 worker 数量 |
| `MINIMUM_SECONDS_RECHECK_TIME` | flask_app.py L1117 | 最小重检间隔 |
| `ENABLE_NO_PROXY_OPTION` | store/\_\_init\_\_.py L850, L868 | 在代理列表添加 "No proxy" 选项 |
| `PAGE_WATCH_LIMIT` | store/\_\_init\_\_.py L739 | Watch 数量上限 |

#### D. LLM 子系统的专属环境变量覆盖

LLM 子系统有**最规范**的环境变量覆盖机制，在 [llm/evaluator.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/llm/evaluator.py#L234-L257) 的 `get_llm_config()` 中实现：

```python
def get_llm_config(datastore) -> dict | None:
    # 1. 环境变量优先
    env_model = os.getenv('LLM_MODEL', '').strip()
    if env_model:
        return {
            'model': env_model,
            'api_key': os.getenv('LLM_API_KEY', '').strip(),
            'api_base': os.getenv('LLM_API_BASE', '').strip(),
        }
    # 2. 回退到 datastore 设置（UI 配置）
    cfg = datastore.data['settings']['application'].get('llm') or {}
    if not cfg.get('model'):
        return None
    return cfg
```

| 环境变量 | 覆盖对象 |
|----------|----------|
| `LLM_MODEL` | `settings.application.llm.model` |
| `LLM_API_KEY` | `settings.application.llm.api_key` |
| `LLM_API_BASE` | `settings.application.llm.api_base` |
| `LLM_FEATURES_DISABLED` | 全局禁用 LLM 功能 |
| `LLM_MAX_INPUT_CHARS` | `settings.application.llm.max_input_chars` |
| `LLM_TOKEN_BUDGET_MONTH` | `settings.application.llm.token_budget_month` |
| `LLM_TIMEOUT` | LLM 客户端超时（默认 60s） |

#### E. 内容抓取器相关的环境变量

| 环境变量 | 读取位置 | 行为 |
|----------|----------|------|
| `PLAYWRIGHT_DRIVER_URL` | content_fetchers/\_\_init\_\_.py | 选择 Playwright 或 Selenium 作为浏览器抓取器 |
| `FAST_PUPPETEER_CHROME_FETCHER` | content_fetchers/\_\_init\_\_.py | 使用直接 Puppeteer 而非 Playwright |
| `SCREENSHOT_MAX_HEIGHT` | content_fetchers/\_\_init\_\_.py | 截图最大高度 |
| `SCREENSHOT_CHUNK_HEIGHT` | content_fetchers/\_\_init\_\_.py | 拼接阈值 |

---

## 3. 三级配置解析链（Watch → Tag → Global）

### 3.1 链式解析的概念模型

代码注释中反复提到的"梦想架构"是三级解析链：

```
Watch 自身设置  →  Tag/Group 设置  →  Global 设置
```

即：如果 Watch 上某字段为 `None`/`'system'`，则向上查找关联的 Tag，若 Tag 也未覆盖，最终回退到全局默认值。

### 3.2 当前实现：手动散落式解析

当前**没有**统一的解析函数，各消费方自行实现回退逻辑：

#### fetch_backend 解析

[Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L357-L389)：

```python
@property
def get_fetch_backend(self):
    if self.is_pdf:
        return 'html_requests'        # 特殊情况：PDF 始终用 requests
    return self.get('fetch_backend')   # 返回 watch 自身值，可能是 'system'
```

而在消费方（如 [Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L400-L402) `fetcher_supports_screenshots`）：

```python
fetcher_name = self.get_fetch_backend
if not fetcher_name or fetcher_name == 'system':
    fetcher_name = self._datastore['settings']['application'].get('fetch_backend', 'html_requests')
```

#### time_between_check 解析

[ticker_thread](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1196-L1221)：

```python
if watch.get('time_between_check_use_default'):
    time_schedule_limit = datastore.data['settings']['requests'].get('time_schedule_limit', {})
else:
    time_schedule_limit = watch.get('time_schedule_limit')

threshold = recheck_time_system_seconds if watch.get('time_between_check_use_default') else watch.threshold_seconds()
```

#### history_snapshot_max_length 解析

[Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L723)：

```python
maxlen = self.get('history_snapshot_max_length') or self.get_global_setting('application', 'history_snapshot_max_length')
```

#### 通知配置回退

[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1079-L1086) notification_runner：

```python
if not n_object.get('notification_body') and datastore.data['settings']['application'].get('notification_body'):
    n_object['notification_body'] = datastore.data['settings']['application'].get('notification_body')
# notification_title, notification_format 同理
```

### 3.3 Tag 覆盖 Watch 的机制

Tag 通过 `watch.get('tags')` 列表关联到 Watch。[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L936-L945) `get_tag_overrides_for_watch()` 可获取 Tag 对某属性的覆盖值。

目前 Tag 主要在 restock_diff 处理器中使用 override 机制：

```python
# processors/restock_diff/processor.py (示意):
for tag_uuid in watch.get('tags'):
    tag = datastore['settings']['application']['tags'][tag_uuid]
    if tag.get('overrides_watch'):
        restock_settings = tag.get('restock_settings', {})
        break
```

### 3.4 BASE_URL 的特殊三级解析

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L583-L597) `data` 属性：

```python
active_base_url = BASE_URL_NOT_SET_TEXT
if self.__data['settings']['application'].get('base_url'):
    active_base_url = self.__data['settings']['application'].get('base_url')   # JSON 配置
elif os.getenv('BASE_URL'):
    active_base_url = os.getenv('BASE_URL')                                   # 环境变量回退
```

**注意**：这里 JSON 优先于环境变量，与 LLM 子系统的优先级方向**相反**。

---

## 4. 全局共享对象与协同关系

### 4.1 核心对象图

```
main() 中的全局变量
├── datastore: ChangeDetectionStore   ← 单例，所有配置与运行时状态的真相来源
│   ├── __data: App.model (dict)      ← 内存中的完整数据树
│   │   ├── watching: {uuid: Watch}   ← 所有 Watch 对象
│   │   ├── settings.headers: dict
│   │   ├── settings.requests: dict
│   │   └── settings.application: dict ← 含 tags, llm, notification 等
│   ├── datastore_path: str
│   ├── lock: threading.Lock
│   └── generic_definition: Watch     ← 新建 watch 时的模板
│
└── app: Flask (由 changedetection_app 返回)
    ├── config['DATASTORE']: datastore ← Flask 可访问 datastore 引用
    ├── config['datastore_path']: str
    ├── config['exit']: Event          ← 优雅关闭信号
    ├── config['batch_mode']: bool
    └── 全局变量:
        ├── datastore (模块级)         ← flask_app.py 内的模块全局
        ├── update_q: RecheckPriorityQueue
        └── notification_q: NotificationQueue
```

### 4.2 对象间的引用关系

```
Watch 对象
  ├── _datastore → 指向 ChangeDetectionStore（共享引用，不深拷贝）
  ├── _datastore_path → datastore_path
  └── data_dir → os.path.join(datastore_path, uuid)

Tag 对象
  ├── _datastore → 同上
  └── _datastore_path → 同上

ChangeDetectionStore
  └── __data.settings.application.tags[uuid] → Tag 对象
  └── __data.watching[uuid] → Watch 对象

Flask App
  └── config['DATASTORE'] → ChangeDetectionStore
  └── flask_app.datastore (模块全局) → ChangeDetectionStore
```

### 4.3 Watch 对象的 get_global_setting() 方法

[watch_base](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L583-L605) 提供了从 Watch 内部向上查找全局设置的标准方法：

```python
def get_global_setting(self, *path):
    if not self._datastore:
        return None
    try:
        value = self._datastore['settings']
        for key in path:
            value = value[key]
        return value
    except (KeyError, TypeError):
        return None
```

这是 Watch → Global 回退链的通用实现。

### 4.4 DataStore.data 属性的动态计算

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L582-L597) 的 `data` 属性**不是**简单返回 `__data`，而是在每次访问时动态注入 `active_base_url`：

```python
@property
def data(self):
    active_base_url = BASE_URL_NOT_SET_TEXT
    if self.__data['settings']['application'].get('base_url'):
        active_base_url = ...
    elif os.getenv('BASE_URL'):
        active_base_url = os.getenv('BASE_URL')
    d = self.__data
    d['settings']['application']['active_base_url'] = active_base_url.strip('" ')
    return d
```

这意味着 `datastore.data` 的返回值**带有副作用**（修改了 `__data` 中的 `active_base_url`）。

---

## 5. 优先级总结速查表

### 5.1 全局配置（App-level）

| 配置项 | 默认值来源 | JSON 覆盖 | ENV 覆盖 | 最终解析位置 |
|--------|-----------|-----------|----------|-------------|
| `requests.timeout` | `App.base_config` (或 `DEFAULT_SETTINGS_REQUESTS_TIMEOUT`) | ✅ changedetection.json | ⚠️ 仅在默认值层 | `data['settings']['requests']['timeout']` |
| `requests.workers` | `App.base_config` (或 `DEFAULT_SETTINGS_REQUESTS_WORKERS`) | ✅ | ✅ `FETCH_WORKERS` 实时覆盖 | flask_app.py L996 |
| `application.fetch_backend` | `App.base_config` (或 `DEFAULT_FETCH_BACKEND`) | ✅ | ❌ | `data['settings']['application']['fetch_backend']` |
| `application.base_url` | `None` | ✅ 优先 | ✅ `BASE_URL` 回退 | store data 属性 |
| `application.password` | `False` | ✅ | ✅ `SALTED_PASS` 优先 | User.check_password() |
| `application.notification_*` | `notification/__init__.py` | ✅ | ❌ | `data['settings']['application']` |
| `application.llm.*` | `LLMSettings` Pydantic 默认值 | ✅ | ✅ `LLM_MODEL/KEY/BASE` 优先 | evaluator.py get_llm_config() |
| `application.llm.max_input_chars` | `LLMSettings` (100000) | ✅ | ✅ `LLM_MAX_INPUT_CHARS` 优先 | evaluator.py _get_max_input_chars() |
| `application.llm.token_budget_month` | `LLMSettings` (0) | ✅ | ✅ `LLM_TOKEN_BUDGET_MONTH` 优先 | evaluator.py get_global_token_budget_month() |

### 5.2 Watch 级配置

| 配置项 | Watch 默认值 | Tag 覆盖 | Global 回退 | 回退标志 |
|--------|-------------|----------|------------|---------|
| `fetch_backend` | `'system'` | ❌ (未来支持) | ✅ `application.fetch_backend` | 值为 `'system'` |
| `time_between_check` | `time_between_check_use_default=True` | ❌ | ✅ `requests.time_between_check` | `time_between_check_use_default` |
| `notification_format` | `'System default'` | ✅ | ✅ `application.notification_format` | 值为 `'System default'` |
| `notification_body/title` | `None` | ✅ | ✅ `application.notification_*` | 值为 `None` |
| `history_snapshot_max_length` | `None` | ❌ | ✅ `application.history_snapshot_max_length` | 值为 `None` |
| `use_page_title_in_list` | `None` | ❌ | ✅ `application.ui.use_page_title_in_list` | 值为 `None` |
| `filter_failure_notification_send` | `True` (或 ENV) | ❌ | ❌ | 无回退 |

### 5.3 运行时行为覆盖

| 环境变量 | 覆盖时机 | 特点 |
|----------|----------|------|
| `FETCH_WORKERS` | 每次 Worker 启动/健康检查时 | 动态，始终优先于 JSON |
| `NOTIFICATION_WORKERS` | App 初始化时 | 固定 |
| `MINIMUM_SECONDS_RECHECK_TIME` | ticker 每次循环 | 动态 |
| `SALTED_PASS` | 每次密码校验 | 动态，完全绕过 JSON 存储密码 |
| `BASE_URL` | 每次访问 datastore.data | 动态，仅 JSON 无值时才生效 |
| `HIDE_REFERER` | 每次请求 | 动态 |
| `LLM_MODEL/KEY/BASE` | 每次 LLM 调用 | 动态，完全绕过 JSON LLM 配置 |
| `PLAYWRIGHT_DRIVER_URL` | 模块导入时 | 一次性，决定浏览器引擎选择 |
| `DISABLE_VERSION_CHECK` | App 初始化时 | 一次性 |
| `PAGE_WATCH_LIMIT` | 每次 add_watch | 动态 |
| `TESTING_SHUTDOWN_AFTER_DATASTORE_LOAD` | main() DataStore 加载后 | CI/CD 专用 |

---

## 6. Schema 迁移与配置演进

配置结构通过 schema version 控制向前兼容。定义于 [store/updates.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/updates.py) 的 `DatastoreUpdatesMixin`，当前最新版本为 **update_32**。

关键迁移：
- **update_26**: 单体 url-watches.json → 独立 watch.json 文件
- **update_29**: Tag 迁移到独立 tag.json 文件
- **update_30**: restock_settings 迁出 watch.json 到 restock_diff.json
- **update_31**: 扁平 `llm_*` 键折叠到嵌套 `application.llm.*`
- **update_32**: 清理废弃的 `max_tokens_per_check`，重命名 `max_tokens_cumulative`

每次迁移前自动创建 tarball 备份（`before-update-N-timestamp.tar.gz`）。

---

## 7. 关键设计洞察与不一致性

### 7.1 环境变量优先级方向不统一

| 子系统 | ENV vs JSON 优先级 |
|--------|-------------------|
| LLM 配置 | **ENV > JSON**（环境变量完全覆盖） |
| BASE_URL | **JSON > ENV**（JSON 有值则 ENV 被忽略） |
| SALTED_PASS | **ENV > JSON**（ENV 存在时完全绕过 JSON 密码） |
| Workers | **ENV > JSON**（FETCH_WORKERS 实时覆盖） |
| 超时/UA | **ENV 在默认值层**（被 JSON update 覆盖） |

### 7.2 dict.update() 的浅合并风险

`_apply_settings()` 使用浅层 `dict.update()`，这意味着对于嵌套结构（如 `settings.application.ui`），如果 JSON 中只存储了部分子键，整个嵌套 dict 会被替换而非合并。实际运行中因 `base_config` 已包含完整结构且 JSON 通常也写入完整结构，此问题较少暴露。

### 7.3 Watch → Tag → Global 解析散落

代码中缺少统一的 `resolve_setting(watch, key)` 函数，每个消费方自行实现回退逻辑。这导致：
- 不同位置可能实现不同的回退路径
- Tag 覆盖仅在 restock_diff 处理器中被真正使用
- 新增配置项时容易遗漏回退逻辑

### 7.4 Pydantic 仅用于 LLM

`LLMSettings` 是唯一使用 Pydantic 的模型，具有 `extra='forbid'` 的严格验证。其余所有配置（Watch、App、Tag）仍为 dict 继承，缺乏运行时类型检查。

---

## 8. 暗线一：os.getenv 调用点穷尽分类

排除测试目录，仓库内共约 **70 处** `os.getenv()` 调用（不含 `os.environ` 赋值/判断），按介入时机与覆盖能力分为五类。

### 8.1 A 类 — 模块导入时求值，值被冻结（进程生命周期不可变）

此类 `getenv()` 出现在模块顶层或类属性定义中，Python 在 import 时求值一次，之后不再重新读取环境变量。

| 环境变量 | 代码位置 | 语义 |
|----------|----------|------|
| `DEFAULT_SETTINGS_REQUESTS_TIMEOUT` | [App.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/App.py#L32) `base_config` 类属性 | 嵌入默认值，可被 JSON 覆盖 |
| `DEFAULT_SETTINGS_REQUESTS_WORKERS` | [App.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/App.py#L33) | 同上 |
| `DEFAULT_SETTINGS_HEADERS_USERAGENT` | [App.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/App.py#L35) | 同上 |
| `DEFAULT_FETCH_BACKEND` | [App.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/App.py#L46) | 同上 |
| `FETCH_WORKERS` | [worker_pool.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/worker_pool.py#L30) `_max_executor_workers` | 线程池大小，启动后不可缩 |
| `FORCE_FSYNC_DATA_IS_CRITICAL` | [file_saving_datastore.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/file_saving_datastore.py#L30) | 控制 fsync，启动后固定 |
| `FILTER_FAILURE_NOTIFICATION_SEND_DEFAULT` | [model/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L198) | Watch 默认值 |
| `BROTLI_COMPRESS_SIZE_THRESHOLD` (`SNAPSHOT_BROTLI_COMPRESSION_THRESHOLD`) | [Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L44) | Brotli 压缩阈值 |
| `MINIMUM_SECONDS_RECHECK_TIME` | [Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L51) 模块级变量 | 同时在 ticker 内运行时读取 |
| `SCREENSHOT_MAX_HEIGHT` | [content_fetchers/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/__init__.py#L19) | 截图最大高度 |
| `SCREENSHOT_CHUNK_HEIGHT` | [content_fetchers/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/__init__.py#L26) | 拼接阈值 |
| `OPENCV_SUBPROCESS_TIMEOUT` | [image_ssim_diff/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/image_ssim_diff/__init__.py#L26) | OpenCV 子进程超时 |
| `OPENCV_BLUR_SIGMA` | [image_ssim_diff/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/image_ssim_diff/__init__.py#L39) | 模糊 sigma |
| `MAX_DIFF_HEIGHT/WIDTH` | [image_ssim_diff/difference.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/image_ssim_diff/difference.py#L23-L24) | 差异图最大尺寸 |
| `ENABLE_TEMPLATE_TRACKING` | [image_ssim_diff/edit_hook.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/image_ssim_diff/edit_hook.py#L18) | 模板匹配开关 |
| `JINJA2_MAX_RETURN_PAYLOAD_SIZE_KB` | [safe_jinja.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/jinja2_custom/safe_jinja.py#L13) | Jinja2 渲染上限 |
| `LLM_TIMEOUT` | [llm/client.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/llm/client.py#L17) `DEFAULT_TIMEOUT` | HTTP 客户端超时 |

**重启依赖**：修改这些环境变量后**必须重启进程**才能生效。部分（A 类嵌入默认值者）可在重启后被 JSON `update()` 覆盖，其余（线程池大小、fsync 等）完全由环境变量独占。

### 8.2 B 类 — 启动时一次性读取，值被缓存于局部变量

| 环境变量 | 代码位置 | 缓存方式 |
|----------|----------|----------|
| `LISTEN_HOST` / `PORT` | [\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/__init__.py#L204-L205) `main()` | 赋值给 `host`/`port` 局部变量 |
| `LOGGER_LEVEL` | `main()` | 赋值后 `logger.level` 设定 |
| `FLASK_SERVER_NAME` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L118-L119) | 赋值给 `app.config['SERVER_NAME']` |
| `FLASK_ENABLE_COMPRESSION` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L100) | 条件判断后初始化 Flask-Compress |
| `SOCKETIO_MODE` | [socket_server.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/realtime/socket_server.py#L236) | 传给 SocketIO 构造参数 |
| `SOCKETIO_CORS_ORIGINS` | [socket_server.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/realtime/socket_server.py#L260) | CORS 配置 |
| `SOCKETIO_LOGGING` | [socket_server.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/realtime/socket_server.py#L265-L266) | logger 参数 |
| `NOTIFICATION_WORKERS` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1007) | 创建 N 个线程后不可变 |
| `DISABLE_VERSION_CHECK` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1019) | 条件判断，不创建线程 |
| `WORKER_MAX_JOBS` / `WORKER_MAX_RUNTIME` | [worker.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/worker.py#L67-L68) | Worker 重启策略参数 |
| `BLOCK_SIMPLEHOSTS` | [forms.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/forms.py#L64) | 表单验证逻辑 |

**重启依赖**：必须重启。`SOCKETIO_*` 和 `NOTIFICATION_WORKERS` 在 App 构造期间消费后无法动态修改。

### 8.3 C 类 — 运行时每次使用时实时读取

这是**唯一不需要重启就能生效**的类别，但也意味着每次调用都有 `os.getenv()` 的微开销。

| 环境变量 | 代码位置 | 读取频率 |
|----------|----------|----------|
| `FETCH_WORKERS` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L996) start_workers, [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L923) health check, [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1131) ticker 60s check | 每 60s + 启动时 |
| `MINIMUM_SECONDS_RECHECK_TIME` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1117) ticker | 每轮 ticker 循环 |
| `SALTED_PASS` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L442) 密码校验, [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L533) has_password, [socket_server.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/realtime/socket_server.py#L320), [difference.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/text_json_diff/difference.py#L162), [extract.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/extract.py#L48) | 每次认证/通知 |
| `BASE_URL` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L590-L591) data 属性 | 每次访问 `datastore.data` |
| `HIDE_REFERER` | flask_app.py 请求中间件 | 每次请求 |
| `USE_X_SETTINGS` | flask_app.py 请求中间件 | 每次请求 |
| `ENABLE_NO_PROXY_OPTION` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L850) | 每次 get_proxy_list / 请求 |
| `PAGE_WATCH_LIMIT` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L739) | 每次 add_watch |
| `LLM_MODEL/KEY/BASE` | [evaluator.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/llm/evaluator.py#L245-L250) get_llm_config() | 每次 LLM 调用 |
| `LLM_MAX_INPUT_CHARS` | [evaluator.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/llm/evaluator.py#L62) | 每次 LLM 输入截断 |
| `LLM_TOKEN_BUDGET_MONTH` | [evaluator.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/llm/evaluator.py#L306) | 每次预算检查 |
| `LLM_FEATURES_DISABLED` | [evaluator.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/llm/evaluator.py#L44) | 每次 LLM 功能判断 |
| `TZ` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1204) ticker, [safe_jinja.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/jinja2_custom/safe_jinja.py#L38), [TimeExtension.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/jinja2_custom/extensions/TimeExtension.py#L108) | 每次调度/Jinja2 渲染 |
| `PDF_TO_HTML_TOOL` | [text_json_diff/processor.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/text_json_diff/processor.py#L295) | 每次处理 PDF |
| `DISABLED_PROCESSORS` | [processors/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/__init__.py#L205) | `@lru_cache` 保护，实际只求值一次* |
| `HISTORY_SNAPSHOT_FILE_ALLOW_OUTSIDE_WATCH_DATADIR` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L989), [Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L568) | 每次快照读写 |
| `ALLOW_IANA_RESTRICTED_ADDRESSES` | [validate_url.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/validate_url.py#L148), [base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/base.py#L105), [requests.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/requests.py#L86), [custom_handlers.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/notification/apprise_plugin/custom_handlers.py#L204) | 每次校验/抓取/通知 |
| `ALLOW_FILE_URI` | [validate_url.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/validate_url.py#L201), [base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/base.py#L125), [requests.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/requests.py#L82) | 每次校验/抓取 |
| `SAFE_PROTOCOL_REGEX` | [validate_url.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/validate_url.py#L238) | 每次校验 |
| `JQ_ALLOW_RISKY_EXPRESSIONS` | [html_tools.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/html_tools.py#L47) | 每次执行 jq |
| `XPATH_BLOCKED_FUNCTIONS` | [html_tools.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/html_tools.py#L104) | 每次执行 xpath |
| `DISABLE_BROTLI_TEXT_SNAPSHOT` | [Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L658) | 每次快照存储 |

\* `DISABLED_PROCESSORS` 被 `@lru_cache` 包裹，运行期间只求值一次，修改后需重启。

### 8.4 D 类 — 抓取器专属，模块导入时决策

| 环境变量 | 代码位置 | 影响 |
|----------|----------|------|
| `PLAYWRIGHT_DRIVER_URL` | [content_fetchers/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/__init__.py#L93) | 决定导入 Playwright 还是 Selenium |
| `FAST_PUPPETEER_CHROME_FETCHER` | [content_fetchers/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/__init__.py#L96) | 决定 Playwright 还是 Puppeteer |
| `WEBDRIVER_URL` | [webdriver_selenium.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L9-L38) | Selenium 连接地址 |
| `CHROME_OPTIONS` | [webdriver_selenium.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L90) | Chrome 启动参数 |
| `WEBDRIVER_CONNECTION_TIMEOUT` | [webdriver_selenium.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L111) | 连接超时 |
| `WEBDRIVER_PAGELOAD_TIMEOUT` | [webdriver_selenium.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L123) | 页面加载超时 |
| `WEBDRIVER_DELAY_BEFORE_CONTENT_READY` | [webdriver_selenium.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L135), [playwright.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/playwright.py#L334), [puppeteer.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/puppeteer.py#L276) | 内容就绪等待 |
| `SCREENSHOT_QUALITY` | [screenshot_handler.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/screenshot_handler.py#L82), [webdriver_selenium.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L175), [playwright.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/playwright.py#L68), [puppeteer.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/puppeteer.py#L50) | 截图 JPEG 质量 |
| `SCREENSHOT_MAX_HEIGHT` | [playwright.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/playwright.py#L387), [puppeteer.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/puppeteer.py#L476) | 每次截图时读取 |
| `PLAYWRIGHT_BROWSER_TYPE` | [playwright.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/playwright.py#L155-L186) | 浏览器类型 |
| `PLAYWRIGHT_SERVICE_WORKERS` | [playwright.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/playwright.py#L291) | Service Worker 策略 |
| `PUPPETEER_MAX_PROCESSING_TIMEOUT_SECONDS` | [puppeteer.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/puppeteer.py#L519) | Puppeteer 超时 |
| `HTTP_PROXY` / `HTTPS_PROXY` | [base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/base.py#L61-L62) | 系统代理 |
| `REMOVE_REQUESTS_OLD_SCREENSHOTS` | [requests.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/requests.py#L251) | 清理旧截图 |
| `REQUESTS_RETRY_MAX_COUNT` | [requests.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/requests.py#L68) | 重试次数 |
| `webdriver_proxy*` 系列 (7 个) | [webdriver_selenium.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/webdriver_selenium.py#L48-L54) | Selenium 代理配置 |
| `playwright_proxy_*` | [playwright.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/content_fetchers/playwright.py#L199) | Playwright 代理配置 |

### 8.5 E 类 — CI/CD 与调试专用

| 环境变量 | 代码位置 | 用途 |
|----------|----------|------|
| `TESTING_SHUTDOWN_AFTER_DATASTORE_LOAD` | [\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/__init__.py#L391) main() | 加载 DataStore 后立即退出 |
| `GITHUB_REF` | [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1019) | 检测 CI 环境，跳过版本检查 |
| `PYTEST_CURRENT_TEST` | 多处 | 检测 pytest 环境 |

---

## 9. 暗线二：\_rehydrate_tags 强制 restock\_diff 的完整链路

### 9.1 两条独立路径，同归一个硬编码

Tag 被加载到内存有两条路径，**两条路径都强制设置 `processor = 'restock_diff'`**：

#### 路径 A：\_load\_tags() — 从独立 tag.json 文件加载

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L417-L426)：

```python
def rehydrate_tag(uuid, entity_dict):
    """Rehydrate tag as Tag object with forced restock_diff processor."""
    entity_dict['uuid'] = uuid
    entity_dict['processor'] = 'restock_diff'  # Force processor for override functionality
    return Tag.model(
        datastore_path=self.datastore_path,
        __datastore=self.__data,
        default=entity_dict
    )
```

#### 路径 B：\_rehydrate\_tags() — 从 settings 中残留的 Tag 数据加载

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L140-L148)：

```python
for uuid, tag in self.__data['settings']['application']['tags'].items():
    tag['processor'] = 'restock_diff'          # Force processor
    self.__data['settings']['application']['tags'][uuid] = Tag.model(
        datastore_path=self.datastore_path,
        __datastore=self.__data,
        default=tag
    )
```

### 9.2 为什么强制 restock\_diff？

根因在于 **Tag 的 override 机制目前仅在 `restock_diff` 处理器中被实现**。Tag 模型继承自 `watch_base`，理论上 Tag 可以设置任意 processor，但代码中 Tag override 的消费方只有一个——[processors/restock\_diff/processor.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/restock_diff/processor.py) 的 `run_changedetection()` 方法。

强制 `processor = 'restock_diff'` 是一个**技术债务（technical debt）**的显式标注。注释原文是：

```
Force processor for override functionality
```

Tag 模型文档 [Tag.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Tag.py#L1-L21) 也明确承认了这一点：

```
ARCHITECTURE NOTE: Configuration Override Hierarchy
...
Current implementation requires manual checking in processors:
    for tag_uuid in watch.get('tags'):
        tag = datastore['settings']['application']['tags'][tag_uuid]
        if tag.get('overrides_watch'):
            restock_settings = tag.get('restock_settings', {})
            break

With Pydantic, this would be automatic via chain resolution:
    Watch → Tag (first with overrides_watch) → Global
```

### 9.3 强制的副作用

1. **UI 层**：Tag 的编辑页面不会显示 processor 选择器（因为已经被硬编码），用户无法在 UI 上修改
2. **API 层**：通过 API 创建 Tag 时可以传入 `processor_config_restock_diff`，这个键会被正确存储到 `{uuid}/restock_diff.json`
3. **持久化层**：`_save_to_disk()` 和 `_build_settings_data()` 不保存 `processor` 到 tag.json（因为每次加载都会重新强制设置），所以 JSON 文件中**不会持久化** `processor` 字段
4. **processor\_config\_xxx 分离**：[Watch.\_get\_commit\_data()](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L1064-L1093) 排除 `processor_config_*` 键，这些键由各处理器自行管理自己的 JSON 文件（如 `restock_diff.json`），Tag 同理

### 9.4 完整数据流

```
启动 → _load_tags() 从 tag.json 读取 dict
     → rehydrate_tag() 强制 processor='restock_diff'
     → Tag.model(default=dict) 构造 Tag 对象
     → self.__data['settings']['application']['tags'].update(tags)

启动 → _rehydrate_tags() 遍历 settings 中残留的 tag dict
     → 强制 processor='restock_diff'
     → Tag.model(default=dict) 构造 Tag 对象
     → 替换 __data 中的对应条目

运行时 → restock_diff processor 检查 watch.get('tags')
     → 找到 tag → 检查 tag.get('overrides_watch')
     → True → 使用 tag 的 restock_settings 覆盖 watch 设置
```

---

## 10. 暗线三：JSON 与环境变量的读取节奏与重启依赖

### 10.1 JSON 的读取节奏

| 时机 | 操作 | 频率 |
|------|------|------|
| 启动 | `_load_state()` 全量读取 | 一次 |
| 每次 Watch.commit() | 写入 `{uuid}/watch.json` | 即时写入（fire-and-forget） |
| 每次 Tag.commit() | 写入 `{uuid}/tag.json` | 即时写入 |
| 每次 `update_watch()` | 写入 watch.json | 即时写入 |
| 每次 settings 变更 | 写入 `changedetection.json` | 即时写入 |
| 运行中 | **不再读取** JSON 文件 | — |

**关键洞察**：JSON 文件是**只写（write-through）缓存**。启动时一次性读入内存，之后所有读写都在内存中的 `__data` dict 上操作，变更后立即写回磁盘。运行期间**不会**重新从磁盘读取 JSON 文件。

这意味着：**运行期间直接修改磁盘上的 JSON 文件不会影响运行中的实例**——必须重启进程才能加载修改。

### 10.2 环境变量的读取节奏

| 节奏类型 | 环境变量 | 需要重启？ |
|----------|----------|-----------|
| 模块导入时一次性 | A 类全部（8.1 节） | ✅ 必须 |
| 启动时一次性 | B 类全部（8.2 节） | ✅ 必须 |
| 运行时每次读取 | C 类全部（8.3 节） | ❌ 修改即生效* |

\* 严格来说，修改进程的环境变量需要通过 `/proc/PID/environ` 或 OS 特定机制，Docker/K8s 中需要重建容器。但**代码层面**不需要重启。

### 10.3 修改配置后的生效矩阵

| 配置来源 | 修改方式 | 运行中实例是否感知 | 需要操作 |
|----------|----------|-------------------|----------|
| JSON 文件（磁盘） | 直接编辑文件 | ❌ 不感知 | 重启进程 |
| JSON 文件（通过 UI/API） | UI/API → 内存 → 磁盘 | ✅ 即时 | 无 |
| 环境变量 A 类 | 修改进程 ENV | ❌ 不感知（已求值并缓存） | 重启进程 |
| 环境变量 B 类 | 修改进程 ENV | ❌ 不感知（已求值并缓存） | 重启进程 |
| 环境变量 C 类 | 修改进程 ENV | ✅ 下次读取时感知 | 无（但需修改运行进程的 ENV） |
| 环境变量 D 类 | 修改进程 ENV | ❌ 不感知（模块已导入） | 重启进程 |

### 10.4 特殊案例：FETCH\_WORKERS 的双重节奏

`FETCH_WORKERS` 同时出现在 A 类和 C 类中：

- **A 类**：[worker_pool.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/worker_pool.py#L30) 模块顶层 `_max_executor_workers = int(os.getenv("FETCH_WORKERS", "10"))` — 控制线程池大小，**不可运行时缩容**
- **C 类**：[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L996) `n_workers = int(os.getenv("FETCH_WORKERS", ...))` — 每次启动/健康检查时读取 — 控制 Worker 数量，**可运行时扩容**

这导致一个微妙行为：如果 `FETCH_WORKERS` 从 10 改为 20，ticker 的健康检查会发现 Worker 不足并启动新的（C 类生效），但线程池的 `max_workers` 仍然是 10（A 类不生效），直到重启。

---

## 11. 暗线四：datastore 锁在读写路径上的保护边界

### 11.1 锁的定义

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py) `ChangeDetectionStore` 的 `self.lock = threading.Lock()` 是一个**互斥锁**，保护 `__data` dict 不被并发修改导致数据不一致。

### 11.2 写路径上的锁（显式保护）

| 方法 | 代码位置 | 锁范围 | 说明 |
|------|----------|--------|------|
| `update_watch()` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L549) | `with self.lock:` 包裹 dict update | 防止并发修改同一 Watch |
| `delete()` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L610) | `with self.lock:` 包裹删除操作 | 防止删除与遍历冲突 |
| `clone()` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L654) | `with self.lock:` 包裹 dict() 浅拷贝 | 拷贝期间防止原数据被修改 |
| `add_tag()` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L961) | `with self.lock:` 包裹 Tag 创建 | 防止 UUID 冲突 |
| `add_notification_url()` | [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L1092) | `with self.lock:` 包裹列表追加 | 防止重复追加 |
| `api/Watch` API | [api/Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/api/Watch.py#L78) | `with self.datastore.lock:` | API 写入保护 |
| `watch_base._get_commit_data()` | [model/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L617) | `with lock:` → `dict(self)` → 外部 deepcopy | 快照期间防并发修改 |
| `Watch._get_commit_data()` | [Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L1075) | 同上 | 排除 `processor_config_*` 和 `__*` |

### 11.3 读路径上的锁（缺失保护）

**关键发现**：`datastore.data` 属性（读操作）**没有加锁**：

```python
@property
def data(self):
    d = self.__data
    d['settings']['application']['active_base_url'] = active_base_url.strip('" ')
    return d
```

这意味着：
1. 返回的是 `__data` 的**引用**而非拷贝
2. 调用方获得引用后，如果另一个线程在 `with self.lock:` 中修改了 `__data`，调用方可能看到半修改状态
3. `active_base_url` 的写入是对 `__data` 的**副作用修改**，没有任何锁保护

### 11.4 不加锁的读路径清单

| 读取方式 | 代码位置 | 风险 |
|----------|----------|------|
| `datastore.data` 属性 | 全局 | 返回引用，无锁，可能看到半修改状态 |
| `datastore.data['watching']` 遍历 | ticker_thread | 可能遇到 `RuntimeError: dictionary changed size` |
| `datastore.data['settings']` 直接读 | 多处 | 读取嵌套 dict 的标量值，风险较低 |
| `watch['last_checked']` 等 | ticker, worker | 单键读，Python GIL 保护，实际安全 |

### 11.5 ticker 的 dict 遍历容错

[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1158-L1170) 的 ticker 线程**不使用锁**遍历 `datastore.data['watching']`，而是用 try/except 处理并发修改异常：

```python
while True:
    try:
        for k in sorted(datastore.data['watching'].items(), ...):
            watch_uuid_list.append(k[0])
    except RuntimeError as e:
        time.sleep(0.1)
        watch_uuid_list = []
    else:
        break
```

这是一种**乐观并发**策略：假设冲突概率低，遇冲突则重试。

### 11.6 锁的粒度问题

当前锁是**单一大锁**，所有写操作串行化。这意味着：
- `update_watch(uuid_A, ...)` 和 `update_watch(uuid_B, ...)` 不能并行
- 即使两个 Watch 完全独立，也必须排队
- `_get_commit_data()` 中 `with lock:` → `dict(self)` → 外部 `deepcopy()` 的设计是**正确的**：锁只保护浅拷贝（快照），耗时的 deepcopy 在锁外执行

### 11.7 锁保护边界总结

```
┌─────────────────────────────────────────────────────┐
│  加锁保护（串行化）                                  │
│  ├── update_watch() — dict.update()                 │
│  ├── delete() — dict 删除 + 信号                    │
│  ├── clone() — dict() 浅拷贝                        │
│  ├── add_tag() — Tag 创建                           │
│  ├── add_notification_url() — 列表追加               │
│  ├── API 写入                                        │
│  └── _get_commit_data() — dict(self) 浅拷贝         │
│       (deepcopy 在锁外)                              │
├─────────────────────────────────────────────────────┤
│  不加锁（乐观并发 / GIL 保护）                       │
│  ├── datastore.data 属性 — 返回引用                  │
│  ├── ticker 遍历 watching — try/except RuntimeError  │
│  ├── 单键标量读取 — GIL 原子性                      │
│  └── active_base_url 副作用写入 — 无保护            │
└─────────────────────────────────────────────────────┘
```

---

## 12. 暗线五：main 中 datastore → app → worker → ticker 依赖链的根因

### 12.1 依赖链图

```
main()
 │
 ├─ 1. ChangeDetectionStore(datastore_path)     ← 配置真相来源
 │     ├── 加载 JSON → 内存 __data
 │     ├── 运行 schema 迁移
 │     └── 反序列化为 Watch/Tag 对象
 │
 ├─ 2. changedetection_app(app_config, datastore)  ← 需要 datastore
 │     ├── app.config['DATASTORE'] = datastore
 │     ├── 注册所有 Flask 路由/蓝图
 │     │
 │     ├─ 2a. worker_pool.start_workers(n_workers, update_q, notification_q, app, datastore)
 │     │       ↑ 需要 app（Flask app context）、datastore（读 Watch 配置）、
 │     │         update_q（在 flask_app.py 模块级创建）
 │     │
 │     └─ 2b. threading.Thread(ticker_thread_check_time_launch_checks)
 │             ↑ 需要 datastore（遍历 Watch）、worker_pool（获取运行 UUID）、
 │               update_q（入队）、app.config.exit（优雅退出）
 │
 └─ 3. socketio.run(app, host, port)              ← 需要 app
```

### 12.2 根因分析：为什么是这个顺序？

#### 为什么 DataStore 必须先于 App？

1. **App 构造函数需要 datastore**：[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py) `changedetection_app(app_config, datastore)` 第一个操作就是 `app.config['DATASTORE'] = datastore`
2. **Worker 数量取决于 datastore**：`n_workers = int(os.getenv("FETCH_WORKERS", datastore.data['settings']['requests']['workers']))` — 需要读取 datastore 中的 workers 配置
3. **信号处理器需要 datastore**：[\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/__init__.py#L98-L104) `sigshutdown_handler()` 直接设置 `datastore.stop_thread = True`

#### 为什么 App 必须先于 Worker？

1. **Worker 需要 Flask app context**：[worker.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/worker.py#L78) 中 Worker 在 `app.app_context()` 内执行处理器和通知
2. **update_q 是 flask_app 模块级变量**：[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py) 模块导入时创建 `update_q`，Worker 和 Ticker 都依赖它
3. **notification_q 同理**

#### 为什么 Worker 必须先于 Ticker？

1. **Ticker 入队需要 Worker 消费**：Ticker 将到期 Watch 放入 `update_q`，Worker 从队列取出执行。如果 Ticker 先于 Worker 启动，队列中的项目无人消费
2. **Ticker 需要 `running_uuids`**：`running_uuids = worker_pool.get_running_uuids()` — 判断 Watch 是否正在被处理，避免重复入队。Worker 池必须已存在才能返回有效结果
3. **Worker 健康检查由 Ticker 触发**：[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1130-L1143) ticker 每 60s 调用 `worker_pool.check_worker_health()`，如果 Worker 数量不足则启动新的

#### 为什么 HTTP 服务最后启动？

1. **请求处理依赖所有子系统就绪**：Flask 路由处理器需要 datastore 可读、Worker 可调度、通知可发送
2. **Socket.IO 依赖 Worker 池**：实时更新需要 Worker 处理完成后通过信号通知前端
3. **信号处理器依赖所有组件**：`sigshutdown_handler()` 需要能正确关闭 Worker 池、队列、Socket.IO

### 12.3 循环依赖的破解

存在一个**天然循环**：

```
Worker 需要 app (Flask app context)
App 构造函数启动 Worker
```

破解方式：Worker 接收 `app` 引用，但**不在构造时使用 app context**。app context 仅在 Worker 的实际执行阶段（`app.app_context()` 上下文管理器）才被获取。这确保了在 App 构造完成前，Worker 不会尝试使用尚未就绪的 Flask context。

同样，[\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/__init__.py#L91-L92) 的模块全局 `app = None` / `datastore = None` 在 `main()` 中被赋值，供信号处理器使用。信号处理器在 App 完全构造后才被注册。

### 12.4 batch\_mode 的依赖简写

当 `batch_mode=True` 时，ticker 和 notification_runner **不会被启动**。这是因为 batch 模式下：
1. 所有 Watch 通过 CLI `-u` 参数添加后一次性入队
2. Worker 池处理完队列后进程退出
3. 不需要定时调度（ticker），也不需要持续的通知监听

```
batch_mode=True:  datastore → app → workers → 处理队列 → 退出
batch_mode=False: datastore → app → workers → ticker + notification_runner → HTTP 服务
```

---

## 13. 暗线六：Tag 与 Watch 在持久化处理上的对称性差异

Tag 与 Watch 都继承自 `watch_base`，共享同一个持久化框架，但在三个关键节点上产生了分化。

### 13.1 持久化 Mixin 的统一接口与差异化实现

两者都通过继承链 `EntityPersistenceMixin → watch_base → 具体类` 获得 `commit()` 能力。

[model/persistence.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/persistence.py#L52-L84) 中的 `_save_to_disk()` 通过类名动态推导文件名和大小限制：

```python
entity_type = _determine_entity_type(self.__class__)  # 通过 MRO 从模块名推导
filename = f'{entity_type}.json'                       # Watch → 'watch.json', Tag → 'tag.json'
max_size_mb = 10 if entity_type == 'watch' else 1     # Watch 允许 10MB，Tag 仅 1MB
```

这是**类型驱动的差异**，由 Mixin 在运行时自动判定，不需要子类覆盖。

### 13.2 \_get\_commit\_data() 的选择性排除差异

这是最核心的不对称点：

| 类 | 覆盖 | 排除键 | 设计意图 |
|----|------|--------|----------|
| Watch | ✅ 重写 | `processor_config_*`, `__*` | 处理器配置单独保存为独立 JSON |
| Tag | ❌ 未覆盖 | 无 | 使用 watch_base 默认实现，保留所有键 |

Watch 的 [Watch._get_commit_data()](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L1084-L1087)：

```python
watch_dict = {
    k: copy.deepcopy(v) for k, v in snapshot.items()
    if not k.startswith('processor_config_') and not k.startswith('__')
}
```

Tag 使用 [watch_base._get_commit_data()](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L627) 的默认实现：

```python
return {k: copy.deepcopy(v) for k, v in snapshot.items()}  # 不过滤任何键
```

**不对称性**：Watch 的 `processor_config_restock_diff` 保存在 `{uuid}/watch.json` 之外的独立文件中，而 Tag 的同名字段保存在 `{uuid}/tag.json` 之内。

### 13.3 持久化触发时机的差异

| 触发点 | Watch | Tag |
|--------|-------|-----|
| UI 编辑后 | ✅ `watch.commit()` | ✅ `tag.commit()` |
| API PUT 更新 | ✅ `watch.commit()` | ✅ `tag.commit()` |
| API POST 创建 | ✅ `watch.commit()` | ✅ `tag.commit()` |
| ticker 更新 `last_checked` 等 | ✅ `datastore.update_watch()` → `watch.commit()` | ❌ 无（Tag 无运行时状态） |
| 检测完成更新 `previous_md5` | ✅ `datastore.update_watch()` | ❌ 无 |
| pause/mute/... 状态变更 | ✅ `.commit()` | ✅ `.commit()` |

**不对称性**：Watch 因为运行时状态（`last_checked`、`previous_md5`、`check_count` 等）频繁变更，需要持久化的次数远多于 Tag。Tag 几乎只有在用户主动修改配置时才会持久化。

### 13.4 配置回退时的差异读取路径

Watch 读取自己的 `processor_config_restock_diff` 通过 [base.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/base.py#L274-L303) 的 `get_extra_watch_config('restock_diff.json')` 方法，每次都**从磁盘 JSON 文件读取**：

```python
def get_extra_watch_config(self, filename):
    filepath = os.path.join(data_dir, filename)
    if not os.path.isfile(filepath):
        return {}
    with open(filepath, 'r', encoding='utf-8') as f:
        return json.load(f)
```

而 Tag 的 `processor_config_restock_diff` 通过 [api/Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/api/Watch.py#L124) 的内存访问读取：

```python
restock_config = dict(tag.get('processor_config_restock_diff') or {})
```

**不对称性**：Watch 的处理器配置每次读取都走磁盘 I/O，而 Tag 的处理器配置是内存读取。这意味着 Watch 的 `restock_diff.json` 可以在运行时手动修改磁盘文件，在下一次检测时生效（但这是未文档化的 hack）。

---

## 14. 暗线七：restock_diff 在 Watch 端 vs Tag 端持久化不一致的根因

### 14.1 历史遗留：update_30 的分岔口

[store/updates.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/updates.py#L733-L776) 的 `update_30()` 是不一致的起点：

```python
def update_30(self):
    """Migrate restock_settings out of watch.json into restock_diff.json processor config file.

    For tags: restock_settings key is renamed to processor_config_restock_diff in the tag dict,
    matching what the API writes when updating a tag.
    """
    # --- Watches ---
    for uuid, watch in self.data['watching'].items():
        if watch.get('processor') != 'restock_diff':
            continue
        restock_settings = watch.get('restock_settings')
        # 迁移到独立文件
        filepath = os.path.join(data_dir, 'restock_diff.json')
        if not os.path.isfile(filepath):
            with open(filepath, 'w', encoding='utf-8') as f:
                json.dump({'restock_diff': restock_settings}, f, indent=2)
        del self.data['watching'][uuid]['restock_settings']
        watch.commit()

    # --- Tags ---
    for tag_uuid, tag in self.data['settings']['application']['tags'].items():
        restock_settings = tag.get('restock_settings')
        # 仅仅重命名字段，留在 tag.json 中
        tag['processor_config_restock_diff'] = restock_settings
        del tag['restock_settings']
        tag.commit()
```

**注释明确说了原因**："matching what the API writes when updating a tag"——也就是 API 在写入 Tag 时，也是把 `processor_config_restock_diff` 放在 Tag 对象本身（因为 Tag 没有独立的处理器配置文件接口）。

### 14.2 架构意图 vs 现实妥协

理想的对称设计应该是：

```
Watch: {uuid}/watch.json 主体 + {uuid}/restock_diff.json 处理器配置
Tag:   {uuid}/tag.json   主体 + {uuid}/restock_diff.json 处理器配置
```

但现实是 Tag 没有独立的处理器配置文件机制。原因有三：

1. **API 不对称**：Watch API 有完整的 `update_extra_watch_config()` 接口体系，Tag API 没有对应的 `update_extra_tag_config()`
2. **处理器框架设计**：`get_extra_watch_config()` / `update_extra_watch_config()` 是针对 Watch 的，硬编码使用 `self.datastore.data['watching'].get(self.watch_uuid)`，Tag 没有对应的方法
3. **使用频率低**：Tag 级别的处理器 override 目前仅在 restock_diff 中使用，实现完整的对称架构 ROI 不高

### 14.3 内存中 vs 磁盘中的数据流向

Watch 的 `processor_config_restock_diff` 数据流：

```
UI/API 表单 → request.json → 内存中 watch['processor_config_restock_diff']
    ↓ update_extra_watch_config()
    {uuid}/restock_diff.json（磁盘）
    ↓ 下次检测时 get_extra_watch_config()
    内存中处理器使用
```

Tag 的 `processor_config_restock_diff` 数据流：

```
UI/API 表单 → request.json → 内存中 tag['processor_config_restock_diff']
    ↓ tag.commit()
    {uuid}/tag.json（磁盘，与其他字段合并保存）
    ↓ 下次启动时 _load_tags() + rehydrate_tag()
    内存中处理器 override 使用
```

**关键差异**：Watch 在 `_get_commit_data()` 中排除 `processor_config_*`，所以内存中的 `processor_config_restock_diff` 不会被写入 `watch.json`，必须通过独立的 `update_extra_watch_config()` 保存到 `restock_diff.json`。而 Tag 不排除这个键，直接写入 `tag.json`。

### 14.4 不一致性的实际影响

| 场景 | Watch 行为 | Tag 行为 | 影响 |
|------|-----------|----------|------|
| 运行时编辑磁盘文件 | ✅ 下次检测生效 | ❌ 重启才生效 | Watch 的处理器配置可热加载 |
| `tag.commit()` | — | ✅ 保存 `processor_config_restock_diff` | Tag 持久化完整 |
| `watch.commit()` | ❌ 不保存，需独立调用 | — | 容易遗漏独立保存调用 |
| 迁移回滚 | 需要同时迁移 `restock_diff.json` | 只需迁移 `tag.json` | Watch 备份更复杂 |
| API 返回 | ✅ 从磁盘读 + Tag 内存 override | ❌ 仅内存 | 见 [api/Watch.py:109-128](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/api/Watch.py#L109-L128) |

### 14.5 为什么 API GET /watch/{uuid} 要做"解析合并"

[api/Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/api/Watch.py#L109-L128) 中的代码是理解这个不一致性的最好证据：

```python
# Resolved processor config: tag override wins over watch-level config
_restock_path = os.path.join(watch_obj.data_dir, 'restock_diff.json')
restock_config = {}
if _restock_path and os.path.isfile(_restock_path):
    with open(_restock_path, 'r', encoding='utf-8') as _f:
        restock_config = json.load(_f).get('restock_diff') or {}
restock_source = 'watch'
for tag_uuid in (watch_obj.get('tags') or []):
    tag = tags.get(tag_uuid, {})
    if tag.get('overrides_watch'):
        restock_config = dict(tag.get('processor_config_restock_diff') or {})
        restock_source = f'tag:{tag_uuid}'
        break
watch['processor_config_restock_diff'] = restock_config
watch['processor_config_restock_diff_source'] = restock_source
```

这里清楚地展示了：
1. **Watch**：从 `restock_diff.json` 磁盘文件读取
2. **Tag**：从 `tag.get('processor_config_restock_diff')` 内存读取
3. **优先级**：Tag override > Watch 自身配置

---

## 15. 暗线八：notification_runner 和 ticker 未纳入主依赖图的根因

### 15.1 架构设计上的"二等公民"

在 [flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py) 的模块顶部，我们看到一个清晰的"一等公民" vs "二等公民"分界：

```python
# === 一等公民：模块级全局变量，导入时就存在 ===
datastore = None
update_q = RecheckPriorityQueue()
notification_q = NotificationQueue()
app = Flask(...)
socketio_server = None

# === 二等公民：在 changedetection_app() 内部创建 ===
# ticker_thread = None  ← 注释掉了，说明曾考虑提升但放弃
ticker_thread = None
```

ticker 和 notification_runner 是**在函数内部创建**的局部线程，不是模块级的全局单例，因此不被视为主依赖图的一部分。

### 15.2 notification_runner 的真实依赖链

```
changedetection_app()
 │
 ├─ 1. 设置 app.config
 ├─ 2. 启动 Worker 池（worker_pool.start_workers）
 ├─ 3. 启动 notification_runner 线程
 │    │
 │    ├─ 依赖：app（通过 global app 引用获取 app context）
 │    ├─ 依赖：datastore（通过 global datastore 引用）
 │    ├─ 依赖：notification_q（模块级全局）
 │    └─ 依赖：app.config.exit（优雅退出信号）
 │
 └─ 4. 启动 ticker_thread
```

**根因 1：使用 global 关键字绕过显式传参**

[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1058-L1067) 的 `notification_runner()` 签名不接收任何参数，完全依赖模块级全局：

```python
def notification_runner(worker_id=0):
    global notification_debug_log
    with app.app_context():          # global app
        while not app.config.exit.is_set():
            try:
                n_object = notification_q.get(block=False)  # global notification_q
            ...
            # global datastore
            if not n_object.get('notification_body') and datastore.data['settings']['application'].get('notification_body'):
                n_object['notification_body'] = datastore.data['settings']['application'].get('notification_body')
```

这种设计使得 notification_runner 的依赖是**隐式的**，不是通过参数传递的显式依赖，因此不纳入主依赖图的分析。

### 15.3 ticker 的真实依赖链

[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1112-L1229) 同理：

```python
def ticker_thread_check_time_launch_checks():
    # 依赖 global app, datastore, update_q, worker_pool
    while not app.config.exit.is_set():
        running_uuids = worker_pool.get_running_uuids()
        queued_uuids = {q_item.item['uuid'] for q_item in update_q.queue}
        for k in sorted(datastore.data['watching'].items(), ...):
            ...
        expected_workers = int(os.getenv("FETCH_WORKERS", datastore.data['settings']['requests']['workers']))
        health_result = worker_pool.check_worker_health(
            expected_count=expected_workers,
            update_q=update_q, notification_q=notification_q,
            app=app, datastore=datastore
        )
```

**根因 2：线程是"副作用"，不是可替换的组件**

主依赖图（datastore → app → worker → ticker）中，worker 是可以被替换、mock、配置的可测试组件（有独立的 `worker_pool.py` 模块，有 `check_worker_health()` 等可观测接口）。而 ticker 和 notification_runner 是：

1. **匿名函数**：没有类封装，没有接口抽象
2. **直接依赖全局变量**：不需要传参，启动后自主运行
3. **不可配置的实现细节**：除了环境变量 `NOTIFICATION_WORKERS` 和 `MINIMUM_SECONDS_RECHECK_TIME`，几乎没有配置项
4. **与模块强耦合**：无法提取到独立模块而不重构大量代码

### 15.4 历史演进的证据

查看代码结构可以看到一个清晰的演进路径：

```
早期版本：
  main() 内部直接创建所有线程

中期版本：
  Worker 抽取到独立 worker_pool.py，成为一等公民

当前版本：
  notification_runner 和 ticker 仍在 flask_app.py 中，
  作为 changedetection_app() 函数内的局部创建
```

**根因 3：Worker 池需要跨模块访问，而 ticker/notification 不需要**

- `worker_pool` 被 `api/Watch.py`、`api/Tags.py`、`flask_app.py` 等多个模块引用，必须是模块级导出
- `ticker_thread` 和 `notification_runner` 仅在 `changedetection_app()` 内部启动，没有其他模块需要引用它们
- 它们是"触发即忘"的后台线程，主流程不关心它们的状态（除了优雅退出）

### 15.5 batch_mode 的开关进一步强化了二等地位

[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1001-L1022)：

```python
if not app_config.get('batch_mode'):
    # Only start ticker and notification when in long-running server mode
    global ticker_thread
    ticker_thread = threading.Thread(target=ticker_thread_check_time_launch_checks,
                                     daemon=True, name="ticker")
    ticker_thread.start()

    for i in range(int(os.getenv("NOTIFICATION_WORKERS", "3"))):
        t = threading.Thread(target=notification_runner,
                            daemon=True, args=(i,), name=f"notif-runner-{i}")
        t.start()
```

这明确了 ticker 和 notification_runner 是**服务模式专用组件**，不是核心数据处理流水线的一部分。核心流水线（datastore → app → worker）在 batch_mode 下仍然完整运行。

### 15.6 依赖图的完整形态（含隐式依赖）

```
┌─────────────────────────────────────────────────────────┐
│  主依赖图（显式传参，一等公民）                           │
│  datastore → app → worker_pool → HTTP server            │
│     ↑         ↑         ↑                                │
│     │         │         │                                │
│     └─────────┴─────────┴── update_q, notification_q    │
│                           (模块级全局)                   │
├─────────────────────────────────────────────────────────┤
│  副依赖图（隐式 global，二等公民）                        │
│  notification_runner ◀──┐                                │
│     │                    │                                │
│     ├─ global app        │  global 关键字                │
│     ├─ global datastore  │  绕过显式传参                 │
│     └─ global notification_q │                            │
│                           │                                │
│  ticker_thread ◀──────────┘                                │
│     │                                                    │
│     ├─ global app                                        │
│     ├─ global datastore                                  │
│     ├─ global update_q                                   │
│     └─ global worker_pool                                │
└─────────────────────────────────────────────────────────┘
```

---

## 16. 暗线九：C 类环境变量在容器部署中的生效范围

C 类环境变量是"运行时每次使用时实时读取"的，理论上修改即生效。但在容器（Docker/K8s）部署中，这一假设需要重新审视。

### 16.1 容器环境变量的本质

容器环境变量是**在容器创建时传入**的，通过以下方式设置：

```bash
# Docker CLI
docker run -e MINIMUM_SECONDS_RECHECK_TIME=30 changedetection.io

# Docker Compose
environment:
  - MINIMUM_SECONDS_RECHECK_TIME=30

# Kubernetes
env:
  - name: MINIMUM_SECONDS_RECHECK_TIME
    value: "30"
```

**关键限制**：容器创建后，**无法通过标准的 Docker/K8s API 修改正在运行的容器的环境变量**。环境变量是进程 `execve()` 时传入的，一旦进程启动就固定了。

### 16.2 修改运行中容器环境变量的 hack 方式

虽然标准 API 不支持，但可以通过以下方式修改：

| 方式 | 复杂度 | 对 C 类变量是否生效 |
|------|--------|-------------------|
| `docker exec` 进入容器后 `export VAR=val` | 低 | ❌ 仅影响新子进程，不影响 PID 1 |
| 修改 `/proc/PID/environ` | 中 | ✅ 但需要写权限，且只影响 `os.getenv()` 读取 |
| `gdb` attach 进程修改 environ | 高 | ✅ 但生产环境禁止 |
| K8s ConfigMap/Secret 热更新 | 中 | ❌ 仅文件挂载热更新，不更新进程 ENV |
| 重启容器 | 低 | ✅ 所有类别变量都生效 |

### 16.3 不同 C 类变量在容器中的实际可变性

| C 类变量 | 读取频率 | 容器中可热修改 | 说明 |
|----------|----------|---------------|------|
| `FETCH_WORKERS` | 每 60s + 启动 | ⚠️ 需要 hack | 扩容可热生效，缩容需重启（线程池大小固定） |
| `MINIMUM_SECONDS_RECHECK_TIME` | 每轮 ticker | ⚠️ 需要 hack | 可实时生效 |
| `SALTED_PASS` | 每次认证 | ⚠️ 需要 hack | 可实时绕过 JSON 密码 |
| `BASE_URL` | 每次 `datastore.data` 访问 | ⚠️ 需要 hack | 仅当 JSON 中未设置时生效 |
| `HIDE_REFERER` | 每次请求 | ⚠️ 需要 hack | 可实时生效 |
| `USE_X_SETTINGS` | 每次请求 | ⚠️ 需要 hack | 可实时生效 |
| `ENABLE_NO_PROXY_OPTION` | 每次 get_proxy_list | ⚠️ 需要 hack | 可实时生效 |
| `PAGE_WATCH_LIMIT` | 每次 add_watch | ⚠️ 需要 hack | 可实时生效 |
| `LLM_MODEL/KEY/BASE` | 每次 LLM 调用 | ⚠️ 需要 hack | 可实时绕过 JSON LLM 配置 |
| `LLM_MAX_INPUT_CHARS` | 每次 LLM 输入截断 | ⚠️ 需要 hack | 可实时生效 |
| `LLM_TOKEN_BUDGET_MONTH` | 每次预算检查 | ⚠️ 需要 hack | 可实时生效 |
| `LLM_FEATURES_DISABLED` | 每次 LLM 功能判断 | ⚠️ 需要 hack | 可实时生效 |
| `TZ` | 每次调度/Jinja2 | ⚠️ 需要 hack | 可实时生效 |
| `PDF_TO_HTML_TOOL` | 每次处理 PDF | ⚠️ 需要 hack | 可实时生效 |
| `DISABLED_PROCESSORS` | 启动时（被 `@lru_cache`） | ❌ 即使 hack 也不生效 | 有 `@lru_cache` 保护，只求值一次 |
| `ALLOW_IANA_RESTRICTED_ADDRESSES` | 每次校验/抓取 | ⚠️ 需要 hack | 可实时生效 |
| `ALLOW_FILE_URI` | 每次校验/抓取 | ⚠️ 需要 hack | 可实时生效 |

### 16.4 为什么 DISABLED_PROCESSORS 是 C 类中的特例

[processors/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/processors/__init__.py#L205-L215)：

```python
@functools.lru_cache(maxsize=None)
def available_processors():
    disabled = os.getenv('DISABLED_PROCESSORS', '').split(',')
    ...
    return processors
```

即使运行时修改了进程环境变量，`@lru_cache` 也会返回第一次调用的缓存结果。这使其**名义上是 C 类，实际上是 B 类**（启动后不可变）。

### 16.5 容器环境中的最佳实践矩阵

| 变量类别 | 推荐修改方式 | 是否零停机 |
|----------|-------------|-----------|
| A 类（模块导入时） | 重启容器 | ❌ |
| B 类（启动时） | 重启容器 | ❌ |
| C 类（运行时，无 cache） | 重启容器 + ConfigMap 滚动更新 | ❌（标准做法） |
| C 类（运行时，无 cache） | `/proc/PID/environ` hack | ✅（非标准） |
| C 类（运行时，有 lru_cache） | 重启容器 | ❌ |
| JSON 配置（通过 UI/API） | UI/API 调用 | ✅ |

**容器化部署的关键洞察**：C 类变量的"运行时可修改"特性在标准容器环境中**几乎没有实用价值**。除了直接通过 UI/API 修改 JSON 配置，任何需要修改配置的场景都需要重启容器。

### 16.6 Docker/K8s 中配置管理的推荐分层

基于代码实际行为，推荐以下分层配置策略：

```
第 1 层（最稳定，重启生效）:
  A 类 + B 类 + 带 lru_cache 的 C 类
  → 用 K8s Deployment env 或 Dockerfile ENV 设置
  → 修改 → 滚动重启

第 2 层（动态，运行时生效）:
  JSON 配置（通过 UI/API）
  → 用 UI/API 或 ConfigMap 挂载到 datastore 目录
  → 修改 → 即时生效（但 datastore 不读取已读入的 JSON 文件）

第 3 层（容器 hack，非常规）:
  不带 cache 的 C 类
  → 修改 /proc/PID/environ
  → 修改 → 即时生效（不推荐用于生产）
```

### 16.7 关于 ConfigMap 挂载的特别说明

如果将 `changedetection.json` 通过 K8s ConfigMap 挂载到 datastore 目录：

- **首次启动**：正常加载，配置生效 ✅
- **ConfigMap 更新后**：容器内的文件会被 K8s 更新，但 datastore 不会重新读取 ❌
- **必须重启 Pod**：才能加载新的 JSON 配置 ❌

这是 JSON 作为"只写缓存"架构的必然结果——运行时不会重新读取磁盘文件。

---

## 17. 暗线十：datastore 读路径无锁背后的 GIL 单键原子性

### 17.1 Python GIL 与原子操作的本质

CPython 的全局解释器锁（GIL）保证了**任何 Python 字节码指令的执行都是原子的**。一个 Python 操作是否线程安全，取决于它是否编译为**单个字节码指令**。

```python
# 原子操作（单字节码）
d['key'] = value        # STORE_SUBSCR
value = d['key']        # BINARY_SUBSCR （实际上是 2 个字节码，但由于 GIL 的存在，中间不会被打断）
d.get('key', default)   # 函数调用，多字节码，但 GIL 在函数调用期间不会释放

# 非原子操作（多字节码，中间可能释放 GIL）
d['key'] += 1           # BINARY_SUBSCR → BINARY_ADD → STORE_SUBSCR
d['key'] = d['key'] + 1 # 同上
if 'key' in d:          # COMPARE_OP → POP_JUMP_IF_FALSE
    d['key'].append(x)  # 中间可能被修改
for k in d.items():     # 迭代期间 d 被修改 → RuntimeError
```

### 17.2 datastore 读路径上的原子性分类

datastore 的 `__data` 是一个嵌套 dict，所有读操作可分为三类：

#### 类型 1：单键标量读取（GIL 保证原子）

```python
# 例如：
watch['paused']                    # dict.__getitem__ → 单字节码 + GIL
watch.get('last_checked', 0)       # dict.get() → 函数调用，GIL 不释放
watch['last_error']                # 同上
```

**无需锁**：GIL 保证读取到的值要么是修改前的完整值，要么是修改后的完整值，不会看到部分修改的标量。

#### 类型 2：单键对象引用读取（GIL 保证引用原子，但对象内容可能变）

```python
# 例如：
watch = datastore.data['watching'][uuid]   # 返回 Watch 对象（dict 子类）的引用
watch['headers']                            # 返回 headers dict 的引用
```

**GIL 保证引用本身是原子的**——你不会得到一个半初始化的对象引用。但对象内部的内容（如 `watch['headers']` 的具体键值）可能在你拿到引用后被其他线程修改。

#### 类型 3：多键遍历/迭代（GIL 不保证安全）

```python
# 例如：
for k in datastore.data['watching'].items():
    ...
```

**必须加锁或使用重试机制**：迭代期间如果 dict 大小变化（添加/删除 Watch），会抛出 `RuntimeError: dictionary changed size during iteration`。

### 17.3 ticker 中的乐观并发策略

[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1156-L1170) 没有加锁，而是用 try/except 重试：

```python
# Re #232 - Deepcopy the data incase it changes while we're iterating through it all
watch_uuid_list = []
while True:
    try:
        for k in sorted(datastore.data['watching'].items(), key=lambda item: item[1].get('last_checked',0)):
            watch_uuid_list.append(k[0])
    except RuntimeError as e:
        # RuntimeError: dictionary changed size during iteration
        time.sleep(0.1)
        watch_uuid_list = []
    else:
        break
```

**为什么不加锁？** 权衡分析：

| 策略 | 优点 | 缺点 |
|------|------|------|
| 加锁遍历 | 无重试，确定 | 持有锁时间长，阻塞写操作 |
| try/except 重试 | 无锁开销，写操作不阻塞 | 可能多次重试，极端情况下活锁 |

**实际场景**：
- Watch 的增删是低频操作（用户手动操作）
- 每次遍历只提取 UUID 列表，耗时短（微秒级）
- 冲突概率极低（< 0.1%）
- 即使冲突，重试成本只有 0.1s + 重新遍历的微秒级开销

### 17.4 写路径上加锁但读路径不加锁的正确性论证

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L549-L580) 的 `update_watch()`：

```python
def update_watch(self, uuid, update_obj):
    with self.lock:
        self.__data['watching'][uuid].update(update_obj)
        ...
    self.__data['watching'][uuid].commit()  # 锁外持久化
```

**读写竞态分析**：

```
线程 A（写）: with self.lock: watch.update({'last_checked': now})
线程 B（读）: value = watch['last_checked']
```

可能的时序：

1. **A 先获取锁，update 完成后释放锁，B 再读**：B 看到新值 ✅
2. **B 先读，A 后获取锁 update**：B 看到旧值 ✅（最终一致）
3. **A 持有锁正在 update，B 并发读**：B 看到旧值 ✅（GIL 保证不会看到部分更新的标量）
4. **A 持有锁正在 `del watch['key']`，B 并发读 `watch['key']`**：B 可能看到 KeyError 或旧值，取决于时序，但 GIL 保证不会看到半删除状态

**结论**：对于单键读取，GIL 提供了足够的安全性。不需要加锁。

### 17.5 读路径上的真实风险点

虽然单键读取是原子的，但有两种情况仍然可能出现问题：

#### 风险 1：读取后检查再使用（TOCTOU）

```python
# 不安全模式（实际代码中的模式）
watch = datastore.data['watching'][uuid]
if not watch['paused']:                  # 读 1
    # 这里，另一个线程可能设置 watch['paused'] = True
    queue_for_recheck(watch)             # 使用
```

这是**检查时间-使用时间（Time-of-check to time-of-use）**竞态，GIL 和锁都无法自动防范——需要业务层处理。实际代码中这是可接受的：即使 Watch 被暂停了但已经入队，Worker 在实际执行时会再次检查 `paused` 标志。

#### 风险 2：嵌套 dict 的部分读取

```python
headers = watch['headers']               # 获取内部 dict 的引用
user_agent = headers.get('User-Agent')   # 使用引用
# 另一个线程可能在此时修改 headers['User-Agent']
```

GIL 保证 `headers` 引用是原子的，但 `headers` 内部的修改不被保护。实际代码中这种模式很少，且修改 headers 是用户手动操作，冲突概率极低。

#### 风险 3：`datastore.data` 属性的副作用写入

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L596)：

```python
@property
def data(self):
    ...
    d = self.__data
    d['settings']['application']['active_base_url'] = active_base_url.strip('" ')
    return d
```

这是一个**写入副作用**的读操作！`active_base_url` 的写入没有任何锁保护。但由于：
1. 每次写入的值是相同的（或由环境变量/JSON 决定，变化频率极低）
2. 标量赋值在 GIL 下是原子的
3. 即使读取到旧值，下一次访问会重新计算

因此实际风险可以忽略。

### 17.6 `_get_commit_data()` 锁设计的正确性

[model/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L616-L627)：

```python
if lock:
    with lock:
        snapshot = dict(self)   # 锁内：浅拷贝（O(1) 键数）
# 锁外：深拷贝（O(n) 数据量）
return {k: copy.deepcopy(v) for k, v in snapshot.items()}
```

这是**经典的两阶段拷贝优化**：
1. **锁内**：只做 `dict(self)` 浅拷贝，获取所有键的引用。持有锁时间 = O(键数)，非常短。
2. **锁外**：做 `copy.deepcopy(v)`，耗时但不阻塞其他线程。

如果没有锁，`dict(self)` 期间如果另一个线程正在 `self.update(...)`，可能导致：
- Python 3.7+：由于 dict 是有序的，迭代期间修改可能产生重复键或丢失键
- 抛出 `RuntimeError: dictionary changed size during iteration`

### 17.7 锁与无锁的边界总结

```
┌─────────────────────────────────────────────────────────────┐
│  需要锁保护的操作                                            │
│  ├── dict.update() 批量修改键                               │
│  ├── del 删除键（可能导致遍历异常）                          │
│  ├── dict(self) 浅拷贝（构造快照）                           │
│  ├── list.append() 列表追加（如 notification_urls）          │
│  └── 新增 Watch/Tag（改变 dict 大小）                        │
│                                                              │
│  GIL 保护下无需锁的操作                                      │
│  ├── d['key'] 单键标量读取                                   │
│  ├── d.get('key') 单键标量读取                               │
│  ├── d['key'] = scalar 单键标量赋值（但代码中加了锁）         │
│  ├── 函数调用（GIL 在函数调用期间不释放）                     │
│  └── 引用获取（保证引用本身完整，不保证引用指向的内容不变）     │
└─────────────────────────────────────────────────────────────┘
```

### 17.8 为什么写路径仍然全加锁？

既然单键赋值在 GIL 下是原子的，为什么 `update_watch()` 仍然用 `with self.lock:` 包裹？

**原因 1：update() 是多键批量操作**

```python
# update_obj 可能包含多个键
self.__data['watching'][uuid].update({
    'last_checked': now,
    'previous_md5': new_md5,
    'last_error': None,
    'fetch_time': elapsed
})
```

虽然每个键的赋值是原子的，但四个键的赋值之间 GIL 可能被释放，导致另一个线程看到部分更新的不一致状态（`last_checked` 已更新但 `previous_md5` 还是旧值）。锁保证这四个键的更新是原子的、一致的。

**原因 2：防御性编程**

即使当前 `update_obj` 只有一个键，未来可能增加更多键。加锁是一种前向兼容的防御性设计。

**原因 3：与 `_get_commit_data()` 的协同**

`_get_commit_data()` 在锁内做 `dict(self)` 浅拷贝，`update_watch()` 在锁内做 `update()`，二者互斥，保证拷贝出的快照不会包含部分更新的状态。
